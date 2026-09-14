# Security Audit Report: Personal MacBook Pro (T2 Hardware, Ubuntu Host)

**Prepared by:** Miguel
**Date of Audit:** August 30, 2026
**System Audited:** Personal MacBook Pro, converted from macOS to Ubuntu via the t2linux community kernel (genuine Apple T2-chip hardware, not clone hardware)
**Framework Used:** NIST Cybersecurity Framework (CSF)

---

## 1. Objective

I conducted a full security audit of my personal MacBook Pro after setting up remote administrative access to it. The goal was to establish a real security baseline before trusting the machine with any ongoing remote access, rather than assuming it was safe by default.

The audit identified three exposure issues on the host, all of which I remediated the same day, and confirmed one systemic gap (no active firewall or intrusion-prevention tooling at all) which I also closed immediately. I additionally verified, by checking the router's port-forwarding configuration directly, that none of these exposures were ever reachable from the public internet. All risk was contained to the local network. One existing control (a locally hosted AI service) was checked and confirmed to already be correctly scoped, requiring no changes.

Net result: three real findings closed, one systemic control gap closed, zero internet-facing risk at any point, full remediation completed within the same audit session.

---

## 2. Method

**In scope:**
- Full inventory of network-listening services on the host
- Host-based firewall and intrusion-prevention posture
- Container-based service network exposure (Docker)
- Router-level port-forwarding configuration, to determine internet-facing exposure
- SSH access configuration for the newly created administrative account

**Out of scope:**
- Router firmware/configuration changes (flagged as high-risk to modify without a dedicated session, given past instability when changing router settings)
- Physical security of the device
- Application-layer review of installed personal software beyond network exposure

**Methodology:** Direct authenticated access to the host via SSH. Enumerated all listening network ports and cross-referenced each against whether the associated service was actually in use. Reviewed `ufw` (firewall) and `fail2ban` (intrusion prevention) status. Reviewed Docker container port bindings for any container published to all network interfaces (`0.0.0.0`) rather than restricted to localhost. Independently verified real-world exposure by checking the router's port-forwarding table directly, rather than relying on host-level findings alone.

---

## 3. Findings

### Finding 1: Remote Desktop Service Exposed to Local Network

- **Severity:** Medium
- **NIST CSF Mapping:** Protect (PR.PT – Protective Technology, PR.AC – Access Control) / Identify (ID.AM – Asset Management)
- **Description:** A system-level remote desktop service (`gnome-remote-desktop`) was running and listening on port 3389 across all network interfaces, reachable by any device on the local network. The service was actually failing to start correctly due to a missing library, but a lower-level socket-activation mechanism kept the port open and listening regardless of that failure.
- **Risk:** An open, listening administrative port increases attack surface for any device on the LAN, regardless of whether the service is fully functional. This is also an asset-visibility gap: the service's presence and state were not previously known or tracked before the audit surfaced it.
- **Remediation:** Disabled the service outright, since it is not used (`systemctl disable --now gnome-remote-desktop`).
- **Verification:** Confirmed port 3389 no longer listening on any interface.
- **Status:** Closed.

### Finding 2: Self-Hosted AI Chat Service Exposed to Local Network

- **Severity:** Medium
- **NIST CSF Mapping:** Protect (PR.AC – Access Control)
- **Description:** A self-hosted AI chat interface, running in a Docker container, was published to all network interfaces (`0.0.0.0`) rather than restricted to local access only. This meant the service and its stored data were reachable by any device on the local network, despite the fact that I only ever use this service locally on the host itself.
- **Risk:** Network-wide exposure of a service that has no legitimate need to be reachable off-host increases the chance of unauthorized access to stored conversation data.
- **Remediation:** Recreated the container with its published port bound explicitly to localhost (`127.0.0.1`) only, preserving all existing data and configuration.
- **Verification:** Confirmed the service remains reachable from the host itself and is refused when accessed from another device on the network.
- **Status:** Closed.

### Finding 3: No Host-Based Firewall or Intrusion Prevention Active

- **Severity:** High
- **NIST CSF Mapping:** Protect (PR.AC – Access Control) / Detect (DE.CM – Security Continuous Monitoring)
- **Description:** The host had no active firewall (`ufw` was installed but inactive) and no intrusion-prevention tooling (`fail2ban` was not installed at all). This meant there was no automated boundary protection and no automated detection/response to repeated unauthorized access attempts against any service on the host, including SSH.
- **Risk:** This is a systemic, host-wide gap rather than a single-service issue. It is the underlying condition that made Findings 1 and 2 more dangerous than they needed to be, and it left every other service on the host without a baseline layer of defense.
- **Remediation:** Installed and enabled `fail2ban` for automated intrusion prevention. Enabled `ufw`, configured to allow SSH only from the local network subnet, with a default-deny policy on all other inbound traffic.
- **Verification:** Confirmed SSH access remained functional after the firewall was enabled, and confirmed the new default-deny policy was active.
- **Status:** Closed.

### Finding 4 (Informational): Internet-Facing Exposure Check

- **Severity:** Informational (no gap identified)
- **NIST CSF Mapping:** Identify (ID.RA – Risk Assessment)
- **Description:** To determine whether any of the above findings represented real-world, internet-facing risk, I directly reviewed the router's port-forwarding configuration rather than assuming based on host-level findings alone.
- **Result:** Only a single port-forwarding rule existed on the router, unrelated to this host, and forwarding to a different device entirely. Nothing was forwarded to this MacBook Pro at any point.
- **Conclusion:** All risk identified in Findings 1–3 was confined to the local network. This host had zero direct exposure to the public internet before or during the audit.
- **Status:** Confirmed, no action required.

### Finding 5 (Informational): Existing Control Verified Effective

- **Severity:** Informational (control already effective)
- **NIST CSF Mapping:** Protect (PR.AC – Access Control)
- **Description:** A locally hosted large-language-model backend service, running alongside the AI chat interface in Finding 2, was reviewed for the same network exposure issue.
- **Result:** This service was already correctly bound to localhost and an internal container-networking address only, and was never exposed to the local network or beyond.
- **Conclusion:** No remediation needed. Documented here to demonstrate the audit reviewed all related services, not only the ones with findings.
- **Status:** No action required.

---

## 4. Remediation & Recommendations

**Remediation summary:** All three exposure findings and the systemic firewall/intrusion-prevention gap were closed within the same audit session. The unused, crash-looping remote desktop service was disabled outright. The self-hosted AI chat interface was rebound to localhost-only. `ufw` and `fail2ban` were both installed and enabled, with SSH restricted to the local subnet and a default-deny policy on everything else.

**Recommendations going forward:**
- Re-run this same listening-port and firewall-rule review periodically, since this audit caught services (the remote desktop service in particular) that were running without my own awareness. Asset visibility, not just fixing what's found, is the real long-term control.
- No outstanding items remain open from this audit. Prior to this audit, the host had two individually exposed services and no baseline automated defense of any kind; following remediation, the host has no unnecessary listening services, all container-based services correctly scoped to their actual usage pattern, and both a firewall and intrusion-prevention system actively running.

---

## 5. Environment Reference

- **Host role:** Personal daily-use laptop, converted from original macOS to Ubuntu Linux using the t2linux community kernel project (genuine Apple T2-chip hardware).
- **Administrative access model:** A dedicated, separate administrative account was created for remote management, authenticated via SSH key only (no password-based SSH login), with elevated (sudo) privileges deliberately kept password-gated rather than passwordless, given that this host holds personal data.
