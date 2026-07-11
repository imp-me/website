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

`.well-known/assetlinks.json` holds the SHA-256 of the certificate that signs the APK
users actually install. It is currently the **Android Debug** cert
(`C=US, O=Android, CN=Android Debug`), because the imp.me Android app signs **both**
debug and release builds with the debug keystore (`app/build.gradle.kts`) and CI
distributes the debug APK via Firebase App Distribution. Extracted from the published
staging APK:

```bash
apksigner verify --print-certs imp-me-staging.apk    # "SHA-256 digest"
# → 09:10:10:74:2E:C0:A2:05:1D:B3:A4:DC:C0:D4:1B:91:70:0A:BD:57:EE:5F:79:09:4F:64:0C:82:BC:3F:DD:22
```

**When this must be updated:**

- The debug keystore lives on the self-hosted CI runner. If that runner is
  reprovisioned (or its `~/.android/debug.keystore` regenerated), the fingerprint
  changes and this file must be updated, or verification breaks.
- When the app moves to a real release/upload key (e.g. Play App Signing), add that
  key's SHA-256 to the `sha256_cert_fingerprints` array (multiple entries are allowed).

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
