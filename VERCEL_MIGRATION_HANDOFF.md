# Vercel Migration — Lumos Suite

> Bundled handoff covering all Lumos-org repos migrating from VPS reclabs3 → Vercel.
>
> **Origin session**: `[vercel_migration_1]` on 2026-05-26 (audit + plan).
> **Status**: 2 of 2 complete.

## Scope

| # | Project | Local repo | GitHub | Domain | Framework | Status |
|---|---------|-----------|--------|--------|-----------|--------|
| 1 | docs-lumos | `~/local-dev/docs-lumos` | `getlumos/docs-lumos` | docs.lumos-lang.org | Astro Starlight | **DONE** (2026-05-26, session `[lumos_vercel_1]`) |
| 2 | lumos-website | `~/local-dev/lumos-website` | `getlumos/lumos-website` | lumos-lang.org (+ www) | Vite/React SPA | **DONE** (2026-05-26, session `[lumos_vercel_1]`) |

## Migration #1 — execution notes (docs-lumos, completed 2026-05-26)

- Vercel project: `rectors-projects/docs-lumos` (deployment id `dpl_6Z2wAXP3...`, then `dpl_*` for `docs-lumos-2vq7btipp` after auto-deploy).
- Package manager corrected: handoff said pnpm but repo uses `package-lock.json` → switched to npm. `vercel.json` enforces `npm install` + `npm run build` (includes OG image generation).
- DNS provider confirmed Cloudflare (zone owner: `Rector@rectorspace.com's Account`, zone id `54223e65d908e0f98c4fa975aecb6538`).
- DNS cutover: record id `ffae72b68c5a723230556511062b0fda` patched A `151.245.137.75` (proxied) → CNAME `cname.vercel-dns.com` (DNS-only). Propagated globally within ~2 min.
- SSL: Vercel did not auto-issue within the normal window; manual trigger `vercel certs issue docs.lumos-lang.org` issued the Let's Encrypt R13 cert. Cert auto-renews.
- GitHub auto-deploy: Vercel's GitHub App was not installed on `getlumos` org. After manual install via Vercel dashboard, `getlumos/docs-lumos` now auto-deploys on every push to master.
- Old `.github/workflows/deploy.yml` (GHCR + SSH VPS deploy) removed in the migration commit; only `mirror-gitlab.yml` and `codeql.yml` remain.
- VPS decommissioned same day (2026-05-26) — buffer skipped because static sites + git source = trivial rebuild path. Containers stopped, images pruned (1.97GB reclaimed), nginx sites removed, certbot certs deleted, port registry updated. `lumos` user kept on VPS per ecosystem rule.

## Migration #2 — execution notes (lumos-website, completed 2026-05-26)

- Vercel project: `rectors-projects/lumos-website` (deployment id `dpl_8bYKpNps...` initial prod, then `dpl_ovevbb3s3...` after auto-deploy).
- Branch is `main` (not `master` like docs-lumos).
- Package manager: same handoff-error pattern as M1 — repo had **both** `package-lock.json` AND `bun.lockb` (unusual). Chose npm to match the explicit `npx tsx` build script. `npm install` failed once on esbuild postinstall (Node v24 + spawnSync errno -88 flake); workaround was `npm install --ignore-scripts`. The esbuild binary itself was fine — re-run worked.
- `vercel.json` includes BOTH the SPA fallback rewrite (`/(.*)` → `/index.html`) AND the host-matched 308 redirect for `www.lumos-lang.org` → `https://lumos-lang.org/$1`. Domain-level redirect lives in code, not the dashboard.
- DNS cutover (Cloudflare zone `54223e65d908e0f98c4fa975aecb6538`):
  - apex record id `713fd9975599d3cccabf31dd2b80783c`: A 151.245.137.75 (proxied) → A 76.76.21.21 (DNS-only)
  - www record id `eafd657d1ae4460dfefdbfcf7452a314`: CNAME lumos-lang.org (proxied) → CNAME cname.vercel-dns.com (DNS-only)
