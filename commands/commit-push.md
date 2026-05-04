---
allowed-tools: Bash(git add:*), Bash(git status:*), Bash(git commit:*), Bash(git push:*), Bash(git diff:*), Bash(git log:*), Bash(git branch:*), Bash(git rev-parse:*)
description: Create a git commit then push
---

## Context

- Current git status: !`git status`
- Staged changes: !`git diff --cached`
- Unstaged changes: !`git diff`
- Current branch: !`git branch --show-current`
- Recent commits: !`git log --oneline -10 2>/dev/null || echo "(no commits yet)"`

## Your task

Based on the above changes, create a single git commit and push to origin.

You have the capability to call multiple tools in a single response. Stage and create the commit using a single message. Do not use any other tools or do anything else. Do not send any other text or messages besides these tool calls unless any command fails. If any command fails, always report the failed command, exit code, and stderr before continuing. Do not add Claude Code attribution in the message.
