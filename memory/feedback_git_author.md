---
name: Git commit rules
description: Always run git status before committing and include --author flag in every commit command
type: feedback
---

Always run `git status` before committing, and always include `--author="Alexei Fedotov <alexei.fedotov@gmail.com>"` in every `git commit` command.

**Why:** User wants commits attributed to them explicitly and to see staged files before every commit.

**How to apply:** Before any commit, run `git status`. In every `git commit` invocation, include `--author="Alexei Fedotov <alexei.fedotov@gmail.com>"`.
