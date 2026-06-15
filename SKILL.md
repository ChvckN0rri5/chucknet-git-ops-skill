---
name: chucknet-git-ops
description: "Use when performing git operations on a private Forgejo or GitHub repo that uses PR-only workflow, PAT-based HTTPS auth, environment-wrapped API calls, and multi-assistant coordination."
version: 1.0.0
author: Hermes Agent
license: MIT
platforms: [linux]
metadata:
  hermes:
    tags: [git, forgejo, github, pr-workflow, pat, environment, api, multi-agent]
    related_skills: [github-pr-workflow]
required_environment_variables:
  - name: FORGE_HOST
    prompt: "Forgejo or GitHub host (e.g. forge.example.com or github.com)"
    help: "Set this to the host where the repo lives."
    required_for: "API and git remote host"
  - name: FORGE_PAT
    prompt: "Personal access token"
    help: "A PAT with repo push/pull access. For GitHub use GITHUB_TOKEN if preferred."
    required_for: "HTTPS push and API auth"
---

# ChuckNet Git Operations

Reusable git and forge-API workflow for a private source-control host. The recipes are written generically so they can be retargeted to a different owner, repo, host, or secret manager.

## Configuration

This skill needs two values supplied by the user:

- **Host** — the forge host, e.g. `forge.example.com` or `github.com`.
- **Token** — a personal access token with push/pull access to the repo.

When this skill loads, Hermes prompts for these via `FORGE_HOST` and `FORGE_PAT`. You can also set them ahead of time in your chosen secret store or shell session.

If you prefer separate values for GitHub, use `GITHUB_HOST` and `GITHUB_TOKEN` and adjust the recipe placeholders accordingly. The examples below use `FORGE_HOST` and `FORGE_PAT`.

In code examples, `<secret-wrapper>` means however you inject the token into a command. That could be a 1Password CLI run, an Infisical run, a manual `export`, or anything else. The skill itself does not depend on any specific secret manager.

## When to Use

- The repo lives on a private Forgejo or GitHub instance.
- SSH push is blocked or unavailable; you must push over HTTPS with a PAT.
- Secrets come from a wrapper such as `infisical run --env=prod --silent --` rather than being exported directly to the shell.
- The project uses PR-only workflow: never push to `main`, always branch and merge via PR.
- Multiple AI assistants or human contributors may be active simultaneously.
- You need reliable recipes for: syncing `main`, creating branches, opening PRs, checking CI, merging via API, and cleaning up branches.

Don't use this skill for public GitHub repos where plain `gh` and `git` already work; prefer `github-pr-workflow` instead.

## Quick Reference

| Item | Typical value / placeholder |
|------|------------------------------|
| Repo host | `<HOST>` e.g. `forge.example.com` or `github.com` |
| Repo path | `OWNER/REPO` (extract from remote URL) |
| API base | `https://<HOST>/api/v1/repos/OWNER/REPO` (Forgejo) or `https://api.<HOST>/repos/OWNER/REPO` (GitHub) |
| Auth header | Forgejo: `AuthorizationHeaderToken: <TOKEN>` (or `?access_token=<TOKEN>` if the header is rejected). GitHub: `Authorization: token <TOKEN>` or `Authorization: Bearer <TOKEN>` |
| Push URL | `https://<TOKEN>@<HOST>/OWNER/REPO.git` |
| Secret wrapper | `<secret-wrapper> bash -c '... <TOKEN> ...'` |
| Default branch | `main` |
| Workflow | branch → commit → push → PR → CI → merge → cleanup |

Substitute `<HOST>` and `<TOKEN>` with the user's configured `FORGE_HOST` and `FORGE_PAT` values (or `GITHUB_HOST` / `GITHUB_TOKEN`).

## Start of Session — Orient Before Acting

When the user asks to "work in the repo today", load this skill first, then orient from live state. Do not start editing until you have confirmed the current branch, latest `main`, open PRs/issues, and whether another assistant is active.

