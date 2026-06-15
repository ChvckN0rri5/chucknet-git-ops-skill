# Forgejo API Patterns (Generic)

Secret-wrapped Forgejo instances have a few quirks that break naive GitHub-style commands. These recipes are written generically so they can be retargeted to a different host, owner, repo, and secret wrapper.

## Assumptions

- Host: `forge.example.com`
- Owner/repo: `OWNER/REPO`
- Token env var: `TOKEN`
- Secret wrapper: `infisical run --env=prod --silent --` (or any equivalent)
- API base: `https://forge.example.com/api/v1/repos/OWNER/REPO`

## Auth methods — header vs query parameter

Most Forgejo docs describe token headers, but this instance's behavior is:

- `AuthorizationHeaderToken: ${TOKEN}` returned `"message":"token is required"`.
- `?access_token=${TOKEN}` succeeded for read and write operations.
- Standard `Authorization: token ${TOKEN}` / `Authorization: Bearer ${TOKEN}` also returned token-required errors.

Therefore, **prefer `?access_token=${TOKEN}` when the header variants fail**. Keep the redirect-to-file pattern either way, because the secret wrapper still emits a leading log line.

```bash
infisical run --env=prod --silent -- bash -c '
  curl -s "https://forge.example.com/api/v1/repos/OWNER/REPO/issues?access_token=${TOKEN}" \
    > /tmp/chucknet_issues.json
'
```

## Creating repositories requires `write:user` scope

`POST /api/v1/user/repos` failed with:
```json
{"message":"token does not have at least one of required scope(s): [write:user]"}
```

A repo-scoped PAT cannot create new repos. Either:
- Use the forge web UI to create the empty repo, then push content with the repo-scoped PAT, or
- Issue/regenerate a token that includes `write:user` scope.

## Pagination

`per_page` is often capped at 30. Iterate `page=` until a page returns fewer than 30 results.

```bash
for p in 1 2 3 4 5; do
  infisical run --env=prod --silent -- bash -c "
    curl -s \
      \"https://forge.example.com/api/v1/repos/OWNER/REPO/issues?state=open&page=${p}&access_token=\${TOKEN}\" \
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
    -H "Content-Type: application/json" \
    -d '"'"'{"do":"merge"}'"'"' \
    "https://forge.example.com/api/v1/repos/OWNER/REPO/pulls/1/merge?access_token=${TOKEN}" \
    > /tmp/chucknet_merge.json
'
python3 -c "import json; print(json.load(open('/tmp/chucknet_merge.json')))"
```

## Branch deletion

`DELETE /api/v1/repos/OWNER/REPO/branches/<name>` returns 204 on success. The `git/refs/heads/` path may return 405 on some Forgejo instances.

```bash
infisical run --env=prod --silent -- bash -c '
  curl -s -X DELETE \
    -w "%{http_code}\n" \
    "https://forge.example.com/api/v1/repos/OWNER/REPO/branches/feat/description?access_token=${TOKEN}"
'
```

## Labels are integer IDs

When creating issues or PRs, the `labels` field must be a list of label IDs, not names.

```bash
infisical run --env=prod --silent -- bash -c '
  curl -s "https://forge.example.com/api/v1/repos/OWNER/REPO/labels?access_token=${TOKEN}" \
    > /tmp/chucknet_labels.json
'
python3 -c "
import json
for l in json.load(open('/tmp/chucknet_labels.json')):
    print(l['id'], l['name'])
"
```

Then pass e.g. `[1, 2]` in the issue JSON.
