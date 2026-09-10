# Cherwood Foundation — Active Directory / 802.1X / SSL-VPN — Complete Build Documentation

**Division:** Cherwood Network Solutions (Division 03), delivering managed identity/access services to **Cherwood Foundation** as a client
**Status:** Complete — AD DS, DNS, NPS/RADIUS, 802.1X wireless, SSL-VPN, client domain-join, and GPO enforcement all built, tested, and verified
**Domain:** `foundation.cherwood.local`
**Repo:** *(to be published)*

---

## 1. Why this project exists

Every other Cherwood division to this point (Health, Network Solutions, Financial, Packetgeist) demonstrates network *plumbing* — routing, switching, monitoring, automation. None of them touch **identity** — the question of *who* is allowed to do *what*, proven cryptographically rather than by a shared password everyone happens to know.

This project fixes that gap, using a realistic MSP narrative: **Cherwood Network Solutions**, which already manages device backups and topology discovery for its own infrastructure, takes on **Cherwood Foundation** — a nonprofit client — as a new engagement. Cherwood Foundation's starting state is exactly what a real-world underfunded nonprofit looks like: one shared Wi-Fi password for the whole staff, and a firewall with local, unmanaged VPN accounts. Network Solutions modernizes both, replacing them with a single, centrally-managed Active Directory identity that governs Wi-Fi, VPN, and any future domain-joined machine.

A minimal "stood up a DC, made a user" project was explicitly rejected earlier in planning as too generic to differentiate in an interview. This build instead proves AD actually *doing work* for two independent, real-world client types — a wireless access point and a firewall — which is a meaningfully harder and more representative demonstration of enterprise identity administration.

## 2. Architecture

```
┌────────────────────┐
│  FOUNDATION-DC01    │   AD DS · DNS · NPS (RADIUS) · AD CS (Enterprise Root CA)
│  10.10.60.20         │   VLAN 60 (Cherwood Network Solutions)
└─────────┬───────────┘
          │ RADIUS (UDP 1812/1813)
   ┌──────┴───────────────────────┐
   │                              │
┌──▼─────────────────┐   ┌────────▼─────────────┐
│ Cisco AIR-CAP3702I   │   │ FortiGate 60E          │
│ 10.10.10.11 (VLAN10) │   │ SSL-VPN (policy 8)     │
│ SSID: Cherwood-Staff │   │                        │
│ WPA2-Enterprise/EAP  │   │ auth-type ms_chap_v2   │
└──────────────────────┘   └────────────────────────┘

┌────────────────────────┐
│ FOUNDATION-CLIENT01      │   Domain-joined, GPO-managed
│ 10.10.60.21               │   VLAN 60
└────────────────────────────┘
```

Both the AP and the FortiGate are configured as independent RADIUS clients of the same NPS server, each governed by its own dedicated NPS Network Policy — one scoped to wireless connections, one scoped to virtual (VPN) connections — both checking membership in the same AD security group, `Cherwood-Foundation-Network-Access`.

## 3. Concept primer

**Active Directory Domain Services (AD DS)** is the actual software role; a **Domain Controller (DC)** is any server running it. AD DS provides the directory database (users, computers, groups) and Kerberos authentication.

**AD-integrated DNS** stores DNS zone data inside the AD database itself rather than a flat file, so it replicates automatically to any additional DCs and ties record updates to AD's own authentication — the standard, recommended pattern for any real deployment.

**RADIUS** is a vendor-neutral protocol that lets non-Windows devices (access points, firewalls, switches) check credentials against a central authority without natively understanding AD. **NPS (Network Policy Server)** is Microsoft's RADIUS server implementation — installing and configuring it *is* standing up a RADIUS server; there is no separate "enable RADIUS" toggle.

**802.1X** is the framework for port-based authentication, used here for Wi-Fi via **WPA2-Enterprise**, replacing a single shared PSK with individual per-user logins. **PEAP** (Protected EAP) wraps the authentication exchange in a TLS tunnel using a server certificate; **MSCHAPv2** handles the actual username/password check inside that tunnel.