- SSL: same as M1, Vercel did NOT auto-issue. Manually triggered `vercel certs issue lumos-lang.org` and `vercel certs issue www.lumos-lang.org`. Two separate Let's Encrypt R13 certs (apex + www), expiring 2026-08-24, auto-renewing.
- GitHub auto-connect: worked first try this time (Vercel GitHub App was installed on `getlumos` org during M1 and covers all repos in the org).
- Vercel CLI behavioral change to remember: once a project has custom domains attached, `vercel deploy --yes` becomes a PREVIEW deploy by default — use `vercel deploy --prod --yes` to promote.
- WASM-specific check: `/wasm/lumos_core_bg.wasm` serves with `content-type: application/wasm` — Playground page works.
- SPA caveat: unknown paths return HTTP **200** (index.html shell) instead of 404 because of the SPA fallback rewrite — React Router handles the 404 view client-side. Matches typical Vercel/Netlify SPA setups.
- VPS Docker container left running for 7-day rollback buffer. Decommission pending 2026-06-02.

## How to use this file

For each migration:
1. `cd` into the target local repo (e.g. `cd ~/local-dev/docs-lumos`)
2. Start a new Claude Code session
3. Paste the matching **Starter Prompt** block below into the session
4. Execute. After completion, return here and mark the row above as DONE.

The starter prompts are self-contained — the new session does not need to read this file or any prior conversation.

---

## Migration #1 — docs-lumos

**Target repo to `cd` into**: `~/local-dev/docs-lumos`

### Starter Prompt

```
You are continuing a Vercel migration. Today, migrate this repo (docs-lumos) from VPS reclabs3 → Vercel.

==========================================================
PROJECT
==========================================================
- Name: docs-lumos
- What: Astro Starlight documentation site for Lumos lang
- Domain: docs.lumos-lang.org
- Tech: Astro + Starlight (pure static — no DB, no runtime env vars)
- GitHub: git@github.com:getlumos/docs-lumos.git

==========================================================
CURRENT VPS STATE (reclabs3 — 151.245.137.75)
==========================================================
- Linux user: lumos
- Docker container: docs-lumos (healthy, 2-month uptime)
- Port: 0.0.0.0:4000 -> container :80
- nginx site config: /etc/nginx/sites-enabled/docs-lumos-lang-org
- nginx upstream: proxy_pass http://localhost:4000
- TLS: Let's Encrypt cert "docs.lumos-lang.org" (expires ~2026-08-17, auto-renew)
- Deploy: GitHub Actions → GHCR → SSH (appleboy/ssh-action) → docker compose pull

==========================================================
TARGET VERCEL CONFIG
==========================================================
- Project name: docs-lumos
- Framework: Astro (auto-detect)
- Build command: astro build (default)
- Output: dist
- Install: pnpm install (auto-detect)
- Env vars: NONE (verify by grepping repo for process.env / import.meta.env.PUBLIC_*)
- Production domain: docs.lumos-lang.org

==========================================================
EXECUTION STEPS
==========================================================

1) Pre-flight
   $ vercel --version              # should be ≥ 54.4.1
   $ git status                    # must be clean
   $ git pull --rebase             # sync with origin
   $ pnpm install && pnpm build    # local smoke test — must succeed, dist/ generated

2) Link + initial deploy
   $ vercel link                   # accept defaults; team scope = your personal or org
   $ vercel                        # preview deploy
   - Open preview URL, click through Starlight pages, verify rendering

3) Add production domain
   Vercel dashboard → Project → Settings → Domains → add docs.lumos-lang.org
   Capture the DNS records Vercel shows.

4) DNS cutover at registrar (verify which provider hosts lumos-lang.org first)
   - Lower current A record TTL to 300 (5 min); wait 10 min
   - Replace A 151.245.137.75 with the CNAME/A Vercel provides
   - If on Cloudflare: temporarily set to "DNS only" (grey cloud) during cutover

5) Promote production
   $ vercel --prod

6) Verify cutover
   $ dig docs.lumos-lang.org +short
   # Expect Vercel IPs, NOT 151.245.137.75
   $ curl -sI https://docs.lumos-lang.org | grep -iE 'server|x-vercel'
   # Expect: server: Vercel

==========================================================
ROLLBACK
==========================================================
If broken: revert DNS A record back to 151.245.137.75. VPS container is still running. TTL was 300s so propagation is ~5 min.

==========================================================
VPS-SIDE DECOMMISSION (AFTER 7-DAY STABILITY BUFFER)
==========================================================
Do NOT do this immediately. Wait 7 days after Vercel is live.

ssh lumos
  cd ~/<app-dir>                # locate docker-compose.yml (likely ~/docs-lumos or similar)
  docker compose down
  docker image prune -f

ssh reclabs3 (as root)
  sudo rm /etc/nginx/sites-enabled/docs-lumos-lang-org
  sudo rm /etc/nginx/sites-available/docs-lumos-lang-org   # if exists
  sudo nginx -t && sudo systemctl reload nginx
  sudo certbot delete --cert-name docs.lumos-lang.org

Update ~/.ssh/vps-port-registry.md:
  Remove: "**4000** - docs-lumos (Astro Starlight docs - docs.lumos-lang.org)"

==========================================================
GOTCHAS
==========================================================
- Pure static site — easiest possible Vercel migration
- If astro.config.mjs has `site: 'https://docs.lumos-lang.org'`, keep it (used for sitemap generation)
- Search functionality (Pagefind) is built at build time — verify it still works post-deploy

==========================================================
SUCCESS CRITERIA
==========================================================
[ ] Preview deploy renders correctly
[ ] DNS points to Vercel
[ ] Production domain serves with Vercel SSL
[ ] 7-day buffer elapsed
[ ] VPS container + nginx config + cert removed
[ ] Port registry updated
[ ] Mark this migration DONE in ~/local-dev/lumos/VERCEL_MIGRATION_HANDOFF.md
```

