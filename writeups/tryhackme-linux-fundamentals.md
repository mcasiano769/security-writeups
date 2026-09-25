# Linux Fundamentals (Parts 1-3): TryHackMe

**Difficulty:** Easy
**Category:** Linux Fundamentals
**Date completed:** 2026-09-02

## Summary

This three-room series was my first hands-on work in Security+ study, and the whole point was building real terminal muscle memory before any of the more interesting exploitation or defense material could actually make sense. Part 1 covers basic filesystem navigation and searching file contents. Part 2 covers command flags and how to look up what a command can actually do. Part 3 covers a terminal text editor, process control, and system logging. None of it is exciting on its own, but it's the foundation everything after it builds on.

## What I Did

**Part 1: getting around the filesystem.** Started with the core navigation commands, `ls` to see what's in a directory, `cd ..` to move up a level, `mkdir` to create new ones. Then used `grep` to search inside a text file for a specific string, which is how I found a flag buried in the file's content instead of having to open and read the whole thing manually.

**Part 2: flags aren't optional extras.** This room's whole point was that most commands do a lot more than their bare form suggests, and flags are how you unlock that. The clearest example was `ls -a`, which reveals hidden dotfiles that a plain `ls` just doesn't show, even though they're sitting right there in the directory. Also covered `--help` and `man`, both of which pull a command's full usage directly into the terminal instead of needing to look anything up on the web.

**Part 3: editing, processes, and logs.** Used `nano` as a terminal text editor for quick edits without leaving the shell. Covered process control signals, `SIGTERM` asks a process to shut down cleanly, `SIGKILL` forces it to die immediately with no cleanup, `SIGSTOP` just pauses it, and how `kill` uses them to control running processes. Closed out with where Linux system logs live and how they're maintained, which is the first thread tying this series into the Security Operations side of Security+.

## What I Learned

- Filesystem navigation becomes fast once the core commands are muscle memory. Nothing clever about it, it's just repetition until `ls`, `cd`, and `mkdir` stop requiring conscious thought.
- `grep` is the real starting point for searching text instead of manually reading through files, and that habit carries straight into log triage and hunting for strings in command output, well beyond this one room.
- A command's bare form often hides most of what it can actually do. `-a` on `ls` was the clearest proof: without it, hidden config files are invisible even though they're right there. Checking `--help` or `man` before assuming a command can't do something became a real habit out of this.
- Not all "stop a process" commands are equal. Force-killing everything by default (`SIGKILL`) can leave things in a bad state where a clean `SIGTERM` wouldn't have. Small distinction, but it's the kind of thing that matters once you're working on a system you can't just reset.
- System logs are a core part of maintaining and troubleshooting a Linux box, and this was the first point in Security+ study where a fundamentals room started connecting directly to the Security Operations material instead of feeling separate from it.

## Tools/Commands Used
- `ls` / `ls -a`: list directory contents, `-a` includes hidden dotfiles
- `mkdir`: create a directory
- `cd ..`: move up one directory level
- `grep`: search text inside a file for a matching string
- `--help` / `man`: quick usage summary and full manual page for a command
- `nano`: terminal text editor for quick in-shell edits
- `kill` (with `SIGTERM` / `SIGKILL` / `SIGSTOP`): send control signals to running processes
