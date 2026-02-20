---
name: git-linear-flow
description: Linear + Git integration workflow. MANDATORY for any implementation task (feature, bugfix, refactor). Defines branch strategy, commit rules, PR process, and Linear issue tracking.
---

# Git Flow — Linear + Git Workflow

> **Every token counts.** Be efficient. Follow the steps. No shortcuts on quality.

---

## Golden Rules

| # | Rule |
|---|------|
| 1 | Default branch is `main`. **NEVER** commit directly to `main`. |
| 2 | Every new implementation must start by creating a **new git worktree** from an **up-to-date** `main`. |
| 3 | If a Linear issue link is provided, **BLOCK** implementation until its content is successfully read via MCP. |
| 4 | Always use the PR template (`.github/PULL_REQUEST_TEMPLATE.md`) when opening a PR. |
| 5 | Before opening a PR: `bun run lint` + `bunx tsc --noEmit` + `bun test` must ALL pass. |
| 6 | Keep the Linear issue updated with comments (progress, decisions, blockers, learnings). |
| 7 | Commits must be atomic — one logical change per commit. |
| 8 | Be token-efficient. No unnecessary reads, no redundant searches. |
| 9 | One implementation = one dedicated worktree + one dedicated branch. |

---

## Branch Naming

Derive the branch name from the Linear issue ID and a short slug:

```
feature/CIL-42-add-user-auth
bugfix/CIL-99-fix-login-redirect
hotfix/CIL-120-patch-xss-vulnerability
refactor/CIL-55-extract-use-case
chore/CIL-70-update-dependencies
```

| Prefix | When |
|--------|------|
| `feature/` | New functionality |
| `bugfix/` | Non-urgent bug fix |
| `hotfix/` | Urgent production fix |
| `refactor/` | Code restructuring (no behavior change) |
| `chore/` | Tooling, deps, config |

> If no Linear issue exists, use a descriptive slug: `feature/add-dark-mode`.

## Worktree Naming

Derive the worktree directory from the branch slug (outside the main repo folder):

```bash
../sigar-CIL-42-add-user-auth
../sigar-CIL-99-fix-login-redirect
```

Use lowercase and hyphenated names; keep it short and unique.

---

## Implementation Workflow (Step by Step)

### Phase 1 — Understand the Task

```
1. Receive task (Linear issue link or description)
2. If Linear link provided:
   → Use Linear MCP tool to READ the issue content
   → If MCP fails to retrieve content → STOP. Do NOT proceed.
   → Extract: title, description, acceptance criteria, labels, priority
3. If no link: work from the user's description
4. Update Linear issue status to "In Progress"
5. Add a comment on the Linear issue: "Starting implementation"
```

> 🔴 **HARD BLOCK:** If a Linear issue link is given but cannot be accessed, DO NOT start any implementation. Inform the user.

### Phase 2 — Prepare Main + Create Worktree (MANDATORY)

```bash
# 1. In the primary repo, switch to main
git checkout main

# 2. Pull latest changes (MANDATORY — never skip)
git pull origin main

# 3. Create a NEW worktree with a NEW branch from main
git worktree add ../sigar-CIL-XX-short-description -b feature/CIL-XX-short-description main

# 4. Enter the new worktree
cd ../sigar-CIL-XX-short-description
```

> Always verify `main` is synced with remote before creating the worktree. If `git pull` fails or has conflicts, resolve FIRST.
>
> Do not implement in the primary working directory when starting a new task.

### Phase 3 — Implement

```
1. Confirm you are inside the dedicated worktree (`git worktree list`)
2. Follow the project's architecture (Clean Architecture + DDD)
2. Make changes in small, focused iterations
3. Commit atomically as you go — don't accumulate a giant diff
4. Each commit message follows Conventional Commits:
```

#### Commit Format

```
<type>(<scope>): <description>

[optional body]

[optional footer: Ref: CIL-XX]
```

| Type | When |
|------|------|
| `feat` | New feature |
| `fix` | Bug fix |
| `docs` | Documentation only |
| `refactor` | No behavior change |
| `test` | Adding/fixing tests |
| `chore` | Tooling, config, deps |
| `perf` | Performance improvement |
| `ci` | CI/CD changes |