1. Load this skill (`skill_view(name='chucknet-git-ops')`).
2. Confirm the local repo path and that `origin` points to the expected host.
3. Read any local plan docs (e.g. `docs/*_repo-analysis_and_plan.md` if the project keeps them).
4. **Re-sync local `main` with remote before trusting local state.** The checkout can lag `origin/main` by several merged PRs. If the working tree has local modifications, stash them first, sync, then restore:
   ```bash
   git stash push -m "pre-main-sync" -- <paths>
   <secret-wrapper> bash -c '
     git -c http.extraHeader="Authorization: Bearer ${FORGE_PAT}" fetch origin main 2>&1
     git switch main 2>&1
     git -c http.extraHeader="Authorization: Bearer ${FORGE_PAT}" pull --ff-only origin main 2>&1
   '
   git stash pop
   ```
5. **Fetch live tracker info from the forge API** unless the user puts the session in local-only planning mode. Use the file-based curl pattern (see below) so the secret wrapper's stdout log line does not pollute JSON. Fetch:
   - `GET /repos/{owner}/{repo}` for open issue/PR counters and default branch.
   - Paginated issues and PRs (`?state=open` and `?state=closed`, iterate `page=` until <30 results per page).
   - Milestones (`?state=all`) and milestone-attached issues if the project uses milestones.
   - The canonical tracking issue if the project has one.
6. Summarize: current `main` HEAD, merged/open PRs, open/closed issues, active milestones, and the next unblocked item.
7. **Check for other active assistants.** If the user says another AI is working in the repo, stop all remote operations (no fetches, pushes, or API calls) until the user explains lane/issue ownership.
8. Ask which issue the user wants to tackle first, or recommend the top unblocked blocker.

## Secret Wrapper and API Calls

### The stdout-pollution problem

Secret wrappers (e.g. `infisical run --`, `op run --`, or a manual `export`) may still emit a leading log line to stdout (e.g. `INF Injecting N secrets`). Piping that output directly into `jq` or a Python JSON parser can fail. The reliable fix is to **redirect curl output to a file and parse from the file**.

```bash
<secret-wrapper> bash -c 'curl -s -H "AuthorizationHeaderToken: ${FORGE_PAT}" "<API_URL>" > /tmp/chucknet_<endpoint>.json'
python3 -c "
import json
with open('/tmp/chucknet_<endpoint>.json') as f:
    data = json.load(f)
print(json.dumps(data, indent=2))
"
```

For paginated lists, save each page separately (`_p1`, `_p2`, ...) and merge with Python.

### Authenticated git push

SSH is assumed blocked. Push via HTTPS with the token in the URL:

```bash
<secret-wrapper> bash -c 'git -C /path/to/repo push https://${FORGE_PAT}@${FORGE_HOST}/OWNER/REPO.git <branch> 2>&1'
```

### Fetch a branch before push/force-with-lease

```bash
<secret-wrapper> bash -c '
  git -c http.extraHeader="Authorization: Bearer ${FORGE_PAT}" fetch origin <branch> 2>&1 || true
'
```

## Creating a repository

`POST /api/v1/user/repos` requires a token with `write:user` scope. A repo-scoped PAT will fail with:
```json
{"message":"token does not have at least one of required scope(s): [write:user]"}
```

Either create the empty repo via the forge web UI and push with the existing PAT, or regenerate the PAT with `write:user`.

## Branch and Commit Workflow

### Create a branch

```bash
git checkout main && git pull origin main
git checkout -b feat/description
```

Branch prefixes: `feat/`, `fix/`, `refactor/`, `docs/`, `ci/`, `chore/`.

### Commit

```bash
git add <files>
git commit -m "type(scope): short description

Longer explanation, wrapped at 72 chars."
```

## Opening a PR

Use the forge's web UI, `gh`, or the API. For a secret-wrapped Forgejo instance, the API body is easiest written to a file first.