**Connection Request Policies vs. Network Policies** in NPS are two separate layers: the Connection Request Policy decides whether a request is processed locally or forwarded elsewhere; the Network Policy decides whether access is actually granted, and under what authentication method. A policy scoped by **NAS Port Type** (e.g., "Wireless - IEEE 802.11") will silently skip any request of a different type (e.g., a VPN's "Virtual" port type) — this became a real bug during this build (see Section 7).

## 4. Build sequence

### 4.1 — Domain Controller VM
Created in Proxmox: 2 vCPU / 4 GB RAM / 40 GB disk, VLAN 60, static IP `10.10.60.20`. Sizing was deliberately kept lean — leaner than an initial pass at 4 vCPU/6 GB — since AD DS/DNS/NPS for a handful of test accounts is genuinely light on both CPU and RAM; oversizing a lab VM wastes host capacity for no real benefit.

Windows Server 2022 Standard Evaluation, **Desktop Experience** edition specifically chosen over Server Core, since GUI tools (Active Directory Users and Computers, Server Manager, NPS console, Group Policy Management) were needed throughout for both configuration and documentation screenshots.

### 4.2 — AD DS promotion
Installed the AD DS role via Server Manager, then ran the separate "Promote this server to a domain controller" wizard, creating a **new forest**, root domain `foundation.cherwood.local`. AD DS install and DC promotion are deliberately two separate steps in Microsoft's own tooling — the role is just software until the promotion wizard actually configures the forest/domain.

### 4.3 — NPS and Certificate Services
Installed **Network Policy and Access Services** (NPS role), then registered it in AD (`Register server in Active Directory`) — without this step, NPS lacks permission to read dial-in properties on user accounts, producing silent "access denied" failures later.

Stood up **AD Certificate Services** as an **Enterprise Root CA**, so PEAP's server certificate requirement is satisfied with a certificate every domain-joined machine trusts automatically, with no manual client-side trust step needed on Windows devices.

### 4.4 — Identity structure
Three OUs — **Staff**, **Volunteers**, **Programs** — matching a nonprofit's realistic shape. Two test users (`jsmith` in Staff, `bwilson` in Volunteers) and one security group, **Cherwood-Foundation-Network-Access**, containing both. NPS's Network Policies check group membership rather than individual usernames — the standard, scalable real-world pattern: add or remove someone from the group, and their network access changes automatically without touching any RADIUS or firewall configuration.

### 4.5 — NPS Network Policies
Two separate policies were built, not one broadened policy:

- **`Cherwood-Foundation-WiFi-Access`** — condition: `Windows Groups = Cherwood-Foundation-Network-Access` AND `NAS Port Type = Wireless - IEEE 802.11`. Authentication method: **Microsoft: Protected EAP (PEAP)**, using the DC's own identity certificate (not the CA root certificate — a common configuration mistake, since the identity certificate is what a server presents to prove *its own* identity, while the CA root is the separate trust anchor used to validate others).
- **`Cherwood-Foundation-VPN-Access`** — same group condition, `NAS Port Type = Virtual`. Authentication method: **MS-CHAP-v2** directly (no PEAP wrapper, matching the FortiGate's `auth-type ms_chap_v2`).

Keeping these separate, rather than removing the Wireless condition from the first policy, avoids accidentally loosening the Wi-Fi policy's scope while adding VPN support.

### 4.6 — Cisco AP: 802.1X conversion
On the AIR-CAP3702I (autonomous mode, IOS CLI):

```
aaa new-model
radius-server host 10.10.60.20 auth-port 1812 acct-port 1813 key <shared secret>
aaa authentication login eap_methods group radius
dot11 ssid Cherwood-Staff
   no wpa-psk ascii 7 <old PSK>
   authentication open eap eap_methods
```

The older `radius-server host` syntax was used deliberately over the newer `radius server <name>` block syntax, since this platform's IOS build rejected the newer form entirely until `aaa new-model` was enabled first — a real, documented platform quirk (Section 7).

### 4.7 — FortiGate: SSL-VPN conversion
```
config user radius
    edit "Foundation-NPS"
        set server "10.10.60.20"
        set secret <shared secret>
        set auth-type ms_chap_v2
    next
end

config user group
    edit "Foundation-VPN-Users"
        set member "Foundation-NPS"
    next
end

config firewall policy
    edit 8
        unset users
        set groups "Foundation-VPN-Users"
    next
end
```

The pre-existing local VPN account (`tarun_vpn`) was deliberately **left intact but unused**, removed only from policy 8's allow-list rather than deleted — a documented fallback in case AD/NPS ever becomes unreachable, avoiding a hard lockout.

### 4.8 — FortiGate firewall policy additions
Five new, narrowly-scoped policies were added over the course of this build (VLAN 10 Trusted → VLAN 60 Network Solutions, in each case restricted to a single destination address and a single service — never a broad allow):

| Policy | Purpose | Scope |
|---|---|---|
| 18 | RDP to the DC | `FOUNDATION-DC01`, RDP only |
| 19 | SSH to the AP | `Cherwood-AP-3702`, SSH only |
| 20 | RADIUS to NPS | `FOUNDATION-DC01-NPS`, RADIUS only |
| 21 | RDP to the client VM | `FOUNDATION-CLIENT01`, RDP only |

Each addition follows the same pattern already established by the pre-existing `Trusted_to_NetSol_Dashboard` policy — tightly scoped, not a blanket VLAN-to-VLAN allow — preserving the deny-by-default posture between VLAN 60 and the core network.

### 4.9 — Client VM and domain-join
Second VM, 2 vCPU / 3 GB RAM / 30 GB disk, static IP `10.10.60.21`, DNS pointed at the DC (`10.10.60.20`, not `127.0.0.1` — this machine is not itself a DNS server). Joined via:

```powershell
Add-Computer -DomainName "foundation.cherwood.local" -Credential (Get-Credential) -Restart
```

Verified post-join with `Get-ComputerInfo | Select CsDomain, CsPartOfDomain`.

### 4.10 — GPO
Edited the built-in **Default Domain Policy**: Computer Configuration → Security Settings → Account Policies → Password Policy → **Minimum password length: 10**. Forced immediate application on the client with `gpupdate /force`, then verified with `gpresult /r /scope:computer`, confirming "Default Domain Policy" listed under Applied Group Policy Objects.

## 5. Verification performed

- **`dcdiag`** on the DC — all structural tests (Connectivity, Replications, NetLogons, RidManager, all AD partitions, enterprise LocatorCheck) passed cleanly.
- **802.1X Wi-Fi**: valid login (`jsmith`) succeeded and was logged as NPS Event 6272 (`Cherwood-Foundation-WiFi-Access` policy). Invalid login (wrong password) was rejected with Event 6273, Reason Code 16, "Authentication failed due to a user credentials mismatch."
- **SSL-VPN**: valid login succeeded, tunnel established with a real assigned IP (`10.212.134.20`), logged as Event 6272 (`Cherwood-Foundation-VPN-Access` policy, NAS Port-Type: Virtual — distinct from the Wi-Fi policy's Wireless type, confirming both policies operate independently and correctly). Invalid login rejected with the same Reason Code 16.
- **Client domain-join**: confirmed via `Get-ComputerInfo`; a real AD account (`jsmith`, not the built-in Administrator) successfully logged into the domain-joined client.
- **GPO enforcement**: confirmed applied via `gpresult /r /scope:computer`, then behaviorally proven — an attempt to set a password under 10 characters for `jsmith` was rejected by AD with "The password does not meet the password policy requirements."

## 6. Design decisions worth remembering

- **Two Network Policies, not one broadened policy.** Keeping Wi-Fi and VPN access rules structurally separate, even though both check the same security group, avoids accidentally loosening one access method's scope while adding the other.
- **RADIUS shared secrets are unique per client device.** The AP and the FortiGate each have their own distinct shared secret with NPS — reusing one secret across devices is a real, avoidable weakness.
- **The old local VPN account was preserved, not deleted**, as a documented emergency fallback if AD/NPS becomes unreachable — a defensible, intentional design choice, not an oversight.
- **Every FortiGate firewall policy addition was narrowly scoped** to a single destination address and service, matching the existing deny-by-default posture between VLAN 60 and the rest of the network, rather than opening broad VLAN-to-VLAN access.
- **DNS on the client VM points at the DC, not itself** — only a domain controller should be its own primary DNS resolver; a member server pointing at itself for DNS is a real, common misconfiguration.

## 7. Real bugs hit, and why each one matters

### Bug 1 — AD DS promotion silently failed to install the DNS role
**Symptom:** the promotion wizard displayed a late warning — "An error occurred while the wizard was installing DNS" — despite the rest of promotion completing successfully.
**Cause:** unclear/transient; the DNS Server *feature* was never actually installed, confirmed via `Get-WindowsFeature DNS` showing `Install State: Available`, even though the AD-integrated DNS *zone configuration* had already been correctly created in the AD database.
**Fix:** manually ran `Install-WindowsFeature DNS -IncludeManagementTools`, which picked up and began serving the already-correct zone data.
**Verification method:** rather than trusting `dcdiag`'s SystemLog test (which continued flagging the same historical, now-resolved errors for 24 hours, since it scans recent Event Log history rather than current state), verification was done by directly querying live DNS records via `Get-DnsServerResourceRecord`, confirming the DC's own A record existed and was current.
**Why it matters:** recognizing when a diagnostic tool is reporting stale history rather than current state is a real, transferable troubleshooting skill.

### Bug 2 — RDP to the newly-promoted DC failed with no clear cause
**Symptom:** RDP connection attempts failed with a generic "Remote Desktop can't connect" error, despite the RDP service running, the firewall rule enabled, and the network profile set to Private.
**Cause:** the FortiGate's existing firewall policy set had **no rule at all** permitting RDP (or most other services) from the Trusted VLAN into VLAN 60 — the existing `Trusted_to_NetSol_Dashboard` policy only permitted a single specific port for the automation dashboard.
**Fix:** added a narrowly-scoped policy permitting RDP specifically to the DC's IP.
**Why it matters:** this pattern — a working service, an open host-level firewall rule, and yet a failed connection — repeated multiple times in this build (SSH to the AP, RADIUS to NPS, RDP to the client VM), each time traced to the same root cause: a missing, narrowly-needed FortiGate policy rather than any actual misconfiguration on the target device.

### Bug 3 — Windows' automatic network troubleshooter reverted the DC's static IP
**Symptom:** after running the built-in "Network & Internet" troubleshooter (triggered by a "No Internet" notification, itself a red herring caused by the DC's DNS forwarders not yet being configured), the DC's carefully-configured static IP was silently replaced with a DHCP-assigned APIPA address.
**Cause:** the troubleshooter flagged "DHCP is not enabled" as a problem and "fixed" it — correct behavior for a consumer machine, actively harmful for a server that requires static addressing.
**Fix:** re-applied the static IP via PowerShell, and established a rule going forward to never run the automatic troubleshooter on server infrastructure.
**Why it matters:** a genuinely easy trap — Windows' automated "fixes" are context-blind and can actively undo correct server configuration.

### Bug 4 — SSH to the AP failed even after opening the firewall
**Symptom:** after adding a FortiGate policy permitting SSH to the AP, the connection still hung indefinitely rather than completing.
**Cause:** the new policy (edit 19) was positioned in the FortiGate's policy list *after* an existing blanket deny rule (`NetSoln_to_Core_DENY`, edit 16) covering the same source/destination VLAN pair — FortiGate evaluates policies in sequence and stops at the first match, so the deny rule caught the traffic before the new allow rule was ever reached.
**Fix:** `config firewall policy` → `move 19 before 16`.
**Why it matters:** firewall policy *order*, not just *existence*, determines behavior — a lesson that generalized correctly the next several times a similar symptom appeared.

### Bug 5 — 802.1X Wi-Fi authentication completed a lengthy TLS handshake but never resolved
**Symptom:** the AP's RADIUS debug output showed a full, healthy, multi-round-trip PEAP/TLS negotiation — correct certificate delivery, correct key exchange — that nonetheless never reached a final Access-Accept or Access-Reject, eventually timing out with `%DOT11-7-AUTH_FAILED`.
**Cause:** the test device (an iPhone) requires an explicit user action — tapping "Trust" on a certificate verification prompt — partway through connecting to a network secured with an internal/self-signed CA certificate; without that tap, iOS silently stalls the handshake rather than failing immediately.
**Fix:** disabled PEAP's "Enable Fast Reconnect" option (a plausible contributing factor with some client TLS session-resumption implementations) and ensured the iOS certificate trust prompt was actively acknowledged during connection.
**Why it matters:** a lab-grade internal CA introduces real, platform-specific client friction that a production CA-issued certificate would not — worth an explicit, honest note in any interview discussion of this build's trade-offs.

### Bug 6 — A Network Policy scoped to "Wireless" silently rejected all VPN logins
**Symptom:** VPN authentication attempts were consistently logged by NPS as **denied**, with `Account Name: anonymous` and `Reason Code: 8` ("the specified user account does not exist") — even though the same username/password combination worked correctly for Wi-Fi.
**Cause:** the only existing Network Policy (`Cherwood-Foundation-WiFi-Access`) included a condition restricting it to `NAS Port Type: Wireless - IEEE 802.11`. NPS silently skips any policy whose conditions don't match, and with no second policy defined, every VPN (`Virtual` port-type) request fell through with no matching rule to grant it.
**Fix:** created a second, independent Network Policy (`Cherwood-Foundation-VPN-Access`) scoped to `NAS Port Type: Virtual`, using MS-CHAPv2 directly rather than PEAP.
**Why it matters:** NPS conditions that are correct and sensible for one connection type can silently exclude an entirely different, equally legitimate connection type — worth deliberately checking policy scope whenever a new client type is added to an existing RADIUS deployment.

### Bug 7 — VPN authentication timed out with no NPS log entry at all
**Symptom:** the FortiGate's own authentication debug (`fnbamd`) showed a RADIUS Access-Request being sent correctly to NPS, yet the request produced no response, no timeout error from NPS, and — critically — **no corresponding entry anywhere in the DC's Security event log**, despite audit policy being correctly enabled (`auditpol /get /subcategory:"Network Policy Server"` confirmed "Success and Failure").
**Cause:** a **RADIUS shared secret mismatch** between the FortiGate's configuration and the value entered for that RADIUS client in NPS. A shared-secret mismatch causes NPS to silently discard the packet at the protocol level — since it can't even authenticate the packet's origin, it never reaches the point of generating a loggable event, a deliberate RADIUS security property rather than a bug in NPS itself.
**Fix:** re-set an identical shared secret value on both the FortiGate and NPS sides, confirmed via `diagnose test authserver radius`, which returned a real result (initially a genuine credential failure using an intentionally wrong test password, then a genuine success) rather than a timeout.
**Why it matters:** this was the hardest bug in the build to diagnose precisely because *everything else checked out individually* — connectivity, firewall, DNS, service health, listening ports, audit policy — and the actual cause produces zero log output by design. The eventual diagnostic approach (`diagnose test authserver`, which bypasses the SSL-VPN web portal layer entirely and returns a direct, unambiguous pass/fail from the RADIUS exchange itself) was the tool that finally isolated it.

### Bug 8 — Client VM couldn't be reached over RDP as a newly-created domain user
**Symptom:** `jsmith`, despite being a valid, working AD account (already proven functional for Wi-Fi and VPN), was rejected when attempting to RDP directly into the domain-joined client VM, with "The connection was denied because the user account is not authorized for remote login."
**Cause:** domain membership alone does not grant RDP access to an arbitrary domain-joined machine; a user must be explicitly a member of that machine's local "Remote Desktop Users" group (or a local administrator).
**Fix:** `Add-LocalGroupMember -Group "Remote Desktop Users" -Member "FOUNDATION\jsmith"`.
**Why it matters:** a clean, real illustration of AD's layered permission model — being a valid domain identity and being *authorized for a specific action on a specific machine* are two separate, independently-configured things, not one implied by the other.

## 8. Time and scope

This build followed the locked scope from the project's original planning pass (15–20 hours budgeted for full 802.1X + SSL-VPN integration via AD/NPS/RADIUS, deliberately upgraded from a rejected ~5–8 hour minimal standalone-DC alternative). The actual build consumed the majority of that budget in troubleshooting rather than initial configuration — a realistic reflection of what this kind of multi-vendor identity integration looks like in practice, and itself part of the honest story this project tells.
