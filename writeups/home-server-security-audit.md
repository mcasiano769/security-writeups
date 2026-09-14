# Security Audit Report: Home Server (extreme-serv)

**Prepared by:** Miguel
**Date of Audit:** August 30, 2026 (with a follow-up control-effectiveness finding on September 12, 2026)
**System Audited:** Home server `extreme-serv`, Ubuntu 26.04 LTS, running Plex, Jellyfin, Portainer, Postfix (outbound mail relay), Samba (4 shares), and WireGuard firewall rules (interface not currently active)
**Framework Used:** NIST Cybersecurity Framework (CSF)

---

## 1. Objective

I conducted a full read-only security audit of my home server before granting it any ongoing remote administrative access, to establish a real baseline rather than assume the existing configuration was safe. This server runs more than a simple file share: it's a genuine multi-service stack, so the audit covered listening ports, firewall rules, Docker container network exposure, SSH configuration, authentication logs, file-share configuration, VPN status, and patch level.

The audit identified one high-severity finding, which I remediated the same day, along with two lower-priority open items that I deliberately deferred with a documented rationale rather than fixing blind. I also investigated a set of suspicious authentication log entries and confirmed them as a false positive rather than assuming either way. A separate follow-up two weeks later uncovered that an automated patch-management control I believed was working had actually been silently failing since it was built, which I also remediated and verified.

Net result: one high-severity exposure closed same day, one control-effectiveness gap caught and closed on follow-up, two low-priority items formally accepted and documented with reasoning, one investigated alert cleared as a false positive, and two existing controls (patch management, file-share configuration) confirmed already effective.

---

## 2. Method

**In scope:**
- Full inventory of network-listening services and open ports
- Firewall (ufw/iptables) rule review
- Docker container network exposure (port publishing)
- SSH configuration and authentication logs
- Samba file-share configuration
- WireGuard/VPN status
- OS patch level and automated update tooling

**Out of scope (at time of original audit):**
- Physical security of the server
- Full penetration testing of any hosted application (Plex, Jellyfin)

**Methodology:** Direct authenticated read-only access to the host via SSH. Enumerated all listening network ports and cross-referenced each against ufw's configured rules, since a published Docker container port can bypass host firewall rules entirely regardless of what the firewall itself is configured to allow. Reviewed sshd configuration directly rather than assuming defaults. Reviewed authentication logs for anomalous activity and independently verified the source of any suspicious entries rather than treating a login attempt as automatically hostile. Reviewed Samba share configuration for guest access and encryption requirements. Checked patch level and confirmed whether automated update tooling was actually applying patches, not just installed and assumed functional.

---

## 3. Findings

### Finding 1: Container Management Interface Exposed to LAN and Potentially the Internet

- **Severity:** High
- **NIST CSF Mapping:** Protect (PR.AC – Access Control, PR.PT – Protective Technology)
- **Description:** Portainer, a Docker management interface with root-equivalent control over every container on the host, was publishing its ports bound to all network interfaces (`0.0.0.0`) rather than a restricted interface. Docker's own firewall rule for published container ports has a source of `0.0.0.0/0`, which bypasses the host firewall's default-deny policy entirely, regardless of the firewall's own configured rules.
- **Risk:** Root-equivalent management access to every container on the host, reachable from the LAN and potentially further, is one of the highest-impact single points of exposure a server like this can have.
- **Remediation:** Recreated the Portainer container with its ports bound explicitly to the Tailscale interface IP only, matching the same pattern already correctly used for another service (Jellyfin) on the same host. Underlying data was preserved in a separate Docker volume, unaffected by the container recreation.
- **Verification:** Confirmed the service remains reachable over the Tailscale IP and is refused when accessed from the plain LAN IP. Remote access for daily use goes over Tailscale already, so this closed the exposure with no loss of legitimate functionality.
- **Status:** Closed, same day as identified.

### Finding 2: SSH Password Authentication Enabled Server-Wide

- **Severity:** Medium
- **NIST CSF Mapping:** Protect (PR.AC – Access Control)
- **Description:** SSH password authentication was enabled server-wide, even though the intent was key-only access. One account on the server has a real password and no SSH key configured, meaning it is theoretically reachable via password guessing over SSH.
- **Risk:** A password-guessable SSH login path on an internet-adjacent service is a real, if partially mitigated, exposure.
- **Remediation decision:** Formally deferred rather than fixed immediately. Disabling password authentication server-wide would also lock out my own primary account, which still logs in by password and does not yet have an SSH key configured. The correct fix is to set up my own key first, then disable password authentication globally, in that order, to avoid locking myself out of my own server.
- **Compensating control already in place:** `fail2ban` is active on the host and mitigates brute-force password guessing in the meantime.
- **Status:** Open, formally risk-accepted with a documented remediation plan and prerequisite, not an oversight.

### Finding 3: Unused VPN Port Open to the Internet

- **Severity:** Low
- **NIST CSF Mapping:** Protect (PR.AC – Access Control) / Identify (ID.RA – Risk Assessment)
- **Description:** A firewall rule allows WireGuard traffic (UDP 51820) from anywhere on the internet, including a blanket allow-all rule for traffic once inside the tunnel. No WireGuard interface is currently active, so nothing is actually listening on this port today.
- **Risk:** Dead attack surface for a service not currently in use. Low likelihood of exploitation with nothing listening, but an unnecessary open rule that should either be closed or formally documented as intentional if WireGuard is deployed later.
- **Status:** Open, low priority, queued for cleanup.