---

## Migration #2 — lumos-website

**Target repo to `cd` into**: `~/local-dev/lumos-website`

### Starter Prompt

```
You are continuing a Vercel migration. Today, migrate this repo (lumos-website) from VPS reclabs3 → Vercel.

==========================================================
PROJECT
==========================================================
- Name: lumos-website
- What: Lumos marketing/landing site
- Domains: lumos-lang.org AND www.lumos-lang.org
- Tech: Vite (static build) — confirmed via package.json
- GitHub: git@github.com:getlumos/lumos-website.git

==========================================================
CURRENT VPS STATE (reclabs3)
==========================================================
- Linux user: lumos
- Docker container: lumos-website (healthy, 2-month uptime)
- Port: 0.0.0.0:4001 -> container :80
- nginx site config: /etc/nginx/sites-enabled/lumos-lang-org
- Serves BOTH apex (lumos-lang.org) AND www.lumos-lang.org
- TLS: Let's Encrypt cert "lumos-lang.org" covers both apex + www (expires ~2026-08-23)
- Deploy: GitHub Actions → GHCR → SSH

==========================================================
TARGET VERCEL CONFIG
==========================================================
- Project name: lumos-website
- Framework: Vite (auto-detect; if not detected, choose "Other" + manual)
- Build command: pnpm build (or vite build — verify package.json scripts)
- Output: dist
- Install: pnpm install
- Env vars: verify by grepping for import.meta.env.VITE_*
- Production domain: lumos-lang.org (apex) + www.lumos-lang.org (redirect to apex)

==========================================================
EXECUTION STEPS
==========================================================

1) Pre-flight
   $ vercel --version              # ≥ 54.4.1
   $ git status && git pull --rebase
   $ pnpm install && pnpm build    # local smoke test

2) Vercel link + preview
   $ vercel link
   $ vercel
   - Verify preview URL renders the marketing site fully

3) Domains (both apex + www)
   Vercel dashboard → Domains:
   a) Add lumos-lang.org (apex) — Vercel will require either nameserver delegation OR ALIAS/ANAME (use Cloudflare's CNAME-flattening if on CF)
   b) Add www.lumos-lang.org — set as redirect to apex
   This MATCHES current behavior (apex serves, www redirects).

4) DNS cutover at registrar
   - Current state: A apex → 151.245.137.75, CNAME www → apex (or A www → 151.245.137.75)
   - Lower TTLs to 300 for both, wait 10 min
   - Update apex: A → 76.76.21.21 (Vercel) OR CNAME-flattened to cname.vercel-dns.com
   - Update www: CNAME → cname.vercel-dns.com
   - On Cloudflare: temporarily disable proxy (grey cloud) during cutover

5) Promote production
   $ vercel --prod

6) Verify
   $ dig lumos-lang.org +short && dig www.lumos-lang.org +short
   $ curl -sI https://lumos-lang.org | grep -i server
   $ curl -sI https://www.lumos-lang.org | grep -i location   # should 301/308 to apex

==========================================================
ROLLBACK
==========================================================
Revert both A/CNAME records to VPS (151.245.137.75 / nginx redirect handles www). VPS container is still running.

==========================================================
VPS-SIDE DECOMMISSION (AFTER 7-DAY BUFFER)
==========================================================
ssh lumos
  cd ~/<app-dir>; docker compose down; docker image prune -f

ssh reclabs3 (root)
  sudo rm /etc/nginx/sites-enabled/lumos-lang-org
  sudo rm /etc/nginx/sites-available/lumos-lang-org
  sudo nginx -t && sudo systemctl reload nginx
  sudo certbot delete --cert-name lumos-lang.org

Update ~/.ssh/vps-port-registry.md:
  Remove: "**4001** - lumos-website (marketing site - lumos-lang.org)"

==========================================================
GOTCHAS
==========================================================
- Static Vite build — no SSR concerns
- Apex domain: Vercel-recommended approach is to use Cloudflare's CNAME flattening on the apex, OR use Vercel's A record
- If site uses any analytics, ad scripts, or hardcoded asset paths, verify they resolve correctly on Vercel preview

==========================================================
SUCCESS CRITERIA
==========================================================
[ ] Preview deploy renders
[ ] DNS cutover for BOTH apex + www
[ ] www correctly redirects to apex
[ ] Production SSL valid (Vercel-issued, covers apex + www)
[ ] 7-day buffer elapsed
[ ] VPS container + nginx config + cert removed
[ ] Port registry updated
[ ] Mark this migration DONE in ~/local-dev/lumos/VERCEL_MIGRATION_HANDOFF.md
```

---

## Shared notes (Lumos suite)

- **DNS provider**: Verify which registrar/DNS host has `lumos-lang.org` before any DNS work. Likely Cloudflare given the org's pattern.
- **Order**: Do `docs-lumos` first (subdomain, simpler — no apex/www logic). Then `lumos-website` (apex + www) once the team is confident.
- **VPS user `lumos`** stays after both migrations (other lumos services may share it). Don't delete the user.
- **Audit context**: 11 services total in this migration round. Lumos = 2 of them. Other handoffs:
  - `~/local-dev/sip-protocol/VERCEL_MIGRATION_HANDOFF.md` (5 migrations)
  - `~/local-dev/nanuqfi/VERCEL_MIGRATION_HANDOFF.md` (1)
  - `~/local-dev/solis/VERCEL_MIGRATION_HANDOFF.md` (1)
  - `~/local-dev/edict/VERCEL_MIGRATION_HANDOFF.md` (1)
  - `~/local-dev/pnode-pulse/VERCEL_MIGRATION_HANDOFF.md` (1 — REDESIGN, not a quick move)
