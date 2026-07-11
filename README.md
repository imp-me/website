# theimp.me

The public site for **imp.me**, served at <https://theimp.me>. Right now it's a
single **coming soon** page — but its real job is to host the Android **App Links**
verification file so magic-link sign-in and password-reset emails open the app
instead of a dead browser tab.

## What's here

| Path | Purpose |
|------|---------|
| `index.html` | Coming-soon page (dark, on-brand, imp voice). Self-contained; Noto Sans bundled under `fonts/`. |
| `.well-known/assetlinks.json` | **Android App Links** Digital Asset Links statement. This is the reason the site must exist. |
| `favicon.svg` | The imp mark, amber on dark. |
| `staticwebapp.config.json` | Azure Static Web Apps config — pins `.json`/`.svg`/`.ttf` MIME types and serves `assetlinks.json` as `application/json`. |
| `robots.txt` | Allow-all. |

## The one thing you must finish: the signing fingerprint

`.well-known/assetlinks.json` currently has a **placeholder** fingerprint:

```
REPLACE_WITH_SHA256_OF_DISTRIBUTED_APK_SIGNING_CERT
```

App Links verification will fail until this is the real SHA-256 of the certificate
that signs the APK users actually install. The imp.me Android app currently signs
**both** debug and release builds with the debug keystore, and CI distributes the
debug APK via Firebase App Distribution — so the fingerprint you need is the one on
**that** APK, not a local dev machine's debug keystore.

Get it straight from a distributed APK:

```bash
apksigner verify --print-certs imp-me-staging.apk    # use the "SHA-256 digest"
# format as colon-separated uppercase hex, e.g. AB:CD:EF:...
```

Paste it into the `sha256_cert_fingerprints` array. Multiple entries are allowed —
add the release/upload key's fingerprint too once the app moves to a real signing key
(e.g. Play App Signing).

Validate with Google's [Statement List Tester](https://developers.google.com/digital-asset-links/tools/generator)
and, on a device, `adb shell pm get-app-links me.imp.mobile`.

## Hosting — Azure Static Web Apps

Hosted on **Azure Static Web Apps** (same cloud as the imp.me backend + admin UI),
deployed from this repo. There is no app build step — it's plain static files, so the
app location is the repo root and the output location is empty.

Setup (needs Azure portal + DNS access):

1. **Create the Static Web App** (Azure portal → Create a resource → Static Web App),
   linked to `imp-me/website`, branch `main`. Build presets: **Custom**;
   app location `/`, output location *(blank)*. Azure commits a deploy workflow to the
   repo and needs the `AZURE_STATIC_WEB_APPS_API_TOKEN` secret (it adds this for you).
2. **Custom domain** (portal → the SWA → Custom domains → add `theimp.me`). Validate
   via the TXT record Azure gives you, then point the apex at the SWA:
   - apex `theimp.me` → `ALIAS`/`A` per Azure's instructions (SWA supports apex domains),
     or host DNS in **Azure DNS** to keep it alongside the rest of the infra.
   - `www` → `CNAME` to the SWA default hostname (optional).
   Managed TLS is issued automatically once validation passes.
3. **Verify** `https://theimp.me/.well-known/assetlinks.json` returns `200` with
   `Content-Type: application/json` and **no redirect** (App Links won't follow one).
   `staticwebapp.config.json` already pins the content-type.

> The domain is registered at GoDaddy; you can keep the registrar there and just point
> DNS at Azure, or move DNS into Azure DNS — either works for App Links.

## Related

- Android deep-link fix + full checklist: `imp-me/imp.me.android` issue #345 and
  `docs/plans/345-fix-deep-link-app-links-hosting.md` in that repo.