```bash
BRANCH=$(git branch --show-current)
cat > /tmp/chucknet_pr.json <<'EOF'
{
  "title": "feat: add thing",
  "body": "## Summary\nAdds thing.\n\nCloses #1",
  "head": "feat/description",
  "base": "main"
}
EOF
<secret-wrapper> bash -c 'curl -s -X POST \\
  -H "Content-Type: application/json" \\
  -d @/tmp/chucknet_pr.json \\
  -H "AuthorizationHeaderToken: ${FORGE_PAT}"  \
  "https://${FORGE_HOST}/api/v1/repos/OWNER/REPO/pulls" > /tmp/chucknet_pr_response.json'
python3 -c "import json; d=json.load(open('/tmp/chucknet_pr_response.json')); print(d.get('number'), d.get('url'))"
```

## CI Status

### Check statuses for a commit

```bash
SHA=$(git rev-parse HEAD)
<secret-wrapper> bash -c 'curl -s \\
  -H "AuthorizationHeaderToken: ${FORGE_PAT}"  \
  "https://${FORGE_HOST}/api/v1/repos/OWNER/REPO/commits/<SHA>/statuses" > /tmp/chucknet_statuses.json'
python3 -c "
import json
for s in json.load(open('/tmp/chucknet_statuses.json')):
    print(f\"{s['status']} - {s['context']}\")
"
```

## Merging a PR

### Via API

Forgejo uses `"do":"merge"` (lowercase, case-sensitive on some instances). GitHub uses `"merge_method":"squash"`.

```bash
PR_NUMBER=<n>
<secret-wrapper> bash -c 'curl -s -X POST \\
  -H "Content-Type: application/json" \\
  -d \'{"do":"merge"}\' \\
  -H "AuthorizationHeaderToken: ${FORGE_PAT}"  \
  "https://${FORGE_HOST}/api/v1/repos/OWNER/REPO/pulls/<PR_NUMBER>/merge" > /tmp/chucknet_merge.json'
python3 -c "import json; print(json.load(open('/tmp/chucknet_merge.json')))"
```

Verify afterward with `GET /pulls/<N>` to confirm `merged: true` and `state: closed`.

### When merge fails

If the API returns "Please try again later" / "The target couldn't be found", check the PR state first. Likely `mergeable: false` because the base branch has moved. Rebase or update the branch from `main` and retry. Direct push to `main` is usually blocked; do not attempt it.

## Post-Merge Cleanup

1. Verify the PR is merged via API.
2. Reset local `main` and fast-forward:
   ```bash
   git checkout main
   <secret-wrapper> bash -c 'git -c http.extraHeader="Authorization: Bearer ${FORGE_PAT}" pull --ff-only origin main'
   ```
3. Delete the remote feature branch. On some Forgejo instances branch auto-delete does not work despite the setting; use the API:
   ```bash
   <secret-wrapper> bash -c 'curl -s -X DELETE \\
     -H "AuthorizationHeaderToken: ${FORGE_PAT}"  \
  "https://${FORGE_HOST}/api/v1/repos/OWNER/REPO/branches/<BRANCH_NAME>"'
   ```
   Note: `DELETE /api/v1/repos/.../branches/<name>` is the working endpoint on this Forgejo instance; `git/refs/heads/` returns 405.
4. Delete the local branch:
   ```bash
   git branch -d <BRANCH_NAME>
   ```

## Amending and Force-Pushing

After `git commit --amend` on a branch that already exists on the forge, `--force-with-lease` can be rejected as stale. Fetch the remote tip first, then force-push through a temporary PAT remote:

```bash
<secret-wrapper> bash -c '
  set -e
  git remote add tmp-pat-push https://${FORGE_PAT}@${FORGE_HOST}/OWNER/REPO.git 2>/dev/null || true
  git fetch tmp-pat-push <branch>
  git push tmp-pat-push <branch> --force-with-lease
  git remote remove tmp-pat-push
'
```

If still rejected, fetch again — the remote tip may have moved.

## Creating Issues

