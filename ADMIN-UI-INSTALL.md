# Admin UI install guide

Add a self-service "Providers" admin page to any Epic EMS portal so staff can edit `providers.json` in this repo from the portal's own UI — no git, no CLI. The page commits changes back to this repo via the GitHub API, and all portals pick them up within ~5 min.

This is the same setup that ships in [epic-ems-portal](https://github.com/ericmoore705/epic-ems-portal) — use that repo as the reference implementation.

---

## 1. Create a GitHub PAT for commits

Do this once per Epic EMS account (or per portal if you want per-portal audit trails).

1. Go to https://github.com/settings/personal-access-tokens/new
2. **Token name**: `Epic EMS Portal Config Editor`
3. **Expiration**: 1 year (calendar-remind yourself; renew before it expires)
4. **Resource owner**: your GitHub account
5. **Repository access**: **Only select repositories** → pick `epic-ems-config`
6. **Permissions** → Repository permissions:
   - **Contents**: **Read and write**
   - (leave everything else at "No access")
7. **Generate token** → copy the `github_pat_...` value once (you won't see it again).

## 2. Add env vars to the portal (Vercel or equivalent)

| Var | Value |
|---|---|
| `GH_CONFIG_TOKEN` | The PAT from step 1 |
| `ADMIN_EMAILS` | Comma-separated allowlist. Default: `emoore@epicems.com`. Only these accounts see + use the Admin tab. |
| `CONFIG_REPO` *(optional)* | `owner/repo` if not the default `ericmoore705/epic-ems-config`. |
| `CONFIG_PATH` *(optional)* | Path within the repo if not `providers.json`. |

For Vercel:

```bash
vercel env add GH_CONFIG_TOKEN production --value <token> --yes
vercel env add GH_CONFIG_TOKEN development --value <token> --yes
vercel env add ADMIN_EMAILS production --value emoore@epicems.com --yes
```

Then redeploy for the values to take effect.

## 3. Copy the endpoint

Copy [`api/admin-providers.js`](https://github.com/ericmoore705/epic-ems-portal/blob/main/api/admin-providers.js) into your portal's `api/` folder as-is.

**One thing to adapt**: the endpoint requires `checkAuth(req)` from your portal's own auth helper. In `epic-ems-portal` that's [`api/_helpers.js`](https://github.com/ericmoore705/epic-ems-portal/blob/main/api/_helpers.js) — Google OAuth restricted to `@epicems.com`. If your other portal uses a different auth mechanism, adjust the two `require`/`checkAuth` lines at the top so they return `{ email, name }` for authenticated users and `null` for anonymous.

The `cleanEnv` helper is also imported from `_notify.js`. If your portal doesn't have that file, inline this two-liner instead:

```js
function cleanEnv(v) {
  return (v || "").replace(/^﻿/, "").replace(/[​-‍]/g, "").trim();
}
```

## 4. Add the frontend Admin tab

The reference implementation in `epic-ems-portal` adds:

- A tab button hidden by default (`display:none`) — surfaced only for admin users after a probe call on login.
- A `tab-content` div with an editable table (`Provider`, `AngelTrack URL`, `Default State`) plus Save / Add / Reload buttons.
- JS functions: `probeAdmin()`, `loadAdminProviders()`, `renderAdminTable()`, `buildAdminRow()`, `addProviderRow()`, `saveAdminProviders()`.

Grab the tab markup, styles, and JS from [`public/index.html`](https://github.com/ericmoore705/epic-ems-portal/blob/main/public/index.html) — search for `tab-admin`, `admin-tbody`, `loadAdminProviders`, `probeAdmin` and copy those blocks. Hook `probeAdmin()` into your existing "after login" flow.

If your portal is React/Vue/etc, port the same shape:

```
GET /api/admin-providers            → { isAdmin, sha, providers, repo, path }
POST /api/admin-providers  { providers, sha }  → { success, commit_sha, commit_url, new_file_sha }
```

## 5. Test

1. Sign in with an email in `ADMIN_EMAILS`.
2. Open the Admin tab — you should see the current provider list loaded.
3. Change an AngelTrack URL, hit **Save Changes**.
4. Response shows a `commit_sha` link. Click it to see the commit in the config repo.
5. Wait ~5 min → other portals' `/api/providers` endpoints will start returning the new value on their next cold-start.

## Concurrency and safety

- The `sha` you POST back must match the one you fetched via GET. GitHub returns 409 if someone else committed in between; the endpoint surfaces that as a 409 error and the frontend should reload.
- The `Remove` button removes a provider row from the UI but only takes effect after **Save Changes**. Reload always discards unsaved edits.
- Provider **`value`** is the key. Renaming it is a breaking change — historical Stripe metadata still references the old name, and email lookups (`angeltrackUrlFor`) won't match until charges age out. Prefer adding a new provider + retiring the old one over renaming.

## Not covered here (build if you need it)

- Per-provider defaultState validation against a real US state list.
- Audit log beyond commit history.
- Bulk import from CSV.
