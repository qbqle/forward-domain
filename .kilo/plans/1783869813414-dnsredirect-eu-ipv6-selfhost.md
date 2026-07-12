# Self-hosting `forward-domain` under `dnsredirect.eu` (IPv6-only)

## Goal
Run the existing `forward-domain` service (willnode/forward-domain) as a public
domain-forwarding service on a Hetzner private server, reachable only over
IPv6, branded as `dnsredirect.eu`. Scope is **minimal**: configuration + doc
fixes only, **no new source code**.

## Key facts from the codebase (verified)
- Forwarding logic is generic. Users' `_.{domain}` / `fwd.{domain}` TXT records
  drive redirects; **no service domain is baked into the redirect path**.
  (`src/util.js:283` `findTxtRecord`, `src/client.js:61`)
- ACME `http-01` challenge is served on the plain HTTP server at
  `/.well-known/acme-challenge/` for any host (`src/client.js:85`, `:93`), so
  port 80 must reach the app for each certified domain.
- Let's Encrypt directory is chosen by `NODE_ENV` in
  `src/certnode/lib/common.js:5-9`. Default is **`development` = Let's Encrypt
  STAGING (fake certs)**. `npm start` sets `NODE_ENV=production`
  (`package.json:16`), but a systemd/pm2 launch must set it too.
- `HOME_DOMAIN` enables `/stat`, `/health`, `/flushcache` (no other domain
  concept in code). `src/client.js:113`
- `newAccount()` is called with **no contact email** (`src/certnode/lib/client.js:252`,
  `sni.js:133`). Optional follow-up only.
- `app.js` calls `plainServer.listen(port)` / `secureServer.listen(port)` with no
  host → binds all interfaces (dual-stack). No code change needed for IPv6.
- `public/` is empty; there is **no landing page**. HOSTING.md references a
  `stat.js` that does not exist in this version (replaced by the `HOME_DOMAIN`
  `/stat` endpoint).

## Decisions
- Domain: `dnsredirect.eu`; CNAME target + `HOME_DOMAIN` = `r.dnsredirect.eu`.
- Network: **IPv6-only**. Server IPv6 `2a01:4f8:1c18:eed4::` (confirm host
  suffix, e.g. `::1`/`::2`, from Hetzner console). Publish **only AAAA**; no
  `A` record. Firewall-block inbound IPv4 so the dual-stack `listen()` never
  sees IPv4. LE validates over IPv6 (supported); outbound to `dns.google`/LE API
  uses IPv6.
- Deploy: Hetzner VPS, app binds 80/443 directly (root or `setcap`).

## Tasks (implementation-ready)
1. **`.env`** (copy from `.env.example`):
   - `HTTP_PORT=80`, `HTTPS_PORT=443`
   - `NODE_ENV=production` (critical — avoid staging certs)
   - `HOME_DOMAIN=r.dnsredirect.eu`
   - `WHITELIST_HOSTS=` (set your allowed roots) and/or
     `BLACKLIST_HOSTS=` + `BLACKLIST_REDIRECT="https://dnsredirect.eu/blacklisted"`
   - Optional: `USE_LOCAL_DNS=false`, `CACHE_EXPIRY_SECONDS=86400`
2. **`.env.example` / `.env.test`**: replace `forwarddomain.net` literals
   (`.env.example:5-6`, `.env.test:5`) with `dnsredirect.eu` / `r.dnsredirect.eu`.
3. **DNS for the service itself**:
   - `r.dnsredirect.eu` → `AAAA 2a01:4f8:1c18:eed4::…` (no A record)
   - Optionally `dnsredirect.eu` apex → `AAAA` (same) if used directly.
4. **User-facing DNS instructions** (for your docs / support):
   - Subdomain forward: `www.user.com  IN  CNAME  r.dnsredirect.eu`
   - Apex forward: `user.com  IN  AAAA  2a01:4f8:1c18:eed4::…` (no CNAME at apex)
   - TXT: `_.user.com  IN  TXT  "forward-domain=https://dest.example/*"`
   - CAA: `user.com  IN  CAA  0 issue "letsencrypt.org"` (or no CAA)
5. **`README.md`**: change CNAME target `r.forwarddomain.net` → `r.dnsredirect.eu`
   (`:36`, `:50`, `:55`) and apex `A 167.172.5.31` → `AAAA` to the IPv6
   (`:44`). Update IPv6 note (`:85`).
6. **`HOSTING.md`**: remove `167.172.5.31` / `forwarddomain.net` IPv4 literals
   (`:13`, `:42`, `:80-122`); replace NGINX example `listen` lines with IPv6
   (`listen [2a01:4f8:1c18:eed4::]:443;`). Delete the stale `stat.js` reference
   (`:42`) — `/stat` is served on `HOME_DOMAIN`. Update the clone URL.
7. **Run as a service**: systemd unit or `pm2` running `npm start`
   (sets `NODE_ENV=production`), as root or with
   `setcap cap_net_bind_service=ep $(command -v node)` so 80/443 bind unprivileged.
8. **Persist certs**: keep `.certs/` (holds LE account key + certs) on durable
   storage / volume; back it up.

## Risks / validation
- **Staging trap**: if `NODE_ENV` is not `production`, certs are from LE staging
  and browsers reject them. Verify with `openssl s_client -connect
  [2a01:4f8:1c18:eed4::]:443 -servername test.dnsredirect.eu` → must show a
  real LE cert (`R3`/ISRG, not `(STAGING)`).
- **IPv6-only reachability**: some clients lack IPv6; they cannot use the
  service. Acceptable per decision, but document it.
- **LE rate limits**: test one domain on `NODE_ENV=development` (staging)
  before switching to production.
- **Firewall**: confirm inbound IPv4 is dropped and IPv6 80/443 open; outbound
  IPv6 to `dns.google` + `acme-v02.api.letsencrypt.org` allowed.
- **CAA**: a user domain whose CAA forbids `letsencrypt.org` fails
  (`src/client.js:55`, `sni.js:70`). Document the CAA requirement.

## Out of scope (optional follow-ups, not minimal)
- `LE_EMAIL` env → pass to `newAccount()` for cert-expiry notices
  (`src/certnode/lib/client.js:252`).
- Landing/setup page in `public/`.
- Custom ACME CA support.
