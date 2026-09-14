# Linux Fundamentals Part 2: TryHackMe

**Difficulty:** Easy
**Category:** Linux Fundamentals
**Date completed:** 2026-09-02

## Summary
Follow-up room building on Part 1, focused on command flags/switches and how to look up what a command can actually do.

## What I Did
- Explored how flags modify command output and behavior.
- Used `ls -a` to reveal hidden files (dotfiles) that don't show up with a plain `ls`.
- Used `--help` and the `man` command to look up a command's full set of options directly from the terminal.

## What I Learned
- Flags aren't optional extras: they're often the difference between a command being useless and useful for a given task. `-a` on `ls` was the clearest example: without it, hidden config files and dotfiles are invisible even though they're right there in the directory.
- Never need to guess or search the web for what a command can do: `--help` gives a quick summary, `man` gives the full manual, both right in the terminal.

## Tools/Commands Used
- `ls -a`: list all directory contents, including hidden (dotfile) entries
- `--help`: quick usage summary for a command
- `man`: full manual page for a command
