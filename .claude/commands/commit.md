---
description: Create a git commit following the Conventional Commits specification
argument-hint: [optional hint about type, scope or message]
allowed-tools: Bash(git status:*), Bash(git diff:*), Bash(git log:*), Bash(git add:*), Bash(git commit:*)
---

## Context

- Current status: !`git status --short`
- Staged changes: !`git diff --cached --stat`
- Unstaged changes: !`git diff --stat`
- Recent commits: !`git log --oneline -10`

User hint (may be empty): $ARGUMENTS

## Task

Create a single commit that follows [Conventional Commits 1.0.0](https://www.conventionalcommits.org/):

```
<type>(<optional scope>): <description>

[optional body]

[optional footer(s)]
```

1. If nothing is staged, review the unstaged/untracked changes and stage only the files that belong together. Never stage `node_modules/`, `.DS_Store`, `dist/`, `.angular/` or `packages/api/resttek.db`. If the changes cover unrelated concerns, ask whether to split them into several commits.
2. Choose the `type`:
   - `feat`: new feature
   - `fix`: bug fix
   - `docs`: documentation only
   - `style`: formatting, no code change
   - `refactor`: code change that is neither a fix nor a feature
   - `perf`: performance improvement
   - `test`: add or fix tests
   - `build`: build system or dependencies (npm, angular.json, tsconfig)
   - `ci`: CI configuration
   - `chore`: other maintenance
3. Choose the `scope` from the monorepo package affected: `api`, `web-admin`, `web-empleados`, `web-clientes`, `web-shared`, or `docs` / `scripts` / `repo`. Leave out the scope if several packages are affected equally.
4. Write the description in English, imperative mood, lowercase, no trailing period, max ~72 characters for the whole header.
5. Add a body only when the *why* isn't obvious. Mark breaking changes with `!` after the type/scope and a `BREAKING CHANGE:` footer.
6. Run `git commit` with the message (use a HEREDOC for multi-line messages), then show `git log --oneline -1` to confirm.

If this is not a git repository, stop and tell the user instead of running `git init`.
