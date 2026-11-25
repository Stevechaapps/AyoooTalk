# Publishing Ayooo on GitHub

This is the operator checklist for the release-only repository
`Stevechaapps/AyoooTalk`. The app source stays private; this repository publishes
binaries, checksums, documentation, and issues.

Replace `Stevechaapps/AyoooTalk` if the repository name changes.

---

## 0. Decide the signing key before the first public release

This is the one decision that is painful to reverse.

Check what signed the current APK:

```powershell
# Android SDK build-tools
apksigner verify --print-certs .\Ayooo.apk

# Or, without build-tools
keytool -printcert -jarfile .\Ayooo.apk
```

If the certificate says `CN=Android Debug`, the build is debug-signed.

Options:

- **Rotate to a release key now.** Best for trust and for any future Play
  distribution. Existing installs cannot update in place across a key change;
  users must uninstall and reinstall, losing local data. Do it before promotion
  while the install base is small.
- **Keep the existing key.** Updates keep working, but the build is not
  Play-ready and the trust signal is weak.

If you rotate, create the keystore outside the repository, wire
`android/key.properties` and a `signingConfig` in `app/android/app/build.gradle.kts`,
and publish the certificate's SHA-256 fingerprint in the release notes. Never
commit the keystore or its passwords.

---

## 1. Build the release

```powershell
cd app
flutter clean
flutter pub get

# Dart AOT obfuscation. The symbol files are the key back to readable stack
# traces: keep them private, never upload them.
flutter build apk --release --obfuscate --split-debug-info=build/symbols/universal
flutter build apk --release --split-per-abi --obfuscate --split-debug-info=build/symbols/split
```

Release builds also run R8 (Java/Kotlin minify) and resource shrinking; the
keep rules live in `android/app/proguard-rules.pro`. Archive the R8 mapping
(`app/build/app/outputs/mapping/release/mapping.txt`) and both symbol
directories in a private place. They de-obfuscate crash reports and must never
be published with the APK.

Outputs in `app/build/app/outputs/flutter-apk/`:

| Built file | Rename to | Purpose |
| --- | --- | --- |
| `app-release.apk` | `Ayooo.apk` | Universal; recommended download |
| `app-arm64-v8a-release.apk` | `Ayooo-arm64-v8a.apk` | Most phones since ~2017 |
| `app-armeabi-v7a-release.apk` | `Ayooo-armeabi-v7a.apk` | Older 32-bit phones |

`x86_64` is deliberately excluded from packaging in
`app/android/app/build.gradle.kts`, so the x86_64 split artifact is broken and
must not be published. Flutter still emits it during `--split-per-abi`; ignore
it.

**The asset names must stay identical across releases.** The download links in
the README and on the site use:

```text
https://github.com/Stevechaapps/AyoooTalk/releases/latest/download/Ayooo.apk
```

GitHub resolves that by asset name on the latest release. Rename an asset and
the link 404s.

Stage the files:

```powershell
$v = "1.4.10"
$out = "..\release\v$v"
New-Item -ItemType Directory -Force $out | Out-Null

Copy-Item build\app\outputs\flutter-apk\app-release.apk          "$out\Ayooo.apk"
Copy-Item build\app\outputs\flutter-apk\app-arm64-v8a-release.apk "$out\Ayooo-arm64-v8a.apk"
Copy-Item build\app\outputs\flutter-apk\app-armeabi-v7a-release.apk "$out\Ayooo-armeabi-v7a.apk"
```

Generate SHA-256 checksums with LF line endings, so `sha256sum -c` works on
Linux and macOS:

```powershell
$lines = Get-ChildItem "$out\*.apk" | ForEach-Object {
  $h = (Get-FileHash $_.FullName -Algorithm SHA256).Hash.ToLower()
  "$h  $($_.Name)"
}
[IO.File]::WriteAllText("$out\SHA256SUMS.txt", ($lines -join "`n") + "`n")
```

Test before publishing: install the universal and the arm64 builds on a real
phone, complete a call, check the version in Settings, and confirm the in-app
update card. On the signup screen, confirm the no-recovery warning is visible,
`Create account` stays disabled until the acknowledgement is ticked, and the
release build launches with R8/obfuscation enabled.

---

## 2. Create the repository

```powershell
cd ..\github-release
git init
git add .
git commit -m "Ayooo 1.4.10"
gh repo create Stevechaapps/AyoooTalk --public --source=. --remote=origin --push
```

If the repository already exists:

```powershell
git remote add origin https://github.com/Stevechaapps/AyoooTalk.git
git branch -M main
git push -u origin main
```

---

## 3. Create the release

