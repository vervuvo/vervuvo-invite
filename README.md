# Vervuvo invite web landing + deep-link verification

Static assets that make shareable duel invite links work for people who don't
have the app yet, and that let iOS/Android route `https://link.vervuvo.com/duel/<id>`
straight into the installed app.

**Hosting decision:** served from **GitHub Pages** on the `link.vervuvo.com`
subdomain — mirroring how `terms.vervuvo.com` is already hosted. The apex
`vervuvo.com` stays on GoDaddy's Websites + Marketing builder (untouched).
The host is wired into the app via [`constants/links.ts`](../constants/links.ts)
and `app.json`. If you change the subdomain, update **all three** (`constants/links.ts`,
`app.json` → `ios.associatedDomains` + `android.intentFilters.host`) **and**
the `CNAME` file here.

## Files in this folder

| File | Served at | Purpose |
|------|-----------|---------|
| `.well-known/apple-app-site-association` | `https://link.vervuvo.com/.well-known/apple-app-site-association` | iOS Universal Links verification (no extension — that's correct). |
| `.well-known/assetlinks.json` | `https://link.vervuvo.com/.well-known/assetlinks.json` | Android App Links verification. |
| `duel/index.html` | `https://link.vervuvo.com/duel/` | Landing page (clean serving + portability to non-GitHub hosts). |
| `404.html` | any unmatched path, incl. `https://link.vervuvo.com/duel/<id>` | **GitHub Pages catch-all.** GitHub can't do wildcard routing, so `/duel/<id>` (no real file) falls through to `404.html`, whose JS reads the duel id from the URL. Keep it identical to `duel/index.html`. |
| `CNAME` | — | Tells GitHub Pages the custom domain is `link.vervuvo.com`. |

## Deploy steps (mirrors your terms.vervuvo.com setup)

1. **Create a GitHub repo** for the invite site (e.g. `vervuvo-invite`). It must
   be separate from the terms repo — GitHub Pages allows one custom domain per
   repo. Copy the entire contents of this `web/` folder into the repo root
   (so `404.html`, `CNAME`, `duel/`, and `.well-known/` sit at the top level).

2. **Enable GitHub Pages:** repo → Settings → Pages → Source = `Deploy from a
   branch`, Branch = `main` / root. The `CNAME` file sets the custom domain to
   `link.vervuvo.com`. Tick **Enforce HTTPS** (GitHub auto-provisions a free
   Let's Encrypt cert; HTTPS is *required* for universal links).

3. **GoDaddy DNS:** in the vervuvo.com DNS zone, add a **CNAME** record —
   Name/Host = `link`, Value = `<your-github-username>.github.io`. (Same kind of
   record you added for `terms`.) Leave the apex `vervuvo.com` records alone so
   your GoDaddy site keeps working. DNS + cert can take 10–60 min.

4. **Fill the placeholders** (search for `REPLACE_`):
   - `.well-known/apple-app-site-association` → `REPLACE_APPLE_TEAM_ID` — your
     Apple Developer Team ID (10 chars, e.g. `A1B2C3D4E5`); from Apple Developer
     → Membership, or `eas credentials`. Result: `A1B2C3D4E5.com.vervuvo.app`.
   - `.well-known/assetlinks.json` → `REPLACE_WITH_ANDROID_SHA256_CERT_FINGERPRINT`
     — SHA-256 of the **app-signing** cert; Play Console → App integrity → App
     signing, or `eas credentials` (Android). Colon-separated hex. List multiple
     if you also use an upload key.
   - `404.html` **and** `duel/index.html` → `REPLACE_APP_STORE_URL`,
     `REPLACE_GOOGLE_PLAY_URL`, and optionally `REPLACE_SUPABASE_URL` /
     `REPLACE_SUPABASE_ANON_KEY` (the public `EXPO_PUBLIC_*` values — filling
     them shows the real challenge title on the page).

5. **Rebuild the native apps** after the `app.json` change. `associatedDomains` /
   `intentFilters` are compiled into the binary — an OTA update will *not* apply
   them.

## Verifying after deploy

- iOS: `curl -i https://link.vervuvo.com/.well-known/apple-app-site-association`
  → expect `200`, JSON body, **no redirect**.
- Android: `curl https://link.vervuvo.com/.well-known/assetlinks.json` → `200`
  JSON; then run it through Google's Statement List Tester.
- Landing: open `https://link.vervuvo.com/duel/test123` in a browser — you should
  see the challenge card with store buttons (a 404 *status* is expected/fine on
  GitHub Pages; the page still renders).

## Note on GitHub Pages + the AASA content-type

GitHub Pages serves the extensionless `apple-app-site-association` as
`application/octet-stream`, not `application/json`. Modern iOS fetches the AASA
through Apple's CDN, which tolerates this, so universal links work in practice.
If you ever hit verification trouble, that's the first thing to rule out — and
the fallback is to move just the `.well-known` files to a host where you can set
`Content-Type: application/json` (e.g. a Supabase/Cloudflare function).

## Known gap (deferred deep linking)

A brand-new user who taps the link, installs, then opens the app loses the duel
id (the OS doesn't hand the URL to a fresh install) and lands on home. Fully
closing this needs a deferred-deep-link service (Branch/AppsFlyer) — deferred.
The page's "reopen this link to accept" hint is the interim workaround: once
installed + signed in, tapping the link again routes correctly.
