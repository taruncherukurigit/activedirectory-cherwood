# Cherwood Foundation — Active Directory / 802.1X / SSL-VPN

Active Directory identity infrastructure, 802.1X wireless authentication, and RADIUS-backed SSL-VPN, built and delivered by **Cherwood Network Solutions** for **Cherwood Foundation**, a nonprofit client.

## What this replaces

Cherwood Foundation's starting state — one shared Wi-Fi password for all staff, and a firewall with local, unmanaged VPN accounts — is exactly what a real, underfunded nonprofit's network typically looks like. This project replaces both with a single, centrally-managed Active Directory identity that governs Wi-Fi, VPN, and any future domain-joined machine.

## What's actually built

- **Active Directory Domain Services** — new forest, `foundation.cherwood.local`, AD-integrated DNS
- **NPS (RADIUS)** — Microsoft's RADIUS server, registered as the authentication backend for two independent client types
- **AD Certificate Services** — Enterprise Root CA, issuing the PEAP server certificate
- **802.1X wireless (WPA2-Enterprise)** on a Cisco AIR-CAP3702I, replacing a shared PSK with per-user AD logins
- **SSL-VPN via RADIUS** on a FortiGate 60E, replacing local VPN accounts with the same AD identity
- **Domain-joined client machine**, with a real AD user (not the built-in Administrator) confirmed logging in
- **Group Policy** — a real, enforced password policy, pushed from the DC and verified both structurally (`gpresult`) and behaviorally (a too-short password rejected by AD)

Every piece above is independently tested for both the valid and invalid case, with server-side log evidence for each — see [`CHERWOOD-FOUNDATION-AD-BUILD-DOC.md`](./CHERWOOD-FOUNDATION-AD-BUILD-DOC.md) for the full build, exact configs, and verification steps.

## Real bugs hit

This build surfaced eight distinct, real issues along the way — a silently-failed DNS role install during DC promotion, three separate missing FortiGate firewall policies, a Windows network troubleshooter reverting a static IP, an NPS policy silently excluding VPN traffic because it was scoped to "Wireless" connections, an iOS certificate-trust prompt stalling a otherwise-healthy PEAP handshake, and a RADIUS shared-secret mismatch that fails completely silently by design. Full symptom/cause/fix/why-it-matters writeups for each are in the build doc.

## Stack

Windows Server 2022 (AD DS, DNS, NPS, AD CS) · Cisco AIR-CAP3702I (autonomous, 802.1X) · FortiGate 60E (SSL-VPN, RADIUS client) · Proxmox VE (VM hosting)

## Related

- [Cherwood Network Solutions](https://networksolutions.tarunc.com) — the MSP delivering this
- [Portfolio home](https://tarunc.com)
