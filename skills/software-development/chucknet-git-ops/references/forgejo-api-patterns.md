# Forgejo API Patterns (Generic)

Secret-wrapped Forgejo instances have a few quirks that break naive GitHub-style commands. These recipes are written generically so they can be retargeted to a different host, owner, repo, and secret wrapper.

## Assumptions

- Host: `forge.example.com`
- Owner/repo: `OWNER/REPO`
- Token env var: `TOKEN`
- Secret wrapper: `infisical run --env=prod --silent --` (or any equivalent)
- API base: `https://forge.example.com/api/v1/repos/OWNER/REPO`

## Header auth on this Forgejo instance

This Forgejo instance accepts `AuthorizationHeaderToken: <YOUR_PAT>`. It rejects the GitHub-style `Authorization: token <YOUR_PAT>` header. If that changes after a server upgrade, switch back to the standard header.

## Redirect curl output to a file

The secret wrapper may print a log line to stdout before curl runs. Always redirect curl output to a file and parse from the file.

```bash
infisical run --env=prod --silent -- bash -c '
  curl -s -H "AuthorizationHeaderToken: <YOUR_PAT>" \
    "https://forge.example.com/api/v1/repos/OWNER/REPO/issues" \
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
  infisical run --env=prod --silent -- bash -c "
    curl -s -H \"AuthorizationHeaderToken: \<YOUR_PAT>\" \
      \"https://forge.example.com/api/v1/repos/OWNER/REPO/issues?state=open&page=${p}\" \
      > /tmp/chucknet_issues_p${p}.json
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
infisical run --env=prod --silent -- bash -c '
  curl -s -X POST \
    -H "AuthorizationHeaderToken: <YOUR_PAT>" \
    -H "Content-Type: application/json" \
    -d '"'"'{"do":"merge"}'"'"' \
    "https://forge.example.com/api/v1/repos/OWNER/REPO/pulls/1/merge" \
    > /tmp/chucknet_merge.json
'
python3 -c "import json; print(json.load(open('/tmp/chucknet_merge.json')))"
```

## Branch deletion

`DELETE /api/v1/repos/OWNER/REPO/branches/<name>` returns 204 on success. The `git/refs/heads/` path may return 405 on some Forgejo instances.

```bash
infisical run --env=prod --silent -- bash -c '
  curl -s -X DELETE \
    -H "AuthorizationHeaderToken: <YOUR_PAT>" \
    -w "%{http_code}\n" \
    "https://forge.example.com/api/v1/repos/OWNER/REPO/branches/feat/description"
'
```

## Labels are integer IDs

When creating issues or PRs, the `labels` field must be a list of label IDs, not names.

```bash
infisical run --env=prod --silent -- bash -c '
  curl -s -H "AuthorizationHeaderToken: <YOUR_PAT>" \
    "https://forge.example.com/api/v1/repos/OWNER/REPO/labels" \
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

If the Forgejo instance is upgraded and `AuthorizationHeaderToken` is removed, the fallback is the query parameter `?access_token=<YOUR_PAT>`. Because this pattern trips some security scanners, it is not shown in the primary examples above; add it only when you have confirmed the header is no longer accepted.
