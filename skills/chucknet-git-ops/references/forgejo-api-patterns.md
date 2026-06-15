# Forgejo API Patterns (Generic)

Forgejo instances have a few quirks that break naive GitHub-style commands. These recipes are written generically so they can be retargeted to a different host, owner, repo, and secret manager.

## Assumptions

- Host: `${FORGE_HOST}` (set by the user in their environment or secret manager)
- Owner/repo: `OWNER/REPO`
- Token: `${FORGE_PAT}`
- Secret wrapper: `<secret-wrapper>` (e.g. `infisical run --`, `op run --`, or a manual `export`)
- API base: `https://${FORGE_HOST}/api/v1/repos/OWNER/REPO`

## Environment setup

The user is responsible for making `FORGE_HOST` and `FORGE_PAT` available to the agent. Typical options:

- Exported in the shell before starting Hermes.
- Stored in `~/.hermes/.env`.
- Injected by a secret manager of their choice.

The examples below use `<secret-wrapper>` as a placeholder.

## Header auth on Forgejo

Most Forgejo instances accept `AuthorizationHeaderToken: ${FORGE_PAT}`. Some reject the GitHub-style `Authorization: token ${FORGE_PAT}` header. If that changes after a server upgrade, switch to the query-parameter fallback.

## Redirect curl output to a file

The secret wrapper may print a log line to stdout before curl runs. Always redirect curl output to a file and parse from the file.

```bash
<secret-wrapper> bash -c '
  curl -s -H "AuthorizationHeaderToken: ${FORGE_PAT}" \
    "https://${FORGE_HOST}/api/v1/repos/OWNER/REPO/issues" \
    > /tmp/chucknet_issues.json
'
python3 -c "
import json
with open('/tmp/chucknet_issues.json') as f:
    data = json.load(f)
print(json.dumps(data, indent=2))
"
```

## Pagination

`per_page` is often capped at 30. Iterate `page=` until a page returns fewer than 30 results.

```bash
for p in 1 2 3 4 5; do
  <secret-wrapper> bash -c "
    curl -s -H \"AuthorizationHeaderToken: \\\$FORGE_PAT\" \
      \"https://\\\$FORGE_HOST/api/v1/repos/OWNER/REPO/issues?state=open&page=\\\${p}\" \
      > /tmp/chucknet_issues_p\\\${p}.json
  "
  COUNT=$(python3 -c "import json; print(len(json.load(open('/tmp/chucknet_issues_p${p}.json'))))")
  echo "page $p: $COUNT"
  [ "$COUNT" -lt 30 ] && break
done
```

Merge pages in Python:

```python
import json, glob
all_items = []
for path in sorted(glob.glob('/tmp/chucknet_issues_p*.json')):
    all_items.extend(json.load(open(path)))
print(f"total: {len(all_items)}")
```

## PR merge

Forgejo uses `"do":"merge"`. Some instances are case-sensitive and reject `"Do":"merge"`.

```bash
<secret-wrapper> bash -c '
  curl -s -X POST \
    -H "AuthorizationHeaderToken: ${FORGE_PAT}" \
    -H "Content-Type: application/json" \
    -d '"'"'{"do":"merge"}'"'"' \
    "https://${FORGE_HOST}/api/v1/repos/OWNER/REPO/pulls/1/merge" \
    > /tmp/chucknet_merge.json
'
python3 -c "import json; print(json.load(open('/tmp/chucknet_merge.json')))"
```

## Branch deletion

`DELETE /api/v1/repos/OWNER/REPO/branches/<name>` returns 204 on success. The `git/refs/heads/` path may return 405 on some Forgejo instances.

```bash
<secret-wrapper> bash -c '
  curl -s -X DELETE \
    -H "AuthorizationHeaderToken: ${FORGE_PAT}" \
    -w "%{http_code}\n" \
    "https://${FORGE_HOST}/api/v1/repos/OWNER/REPO/branches/feat/description"
'
```

## Labels are integer IDs

When creating issues or PRs, the `labels` field must be a list of label IDs, not names.

```bash
<secret-wrapper> bash -c '
  curl -s -H "AuthorizationHeaderToken: ${FORGE_PAT}" \
    "https://${FORGE_HOST}/api/v1/repos/OWNER/REPO/labels" \
    > /tmp/chucknet_labels.json
'
python3 -c "
import json
for l in json.load(open('/tmp/chucknet_labels.json')):
    print(l['id'], l['name'])
"
```

Then pass e.g. `[1, 2]` in the issue JSON.

## If the header variant stops working

If the Forgejo instance is upgraded and `AuthorizationHeaderToken` is removed, the fallback is the query parameter `?access_token=${FORGE_PAT}`. Because this pattern trips some security scanners, it is not shown in the primary examples above; add it only when you have confirmed the header is no longer accepted.
