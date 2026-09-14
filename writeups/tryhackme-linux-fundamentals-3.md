# Linux Fundamentals Part 3: TryHackMe

**Difficulty:** Easy
**Category:** Linux Fundamentals
**Date completed:** 2026-09-02

## Summary
Final room in the Linux Fundamentals series, covering terminal text editors, general system utilities, process management/automation, and system logging.

## What I Did
- Used `nano` as the terminal text editor for creating/editing files directly from the command line.
- Learned process signals (`SIGTERM`, `SIGKILL`, and `SIGSTOP`) and how they're used with `kill` to control running processes (graceful stop vs. force kill vs. pause).
- Covered general system maintenance concepts: useful utilities, automation, and where system logs live and how they're maintained.

## What I Learned
- `nano` is fast and low-friction for quick edits directly in the terminal. No need to leave the shell to touch a config file or script.
- Not all "stop a process" commands are equal: `SIGTERM` asks a process to shut down cleanly, `SIGKILL` forces it to die immediately with no cleanup, and `SIGSTOP` just pauses it. Knowing the difference matters: force-killing everything by default can leave things in a bad state.
- System logs are a core part of maintaining and troubleshooting a Linux box, tying directly into the "Security Operations" side of what Security+ covers.

## Tools/Commands Used
- `nano`: terminal text editor
- `kill` (with `SIGTERM` / `SIGKILL` / `SIGSTOP`): send control signals to running processes
