---
name: commit-description
description: Writes commit descriptions. Use when creating a commit or when the user asks to summarize changes for a commit.
allowed-tools: Bash(git add.), Bash(git status:*), Bash(git commit:*)
user-invocable: true
---

When writing a commit description:

1. Run `git add -A` to stage all modified, added, and deleted files.
2. Run `git diff --cached` to see staged changes.
3. Run `git status` to get a clear picture of added, modified, or deleted files.                                                                                                                                                                                                              
4. If `package.json` was modified, extract the new version from it.
5. Analyze the changes and write a commit message **always in English**, regardless of the language used by the user, in this format:

```
type(scope): short imperative description [vX.Y.Z if package.json version changed]

## What
One sentence explaining what this commit does.

## Changes
- Bullet points of specific changes made
- Group related changes together
- Mention any files deleted or renamed
```

Use conventional commit types: `feat`, `fix`, `refactor`, `chore`, `docs`, `test`, `build`, `ci`. The scope is optional and should reflect the area of the codebase (e.g. `auth`, `api`, `ui`).

**NEVER** add a `Co-Authored-By` trailer or any Claude/AI attribution to the commit message.