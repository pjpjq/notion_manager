# Hot-loading a Notion account via `/admin/accounts/add`

[← Back to README](../README.md)

> Available since `fix(admin): HandleAddAccount 接受 expected_email 防止 token_v2 误绑用户` (PR #3 on `pjpjq/notion_manager`).

## What this endpoint does

`POST /admin/accounts/add` accepts a `token_v2` cookie value from any logged-in Notion session and:

1. Calls `DiscoverAccountFromTokenWithOptions` against Notion's `loadUserContent`
2. Validates the optional `expected_email` selector (filters users at the discovery layer)
3. Persists the resolved account to `accounts/<account_id>__<email>.json` (mode `0600`)
4. Hot-loads the account into the running pool (no Space restart, no process recycle)

This is the recommended path for adding a new member to a workspace that is **already shared** with multiple users — the same workspace can host one pool entry per Notion account.

## Auth

The endpoint requires a **dashboard session cookie**, not the API key.

```bash
# 1) Login (client-side SHA256(salt + password) is performed by the dashboard frontend)
#    After successful login, the response sets a `dashboard_session` cookie.

# 2) Reuse that cookie in subsequent admin calls
curl -X POST https://pjpjq-mirofish.hf.space/admin/accounts/add \
  -H 'Cookie: dashboard_session=<value>' \
  -H 'Content-Type: application/json' \
  -d '{"token_v2":"v03:...","expected_email":"someone@outlook.com"}'
```

## Request body

| Field           | Type   | Required | Notes                                                                                                                                  |
| --------------- | ------ | -------- | -------------------------------------------------------------------------------------------------------------------------------------- |
| `token_v2`      | string | yes      | Notion `token_v2` cookie value, must be URL-encoded as it would be in a real `Cookie:` header                                       |
| `notion_user_id`| string | no       | Pin to a specific `notion_user` id. Useful when one `token_v2` is shared across a multi-account browser session                      |
| `expected_email`| string | **yes**  | Email the discovered user must match. Mismatches are rejected at the discovery layer with `no user matched the configured Notion account selectors` |

> **Always pass `expected_email` when importing an account from a shared workspace.** Without it, the discovery layer returns the *first* user in the `loadUserContent` response — which can be the wrong one in a multi-account browser session. This is the "use token_v2 alone and you get duplicates" failure mode.

## Response

```json
{
  "status": "ok",
  "filename": "<account_id>__<email>.json",
  "account": {
    "name": "<notion display name>",
    "email": "<verified email>",
    "space": "<workspace name>",
    "plan_type": "<team|personal|...>"
  }
}
```

On mismatch (HTTP 400):

```json
{
  "error": "Failed to discover account: no user matched the configured Notion account selectors"
}
```

## End-to-end example (zantoartieg4@outlook.com)

```bash
# 1) Extract token_v2 from the target account's Notion tab
#    DevTools → Application → Cookies → www.notion.so → token_v2
TOKEN='v03%3AeyJhbGciOiJkaXIiLCJraWQiOiJwcm9kdWN0aW9uOnRva2VuLXYzOjIwMjQtMTEtMDci...'
EMAIL='zantoartieg4@outlook.com'

# 2) POST to /admin/accounts/add with dashboard session cookie
curl -sS -X POST https://pjpjq-mirofish.hf.space/admin/accounts/add \
  -H 'Cookie: dashboard_session='"$DASHBOARD_SESSION" \
  -H 'Content-Type: application/json' \
  -d "{\"token_v2\":\"$TOKEN\",\"expected_email\":\"$EMAIL\"}" | jq .
# → { "status": "ok", "filename": "e1050e67...__zantoartieg4@outlook.com.json", ... }

# 3) Verify via /v1/models (the model list grows by one account's worth)
curl -sS https://pjpjq-mirofish.hf.space/v1/models \
  -H 'Authorization: Bearer '"$API_KEY" | jq '.data | length'
# → 27  (each account contributes its full model list)
```

## Dry-run before you push

You can verify the `token_v2` against Notion's API directly (no need to hit `notion-manager`):

```bash
USER_ID='3b5d872b-594c-81d7-8cd3-000231c06587'   # from x-notion-active-user-header
COOKIE="notion_browser_id=<...>; device_id=<...>; notion_user_id=${USER_ID}; notion_users=%5B%22${USER_ID}%22%5D; token_v2=${TOKEN}"

curl -sS -X POST https://www.notion.so/api/v3/loadUserContent \
  -H "Content-Type: application/json" \
  -H "User-Agent: Mozilla/5.0 (Macintosh; Intel Mac OS X 10_15_7) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/137.0.0.0 Safari/537.36" \
  -H "x-notion-active-user-header: ${USER_ID}" \
  -H "Cookie: ${COOKIE}" \
  -d '{}' | jq '.recordMap.notion_user'
# → { "<USER_ID>": { "value": { "value": { "name": "...", "email": "zantoartieg4@outlook.com" } } } }
```

If the email in the response is not exactly the email you intend to add, do **not** proceed — investigate first.

## Deduplication

`HandleAddAccount` calls `EnsureAccountID()` on the discovered account before saving, then `pool.AddAccount` de-duplicates by `AccountID`. Re-posting the same `token_v2 + expected_email` is a no-op on disk (the existing file is updated in place) and is also a no-op in the pool (the same `AccountID` is ignored).

If two `token_v2` values both resolve to the same `AccountID` (e.g. the same user logging in twice with different device cookies), the pool keeps the first one and the second is dropped at load time. No duplicates leak through.

## Failure modes

| Symptom                                                                 | Cause                                                                          | Fix                                                                 |
| ----------------------------------------------------------------------- | ------------------------------------------------------------------------------ | ------------------------------------------------------------------- |
| `no user matched the configured Notion account selectors`               | `expected_email` does not match any user the token can see                    | Re-run the dry-run above, fix the email, retry                      |
| `loadUserContent API error 401`                                         | `token_v2` is expired or wrong workspace                                      | Have the user re-login to Notion, copy a fresh `token_v2`           |
| `loadUserContent API error 403`                                         | Notion rejected the synthetic `x-notion-active-user-header`                    | Drop `notion_user_id` from the body, let discovery pick the only one |
| HTTP `401 unauthorized, dashboard login required`                       | Missing or expired `dashboard_session` cookie                                  | Re-login at `/dashboard/`, copy the new cookie                      |
| Pool already shows the account but quota lookup fails                  | File persisted but the Space is mid-rebuild                                    | Wait for `RUNNING` and re-check `/admin/accounts`                    |

## See also

- `docs/configuration.md` — env-var based startup secrets (`NOTION_TOKEN_V2_<N>` etc.) as the alternative path that requires a Space restart.
- `docs/registration.md` — bulk register via the `notion-manager-register` CLI (Microsoft SSO credentials required).
- `internal/proxy/account_api.go` — `HandleAddAccount` source.
- `internal/proxy/account_api_discovery_test.go` — `TestHandleAddAccountRejectsMismatchedExpectedEmail`, `TestHandleAddAccountAcceptsMatchingExpectedEmail`.
