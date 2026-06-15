# Multi-Assistant Repo Coordination

When more than one AI assistant may be working in the same repository at the same time, follow these rules to avoid races and lost work.

## Default Rule

If the user says another assistant is active in the repo, **stop all remote operations** (fetch, push, API calls) until the user explains:

- Which assistant owns which lane or issue.
- Whether the current assistant should wait, work on a different issue, or proceed.

## Lane Awareness

Many projects split work into lanes. A typical split is:

- **Architecture / infra / styling / routing** — assistant-implementation lane.
- **Content / data / legal wording / media** — human owner or a different assistant.

Do not take content/data work unless the user explicitly assigns it to the current assistant. This prevents cross-lane collisions.

## Race Prevention

- Do not push branches that another assistant may also be updating.
- Do not merge a PR another assistant is still editing.
- Do not call the issue/PR API to update status on an item owned by another assistant without explicit direction.
- When drafting issues locally in planning mode, use placeholder issue numbers for cross-references. Rewrite real numbers only after the user approves creation.

## Local-Only Planning Mode

Signals that the session should stay local:

- "we are going to be writing issues"
- "assume it's up to date"
- "we are not making changes"
- "draft issues locally first"

Rules:

- No API calls, no pushes, no branch creation on the forge.
- Draft issue bodies as local markdown files (e.g. under `/tmp/`).
- Use placeholder issue numbers.
- Batch-create via the API only when the user explicitly says "go for it".

## Syncing `main` Without Collisions

Before syncing local `main`, check the working tree:

```bash
git status --short
```

If there are local changes, stash them, sync, then restore:

```bash
git stash push -m "pre-main-sync" -- <paths>
<secret-wrapper> bash -c '
  git -c http.extraHeader="Authorization: Bearer *** fetch origin main
  git switch main
  git -c http.extraHeader="Authorization: Bearer *** pull --ff-only origin main
'
git stash pop
```

Replace `${FORGE_HOST}` and `${FORGE_PAT}` with whatever environment variables the user has configured.