Labels in the JSON payload must often be **label IDs (integers)**, not names. Query labels first, map names to IDs, create missing labels if needed, then pass e.g. `[1, 2]`.

```bash
<secret-wrapper> bash -c 'curl -s \\
  -H "AuthorizationHeaderToken: ${FORGE_PAT}"  \
  "https://${FORGE_HOST}/api/v1/repos/OWNER/REPO/labels" > /tmp/chucknet_labels.json'
python3 -c "
import json
for l in json.load(open('/tmp/chucknet_labels.json')):
    print(l['id'], l['name'])
"
```

For multi-line issue bodies, write the body to a JSON file and use `--data @/tmp/...`.

## Multi-Assistant Coordination

- If the user says another assistant is active in the repo, **stop all remote operations** until you know which lane/issue the other AI owns.
- Do not push branches or call the API in ways that could race.
- When in doubt, ask the user how to split work before acting.

## Issue/Planning Mode (Local-Only)

Signals: "we are going to be writing issues", "assume it's up to date", "we are not making changes", "draft issues locally first".

Rules:
- Stay local. No API calls, no pushes, no issue open/close unless explicitly asked.
- Assume `main` is current unless asked to sync.
- Draft issue bodies as local markdown files.
- Use placeholder issue numbers for cross-references; rewrite dependencies after creation if needed.
- For multi-issue scoped projects, consider creating a milestone first and attaching issues to it.

## Common Pitfalls

1. **Piping secret-wrapper output directly to a JSON parser.** Redirect curl to a file and parse from the file.
2. **Using the wrong auth method.** Forgejo instances vary: `AuthorizationHeaderToken`, `Authorization: token`, and `Authorization: Bearer` may all return "token is required". The fallback that worked in this session was `?access_token=${FORGE_PAT}`. GitHub uses `Authorization: token` or `Bearer`.
3. **Pushing to `main`.** Always PR-only. Direct push is usually blocked by branch protection.
4. **Assuming local `main` is current.** Sync from remote before every tracker/plan update.
5. **Forgetting to fetch the remote tip before `--force-with-lease`.** Leads to stale-info rejection.
6. **Using string label names in issue/PR JSON payloads.** Map names to integer IDs first.
7. **Writing multi-line JSON bodies inline in `bash -c`.** Write payloads to files and use `-d @/tmp/...`.
8. **Relying on `[skip ci]` to skip post-merge deploys.** Some Forgejo instances fire deploy workflows on every push to `main` regardless.
9. **Expecting branch auto-delete after merge.** Verify and clean up manually via the API if needed.
10. **Not verifying merge success.** `GET /pulls/<N>` should show `merged: true` and `state: closed`.
11. **Ignoring pagination caps.** This Forgejo instance caps `per_page` at 30; iterate `page=` until a page returns fewer than 30 results.
13. **Creating a repo via API requires `write:user` scope.** A repo-scoped PAT can push to existing repos but cannot create new ones. Create the empty repo in the web UI first if your token lacks that scope.
14. **Case-sensitive merge JSON.** On this Forgejo instance, `"do":"merge"` works; `"Do":"merge"` fails.

## Verification Checklist

- [ ] `origin` remote is the expected host/owner/repo.
- [ ] Local `main` is fast-forwarded to `origin/main`.
- [ ] Working branch created from latest `main`.
- [ ] Commits follow conventional-commit style.
- [ ] PR opened with base `main`.
- [ ] CI status checked and green (or user-approved with known exceptions).
- [ ] PR merged via API and verified.
- [ ] Remote and local feature branches deleted.
- [ ] `main` reset to latest after merge.
- [ ] No other assistant is racing the same branch/issue.

## Reference Files

- `references/forgejo-api-patterns.md` — Reusable recipes for authenticated API calls and JSON parsing against Forgejo, including the `?access_token=${FORGE_PAT}` fallback and the `infisical` log-line gotcha.
- `references/multi-assistant-coordination.md` — Rules for avoiding races when more than one assistant is active in the same repo.
