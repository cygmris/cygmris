# Self-Hosted GitHub Stats — Setup & Redeploy

The GitHub stats cards in [`README.md`](../README.md) are served from a **self-hosted**
[github-readme-stats](https://github.com/anuraghazra/github-readme-stats) instance,
to avoid the rate limits (HTTP 503) of the shared public instance.

## Live instance

- **URL:** https://cygmris-stats.vercel.app
- **Vercel project:** `cygmris-stats` (account `chrisheng86`)
- Consumed by `README.md` for both the stats card (`/api`) and top languages (`/api/top-langs/`).

## Configuration

These live in the **Vercel project → Settings → Environment Variables (Production)**, NOT in git:

| Var | Value | Purpose |
|-----|-------|---------|
| `PAT_1` | a GitHub **classic** token with **no scopes** | raises the GitHub API limit 60 → 5000/hr. No scopes = public data only = safest. |
| `WHITELIST` | `cygmris` | only this username may be queried; anyone hitting the public endpoint with another username gets a "not whitelisted" error card. Closes the open-endpoint abuse vector. |

> ⚠️ **Never commit the token value** (here or anywhere). Create it at
> <https://github.com/settings/tokens> (classic, no scopes). To allow more usernames,
> set `WHITELIST` to a comma-separated list (e.g. `cygmris,otheruser`) and redeploy.

The allowlist is a **built-in** feature of github-readme-stats
(`src/common/access.js` → `guardAccess`), enabled purely by the `WHITELIST` env var — **no code patch**.
The deployed source is unmodified upstream, so there is nothing custom to preserve.

## Rebuild / redeploy from scratch

```bash
# 1. Vercel CLI + login (as chrisheng86)
npm i -g vercel
vercel login

# 2. Clone upstream (no changes needed) and link to the existing project
git clone --depth 1 https://github.com/anuraghazra/github-readme-stats
cd github-readme-stats
vercel link --project cygmris-stats --yes

# 3. Set env vars ONLY if missing (value entered at the prompt, not in shell history)
vercel env add PAT_1 production              # paste a no-scope GitHub token
printf cygmris | vercel env add WHITELIST production

# 4. Deploy to production
vercel --prod --yes
```

## Verify

```bash
# allowed -> real SVG card (HTTP 200)
curl -s -o /dev/null -w '%{http_code}\n' "https://cygmris-stats.vercel.app/api?username=cygmris"

# blocked -> SVG body contains "not whitelisted"
curl -s "https://cygmris-stats.vercel.app/api?username=torvalds" | grep -o "not whitelisted"
```

## Notes

- The instance is deployed via the Vercel **CLI** (not linked to a GitHub repo for auto-deploy),
  so "redeploy" = re-run step 4 from a clone.
- Env vars persist on Vercel across deployments; the current deployment stays live regardless of
  the local clone, which may be a temporary directory.
- If the stats cards ever break: first check whether the public-vs-self-hosted domain in `README.md`
  is `cygmris-stats.vercel.app`, then confirm `PAT_1` / `WHITELIST` still exist (`vercel env ls`).
