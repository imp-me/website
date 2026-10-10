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

## The signing fingerprint

`.well-known/assetlinks.json` holds the SHA-256 of the certificate that signs the APK
users actually install. It is currently the **CI signing key**. Android CI signs every
distributed build with one keystore, supplied by the `ANDROID_KEYSTORE_BASE64` repo
secret in `imp-me/imp.me.android` (see imp.me.android#436), and ships the staging APK
through Firebase App Distribution. Read it off the staging APK:

```bash
apksigner verify --print-certs imp-me-staging.apk    # "SHA-256 digest"
# → AA:E3:17:43:6C:5C:44:A3:40:E4:BE:DE:9C:A6:FD:1C:4F:BF:A3:B8:C4:73:5B:FD:5F:E7:EC:95:07:74:D2:37
```

The CI "Guard — verify staging APK signer" step reads the same value and refuses to
distribute when it doesn't match the `ANDROID_SIGNING_SHA256` secret. The older
`09:10:10:74…` entry was one self-hosted runner's debug key; it was removed in #1 once
every runner signed with the shared keystore.

**When this must be updated:**

- **The CI keystore changes.** If `ANDROID_KEYSTORE_BASE64` is ever replaced, the
  fingerprint changes. Update this file and `ANDROID_SIGNING_SHA256` together, or
  magic links stop opening the app for new builds.
- **The app ships on Google Play.** Play App Signing re-signs Play installs with a key
  Google creates on the first upload, so those installs carry a different certificate.
  Copy the **app signing key** SHA-256 from Play Console → Setup → App integrity → App
  signing (the page also offers a ready-made Digital Asset Links snippet) and add it as
  a second entry in `sha256_cert_fingerprints`. Keep the CI entry for as long as
  Firebase test builds are signed with the CI key.

Local builds sign with the developer's own `~/.android/debug.keystore`, so App Links
don't verify on them. That's expected.

Validate with Google's [Statement List Tester](https://developers.google.com/digital-asset-links/tools/generator)
and, on a device, `adb shell pm get-app-links me.imp.mobile`.

## Hosting — GitHub Pages

Hosted on **GitHub Pages**, served from this repo's `main` branch (root), custom
domain `theimp.me`. Plain static files — no build step.

Setup (needs repo admin + DNS access):

1. **Enable Pages** — repo → Settings → Pages → Source **Deploy from a branch**,
   branch `main`, folder `/ (root)`. Save.
2. **DNS** (registrar + DNS: **Namecheap**, Advanced DNS tab) — point the apex `theimp.me`
   at GitHub Pages with four `A` records (and optional `AAAA` for IPv6):
   ```
   A Record    @    185.199.108.153
   A Record    @    185.199.109.153
   A Record    @    185.199.110.153
   A Record    @    185.199.111.153
   CNAME       www  imp-me.github.io.
   ```
   Remove any pre-existing Namecheap parking / URL-redirect record on `@` or `www` first.
   (Namecheap's editor sometimes visually drops a freshly added row on refresh even though
   it saved — trust the live DNS, not the editor.)
3. **Custom domain** — Settings → Pages → Custom domain → `theimp.me` → Save (this
   writes/confirms the `CNAME` file). Once DNS resolves, tick **Enforce HTTPS**
   (GitHub issues a Let's Encrypt cert automatically).
4. **Verify** `https://theimp.me/.well-known/assetlinks.json` returns `200`,
   `Content-Type: application/json`, and **no redirect** (App Links won't follow one).
   `.nojekyll` ensures the `.well-known/` folder is published.

> The domain is registered at **Namecheap**; manage records under Domain List →
> `theimp.me` → Manage → **Advanced DNS**. No nameserver change needed.

## Related

- Android deep-link fix + full checklist: `imp-me/imp.me.android` issue #345 and
  `docs/plans/345-fix-deep-link-app-links-hosting.md` in that repo.