### Finding 4: Unnecessary SSH Feature Enabled

- **Severity:** Low
- **NIST CSF Mapping:** Protect (PR.PT – Protective Technology, secure configuration baseline)
- **Description:** `X11Forwarding` is enabled in the SSH daemon configuration. This server is headless and has no need for forwarded graphical sessions.
- **Risk:** Minor unnecessary feature surface on a service that is otherwise a primary administrative access point.
- **Status:** Open, low priority, to be cleaned up the next time SSH configuration is touched for the password-authentication fix in Finding 2.

### Finding 5 (Informational): Investigated Authentication Log Anomaly, Confirmed False Positive

- **Severity:** Informational, no gap identified
- **NIST CSF Mapping:** Detect (DE.AE – Anomalies and Events, DE.CM – Security Continuous Monitoring)
- **Description:** Authentication logs showed 28 SSH login attempts against an administrative account from a single LAN IP address within a five-minute window. Rather than assume this was either malicious or benign, I independently verified the source.
- **Result:** The source IP and MAC address matched my own working laptop exactly, consistent with a stray leftover connection attempt from a setup session the prior night. The account in question has no password set and is key-only, so no compromise was possible regardless.
- **Conclusion:** Confirmed false positive through direct verification, not assumption.
- **Status:** Cleared, no action required.

### Finding 6 (Informational): Patch Level and Update Automation Verified

- **Severity:** Informational, control effective
- **NIST CSF Mapping:** Protect (PR.MA – Maintenance)
- **Description:** Reviewed current patch level and confirmed automated update tooling for the security-relevant package channel.
- **Result:** Patch level was current at time of audit, with only two trivial non-security packages pending. Unattended upgrades were enabled and actively applying security patches.
- **Status:** No action required at time of original audit. See Finding 8 below for a related control-effectiveness issue found on follow-up.

### Finding 7 (Informational): File-Share Configuration Verified

- **Severity:** Informational, control effective
- **NIST CSF Mapping:** Protect (PR.AC – Access Control)
- **Description:** Reviewed Samba file-share configuration across all four configured shares.
- **Result:** No guest access permitted on any share, valid users scoped per share individually, and a minimum of SMB3 with encryption required on the primary shares.
- **Status:** No action required.

### Finding 8: Patch Management Automation Silently Non-Functional (Follow-Up Finding, September 12, 2026)

- **Severity:** Medium
- **NIST CSF Mapping:** Protect (PR.MA – Maintenance); relevant to control-effectiveness validation generally
- **Description:** Two custom scripts intended to apply nightly and weekly OS updates automatically had failed on every single execution since the day they were created, roughly two weeks prior, due to a missing privilege-escalation command in both scripts. Neither script had ever successfully applied a single patch or completed an unattended reboot despite running on schedule the entire time.
- **Risk:** A control I believed was actively patching the system had never actually functioned. This is exactly the kind of gap a compliance review is meant to catch: a control assumed effective without ever being verified end to end.
- **Mitigating factor:** The operating system's own separate, built-in unattended-upgrades service had been functioning correctly and independently the entire time, and had kept all security-relevant packages patched regardless of the custom scripts' failure. Confirmed no actual security exposure resulted.
- **Remediation:** Corrected both scripts to properly invoke elevated privileges. Verified by manually executing the nightly script, which then successfully applied 13 previously stuck package updates end to end.
- **Status:** Closed, same day as identified, with real end-to-end verification rather than assuming the fix worked.

---

## 4. Remediation & Recommendations

**Remediation summary:** The high-severity Portainer exposure and the patch-management control-effectiveness gap were both closed with same-day fixes and real end-to-end verification, not just assumed working. Two low-priority items were formally deferred, not ignored, each with a documented rationale and a compensating control already in place where relevant.

**Recommendations going forward:**
- Set up my own SSH key on this server, then disable password authentication server-wide, closing Finding 2 for good. This is the correct order, doing it in reverse would lock myself out.
- Close or explicitly document the open WireGuard port (Finding 3) and disable `X11Forwarding` (Finding 4) the next time SSH configuration is touched, most naturally alongside the Finding 2 fix.
- Periodically re-verify that automation assumed to be working actually is, the way Finding 8 was caught. A control running on schedule is not the same as a control succeeding, and this audit only caught the gap because the underlying result (patch level) was checked directly rather than the script's exit status being assumed.

---

## 5. Environment Reference

- **Host role:** Home server running media services (Plex, Jellyfin), container management (Portainer/Docker), outbound mail relay (Postfix), and file sharing (Samba), on Ubuntu 26.04 LTS.
- **Administrative access model:** A dedicated, separate administrative account authenticated via SSH key only (no password set), with full passwordless sudo, explicitly approved after the tradeoff (unattended root-equivalent access with no human checkpoint at time of use) was clearly stated up front.
- **Remote access model:** All legitimate remote access to server-hosted services routes through Tailscale rather than direct router port-forwarding, which materially reduces the real-world impact of any LAN-scoped finding above.