#### Example

```
feat(contracts): add lease expiration notification

- Add cron job to check expiring contracts
- Send email 30 days before expiration
- Create notification preference setting

Ref: CIL-42
```

### Phase 4 — Pre-PR Validation

**ALL checks must pass before opening a PR:**

```bash
# 1. Lint
bun run lint

# 2. Type check
bunx tsc --noEmit

# 3. Unit Tests
bun test:unit
```

| Check | Command | Must Pass? |
|-------|---------|------------|
| Lint | `bun run lint` | ✅ YES |
| TypeScript | `bunx tsc --noEmit` | ✅ YES |
| Unit Tests | `bun test:unit` | ✅ YES |

> If ANY check fails → fix it BEFORE opening the PR. Never open a PR with failing checks.

### Phase 5 — Open the Pull Request

```bash
# Push the branch
git push -u origin feature/CIL-XX-short-description
```

**PR Description:** Use the template from `.github/PULL_REQUEST_TEMPLATE.md`:

```markdown
## Motivação
<!-- What problem does this solve? -->
Resolve: [CIL-XX](https://linear.app/cilas/issue/CIL-XX/title)

## O que foi alterado
- File/component A: what changed
- File/component B: what changed

## Tipo de mudança
- [x] Nova feature  <!-- or Bug fix, Refatoração, etc. -->

## Plano de testes
- [ ] Test case 1
- [ ] Test case 2

## Checklist
- [x] `bun run lint` passou sem erros
- [x] `bunx tsc --noEmit` passou sem erros
- [x] Testes novos escritos com `vitest`
- [x] Nenhuma credencial ou secret exposto no código
```

> Use the GitHub MCP to create the PR. Always link the Linear issue.

### Phase 5.1 — Worktree Cleanup (after merge/close)

```bash
# From the primary repo
git worktree list
git worktree remove ../sigar-CIL-XX-short-description

# Optional: remove stale metadata
git worktree prune

# Optional: delete local branch if no longer needed
git branch -d feature/CIL-XX-short-description
```

> Never remove a worktree with uncommitted changes.

### Phase 6 — Update Linear

```
1. Add comment on the Linear issue summarizing:
   - What was implemented
   - Key decisions made
   - Any remaining TODOs or follow-ups
2. Link the PR URL in the comment
3. Update issue status as appropriate
```

---

## Quick Reference — Full Flow

```
┌─────────────────────────────────────────────────┐
│  1. READ Linear issue (MCP) — BLOCK if fails    │
│  2. Update Linear → "In Progress" + comment     │
│  3. git checkout main && git pull origin main   │
│  4. git worktree add ../sigar-CIL-XX-slug ...   │
│  5. cd ../sigar-CIL-XX-slug                     │
│  6. Implement (atomic commits)                  │
│  7. bun install                                 │
│  8. bun run lint ✓                              │
│  9. bunx tsc --noEmit ✓                         │
│ 10. bun test:unit ✓                             │
│ 11. git push -u origin feature/CIL-XX-slug      │
│ 12. Open PR (use template, link Linear issue)   │
│ 13. Comment on Linear (summary + PR link)       │
│ 14. Remove worktree after merge                 │
└─────────────────────────────────────────────────┘
```

---

## Anti-Patterns (NEVER Do)

| ❌ Don't | ✅ Do |
|----------|-------|
| Commit directly to `main` | Always use a feature branch in a dedicated worktree |
| Start coding without reading the Linear issue | Read issue first, THEN code |
| Start implementation in the primary repo directory | Create and use a new worktree first |
| Push without running checks | Run lint + tsc + tests before push |
| One giant commit with all changes | Atomic commits, one logical change each |
| Forget to update Linear | Comment progress, decisions, learnings |
| Create worktree/branch from stale `main` | Always `git pull origin main` first |
| Open PR without using the template | Always fill in the PR template |
| Guess the issue requirements | Use MCP to read the actual issue |