```powershell
gh release create v1.4.10 `
  ..\release\v1.4.10\Ayooo.apk `
  ..\release\v1.4.10\Ayooo-arm64-v8a.apk `
  ..\release\v1.4.10\Ayooo-armeabi-v7a.apk `
  ..\release\v1.4.10\SHA256SUMS.txt `
  --title "Ayooo 1.4.10 — free encrypted calls for Android" `
  --notes-file docs/release-notes/v1.4.10.md
```

Release title guidance: include the brand, the version, and one concrete
keyword phrase. Do not stuff keywords; GitHub and search engines both prefer a
readable title.

---

## 4. Point the product at GitHub (website stays primary)

The website remains the front door. It keeps the stable, branded URL
`https://ayoootalk.com/downloads/ayooo.apk`; the Worker and static redirect send
that URL to the current GitHub Release asset. R2 is no longer the APK origin.

Keep `app/lib/config.dart` `appHomepage` at `https://ayoootalk.com`. The app
fetches `<homepage>/downloads/latest.json`, and the help, terms, and donate
links also depend on that host. Do **not** point `appHomepage` at GitHub.

Keep the in-app update card on the website:

`worker/public/downloads/latest.json`

```json
{"version": "1.4.10", "homepage": "https://ayoootalk.com/"}
```

The card opens the site; the site's Download button resolves to GitHub. If you
would rather the card open the GitHub release page, change `homepage` to
`https://github.com/Stevechaapps/AyoooTalk/releases/latest` — but the site remains
the primary entry point either way.

Point the branded download URL at GitHub:

`worker/public/_redirects`

```text
/downloads/ayooo.apk https://github.com/Stevechaapps/AyoooTalk/releases/latest/download/Ayooo.apk 302
/downloads/call-core-arm64.apk https://github.com/Stevechaapps/AyoooTalk/releases/latest/download/Ayooo.apk 302
/downloads/call-core-arm32.apk https://github.com/Stevechaapps/AyoooTalk/releases/latest/download/Ayooo.apk 302
/downloads/call-core.apk https://github.com/Stevechaapps/AyoooTalk/releases/latest/download/Ayooo.apk 302
```

The Worker has the same redirect as a fallback (`worker/src/index.js`), so the
URL works even if the static redirect is bypassed.

Update the site's download links and structured data:

- `worker/public/index.html` — the three Download links keep the branded
  `/downloads/ayooo.apk` URL. Remove the `download` attribute: the response is
  a cross-origin 302 to GitHub, and GitHub sends
  `Content-Disposition: attachment` itself.
- JSON-LD `SoftwareApplication`:
  - `downloadUrl` → `https://ayoootalk.com/downloads/ayooo.apk` (the stable,
    branded link).
  - `fileSize` → the universal APK size (currently `27MB`).
  - `softwareVersion` → `1.4.10`.
  - `sameAs` → `https://github.com/Stevechaapps/AyoooTalk`.

R2 is decommissioned. `worker/wrangler.jsonc` no longer declares a binding and
`worker/src/index.js` no longer reads `env.DOWNLOADS`. The `ayooo-downloads`
bucket was deleted on 2026-11-25 after the GitHub redirect was verified; do not
recreate it.

> **Do this only after the GitHub release is live.** The site's download URL is
> now a redirect to `releases/latest/download/Ayooo.apk`. If you deploy the
> Worker before the repository and release asset exist, every download on
> ayoootalk.com returns a GitHub 404. Create the repo, publish the release, and
> verify the asset URL first; then deploy.

Deploy the Worker:

```powershell
cd worker
npx wrangler deploy
```

Verify the chain:

```powershell
curl.exe -I https://ayoootalk.com/downloads/ayooo.apk
curl.exe -I -L https://ayoootalk.com/downloads/ayooo.apk
curl.exe https://ayoootalk.com/downloads/latest.json
```

The first must return `302` with a `Location:` on
`github.com/Stevechaapps/AyoooTalk/releases/latest/download/Ayooo.apk`. The second
must end at GitHub with `200`. The third must still advertise the new version
and the site homepage.

---

## 5. Set the repository metadata

```powershell
gh repo edit Stevechaapps/AyoooTalk `
  --description "Free, encrypted voice and video calls for Android - 1:1 over data, no phone number, no ads. APK releases." `
  --homepage "https://ayoootalk.com" `
  --add-topic android `
  --add-topic webrtc `
  --add-topic video-call `
  --add-topic voice-call `
  --add-topic voip `
  --add-topic flutter `
  --add-topic apk `
  --add-topic p2p `
  --add-topic encrypted `
  --add-topic low-bandwidth `
  --add-topic philippines `
  --add-topic sideload
```

Then, in **Settings**:

- **Social preview:** upload `docs/og.png` (1200×630; 1280×640 is ideal).
- **Features:** enable Issues. Discussions are optional; use them only if you
  will answer them.
