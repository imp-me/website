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
| `.nojekyll` | Disables Jekyll on GitHub Pages so the `.well-known/` folder is served (Jekyll ignores dot-folders). **Do not delete.** |
| `CNAME` | Binds the GitHub Pages site to the `theimp.me` custom domain. |
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

Hosted on **GitHub Pages**, served from this repo's `main` branch (root), custom
domain `theimp.me`. Plain static files — no build step.

Setup (needs repo admin + DNS access):

1. **Enable Pages** — repo → Settings → Pages → Source **Deploy from a branch**,
   branch `main`, folder `/ (root)`. Save.
2. **DNS** (registrar: GoDaddy) — point the apex `theimp.me` at GitHub Pages with four
   `A` records (and optional `AAAA` for IPv6):
   ```
   A    @   185.199.108.153
   A    @   185.199.109.153
   A    @   185.199.110.153
   A    @   185.199.111.153
   ```
   Remove any pre-existing GoDaddy parking/forwarding record on `@` first.
3. **Custom domain** — Settings → Pages → Custom domain → `theimp.me` → Save (this
   writes/confirms the `CNAME` file). Once DNS resolves, tick **Enforce HTTPS**
   (GitHub issues a Let's Encrypt cert automatically).
4. **Verify** `https://theimp.me/.well-known/assetlinks.json` returns `200`,
   `Content-Type: application/json`, and **no redirect** (App Links won't follow one).
   `.nojekyll` ensures the `.well-known/` folder is published.

> The domain is registered at GoDaddy. Keep the registrar there and just point the
> apex `A` records at GitHub — no need to move nameservers.

## Related

- Android deep-link fix + full checklist: `imp-me/imp.me.android` issue #345 and
  `docs/plans/345-fix-deep-link-app-links-hosting.md` in that repo.