- **About:** confirm the website and topics.
- **Pin the repository** on your profile.

---

## 6. SEO and GEO

Traditional SEO gets the page found. Generative engine optimization (GEO) gets
it quoted by ChatGPT, Perplexity, Gemini, and AI Overviews. Both depend on the
same thing: specific, consistent, verifiable facts.

Do these in order:

1. **Keep one fact table.** Version, size, Android version, price, phone-number
   requirement, encryption wording, and supported call types must match across
   the README, the site, `llms.txt`, and release notes. Mismatches get filtered
   out by answer engines.
2. **Use question-shaped headings.** The FAQ already does this. Add the same
   questions to the site's help page if they are not there.
3. **Add `FAQPage` JSON-LD to the site.** The site already has `WebSite` and
   `SoftwareApplication`; a `FAQPage` block can win rich results and gives
   crawlers explicit Q&A pairs.
4. **Put `softwareVersion` and `downloadUrl` in the site's JSON-LD.** They are
   the fields answer engines use to cite a current release.
5. **Submit the sitemap** in Google Search Console and Bing Webmaster Tools
   after deploying. GitHub indexes the repository on its own, but the site
   needs a nudge.
6. **Link both ways with descriptive anchors.** Site → "Ayooo on GitHub"
   (not "click here"); README → website and help. GitHub links are nofollow, but
   they still drive traffic and corroborate the entity.
7. **Publish `llms.txt`.** It is already in this repository. Keep the "Last
   updated" date current.
8. **Date-stamp claims.** "As of November 2026" beats an undated superlative.
9. **Avoid unverifiable superlatives.** "Best video calling app" is noise.
   "24 kbps audio, 240×320 video, direct media path" is quotable.
10. **Earn third-party mentions.** A listing in a "best free calling apps"
    roundup or a PH tech article is worth more than ten keywords. Pitch the
    specific angle: no phone number, prepaid-data friendly, direct media.

---

## 7. Launch distribution

GitHub alone will not reach mainstream users. Use it as the trust surface and
point communities at it.

- **GitHub:** pin the repo, publish the release, keep the README current.
- **Reddit:** `r/AndroidApps` (follow their self-promotion rules),
  `r/Philippines`, `r/InternetPH`, `r/privacy`. Post as the maker, state the
  limitations, answer questions.
- **Hacker News:** a factual `Show HN: Ayooo – free 1:1 calls over data, no
  phone number` works better than marketing copy. Expect scrutiny of the
  "encrypted" and "no relay" claims; the README already addresses both.
- **Product Hunt:** schedule a launch; use `docs/og.png` as the thumbnail.
- **X / Facebook / PH tech groups:** short demo clip beats a screenshot. Show a
  call surviving a throttled connection, or the rescue-mode camera pause.
- **Press:** Yugatech, Gadget Pilipinas, and Jam Online cover PH telco topics.
  Pitch the weak-data/prepaid angle and the ViLTE limitation honestly.

Do not astroturf, buy reviews, or seed fake "best app" lists. Answer engines
and communities both punish it.

---

## 8. Measure

- **GitHub → Insights → Traffic:** views, clones, referrers.
- **Release asset download counts:** universal vs arm64 tells you how technical
  your audience is.
- **Site analytics:** Cloudflare Web Analytics is enough; keep it off the app.
- **Issues:** install failures and OEM battery problems are the real support
  load. Add them to the help page as they recur.
- **Search Console:** queries that already bring impressions. Write the help
  page that answers them.

---

## 9. Known inconsistencies to fix before promotion

- **Version sync:** the site, `latest.json`, JSON-LD, and `pubspec.yaml` must all
  say `1.4.10`; the hero line and JSON-LD are now `1.4.10` / `27 MB`. Bump all
  of them on every release.
- **"Encrypted" wording:** the site says "nobody in the middle"; the signaling
  server necessarily relays SDP and metadata. "Media is direct and encrypted;
  the server rings the bell" is accurate and still strong.
- **"No relay, ever":** true for the public STUN-only service, but the code is
  TURN-capable. Keep the public claim scoped to the public service.
- **Debug signing:** resolve before the first public release (see step 0).
- **R2 decommissioned:** the `DOWNLOADS` binding and the R2 route are removed,
  and the `ayooo-downloads` bucket was deleted on 2026-11-25. GitHub Releases
  is the only APK origin.
- **`download` attribute:** removed from the site's links because the final
  response is a cross-origin 302 to GitHub. GitHub sends
  `Content-Disposition: attachment` itself.
- **Stale README claims:** the root development README still describes
  `usesCleartextTraffic="true"` and the old LAN default. Keep development docs
  separate from this public README.
