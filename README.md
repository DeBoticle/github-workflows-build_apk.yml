# FieldFlow Pro

The job book for one-truck contractors. Built for HVAC, plumbing, electrical and handyman work:
create a job in under a minute, capture before/after photo evidence, build a quote that totals
itself, get the customer's signature on site, and hand over a PDF job packet — all with no signal.

This repository is a complete, standalone **Android** project (Kotlin + Jetpack Compose). It is not
embedded in anything else and does not depend on any hosted backend to run.

```
FieldFlowPro/
├── app/                     Android application module
│   └── src/
│       ├── main/            Kotlin sources, Compose UI, resources, bundled templates
│       └── test/            JVM unit tests for the money/total maths
├── server/                  Optional Node reference implementation of the sync API
├── assets/
│   ├── promo/               Store promo images (1920x1080 PNG, five scenes)
│   ├── play/                512x512 app icon + 1024x500 feature graphic
│   └── promo-src/           Re-runnable generator script + source photos for the art above
├── gradle/                  Version catalog + Gradle wrapper
├── BUILD.md                 Step-by-step APK/AAB build + Play Console checklist
├── PLAY_STORE_LISTING.md    Listing copy, data safety answers, image requirements
├── ARCHITECTURE.md          Module map, data flow, sync model
├── API_CONTRACT.md          HTTP contract for /v1/sync/push and /v1/sync/pull
├── PRIVACY_POLICY.md        Ready-to-host privacy policy (Play requires a public URL)
└── USER_GUIDE.md            Short end-user guide you can ship as help content
```

## Quick start

```bash
# 1. Point the build at your Android SDK
echo "sdk.dir=/path/to/Android/sdk" > local.properties

# 2. Debug build → run on a Samsung S23 (or any device on API 26+)
./gradlew assembleDebug
adb install -r app/build/outputs/apk/debug/app-debug.apk

# 3. Play-ready bundle
./gradlew bundleRelease
# → app/build/outputs/bundle/release/app-release.aab
```

Open the folder in Android Studio (Koala or newer) if you prefer the IDE: `File ▸ Open` and pick
the project root. Everything — Gradle wrapper, version catalog, manifest, launcher icon, themes —
is included, so there is nothing to generate by hand.

## Configuration (all optional)

The app builds and runs with **zero configuration**: it signs in locally, stores everything in an
encrypted on-device vault, and syncs against an on-device mirror. Add keys to `local.properties`
(or export them as environment variables) to switch on the cloud features:

```properties
# Google Sign-In (OAuth *Web* client id from Play Console / Cloud Console)
GOOGLE_WEB_CLIENT_ID=1234567890-abcdefg.apps.googleusercontent.com

# Optional: your own sync server. Leave blank to use the built-in on-device mirror.
SYNC_BASE_URL=https://sync.example.com
SYNC_API_KEY=long-random-shared-secret

# Optional: anonymous analytics collector (POSTs a JSON array of events)
ANALYTICS_ENDPOINT=https://analytics.example.com/fieldflow

# Optional: support address shown on the About screen
SUPPORT_EMAIL=support@example.com
```

For a signed release build, copy `keystore.properties.example` to `keystore.properties` and point it
at your upload key. Without it, the release build is signed with the debug key (fine for local
testing, not for Play).

## Feature map

| Requirement | Where it lives |
| --- | --- |
| Ultra-simple job creation (title, client, address, materials, cost) | `ui/screens/JobEditorScreen.kt`, `ui/screens/JobDetailScreen.kt` |
| Photo capture + markup + before/after comparison | `ui/screens/PhotosScreen.kt`, `MarkupScreen.kt`, `CompareScreen.kt`, `ui/components/Media.kt`, `data/photo/PhotoStore.kt` |
| Quote builder with auto-calculated totals | `ui/screens/QuoteBuilderScreen.kt`, `data/local/Repositories.kt` (`QuoteRepository`), `data/model/Models.kt` (`Quote`) |
| Offline-first (everything works with no internet) | Every screen reads/writes `EncryptedJsonStore` + `SyncQueueRepository` first; network is never on the critical path |
| Auto-sync when online | `data/sync/SyncEngine.kt`, `SyncWorker.kt` (WorkManager, 30-minute periodic + immediate on reconnect) |
| Client signature capture | `ui/screens/SignatureScreen.kt`, `ui/components/Media.kt` (`SignaturePad`) |
| Export job packets as PDF | `data/pdf/PacketExporter.kt` (hand-built PDF: cover, scope, materials, quote, photo grid, before/after, signature page) |
| Per-trade templates (HVAC, plumbing, electrical, handyman) | `app/src/main/assets/templates.json` + `data/templates/TemplateRepository.kt` |
| Dark mode + large-button glove UI | `ui/theme/Theme.kt` (light/dark schemes, `FfDimens` standard vs glove) + Settings ▸ Work style |
| Login (email + Google) | `data/local/AuthRepository.kt`, `data/auth/GoogleSignInClient.kt`, `ui/screens/LoginScreen.kt` |
| Settings page | `ui/screens/SettingsTab.kt` |
| Error handling | `core/AppResult.kt` (`AppError` + `AppResult`), `core/AppLogger.kt` (ring buffer + logcat), `DiagnosticsScreen.kt` |
| Local encrypted storage | `core/CryptoBox.kt` (Android Keystore AES-256-GCM) + `core/EncryptedJsonStore.kt` |
| Analytics events | `core/analytics/Analytics.kt` (local-first log, opt-in upload) + `AnalyticsScreen.kt` |
| Onboarding tutorial | `ui/screens/OnboardingScreen.kt` (6 steps: welcome, business, rates, appearance, offline, permissions) |

## Design notes

* **Offline is the default path, not a fallback.** A save is only "done" once it is encrypted into
  the local vault and an operation is queued. Sync is opportunistic.
* **No Room, no codegen.** Persistence is a small hand-rolled `JsonTable<T>` over an
  AES-256-GCM-encrypted JSON file per entity type. Fewer moving parts, trivially reviewable,
  and the encryption story is obvious.
* **Media is encrypted at rest.** Photos and signature PNGs are stored encrypted; a decrypted copy
  is written to the cache only when a screen needs to display it (`PhotoStore.displayFile`), and
  "Clear image cache" wipes those copies.
* **Glove mode is a dimension set, not a separate UI.** `LocalFfDimens` swaps 60dp targets for 84dp
  and scales type + money displays, so every screen gets bigger at once.

## Store artwork

Everything Google Play asks for is already in `assets/`:

| File | Use |
| --- | --- |
| `assets/promo/01-job-creation.png` … `05-signature.png` | 1920×1080 promo/screenshot images (16:9, well inside Play's 320–3840 px rule) |
| `assets/play/icon-512.png` | 512×512 Play listing icon (full-bleed, opaque — Play applies its own mask) |
| `assets/play/feature-graphic-1024x500.png` | 1024×500 feature graphic (required for a listing) |

`assets/promo-src/generate-promos.js` regenerates all seven files from scratch — it is plain
canvas code with no build step. It draws the layout, the phone mockups and the signature stroke
art, and it composites the three job photos in `assets/promo-src/photo-*.jpg` into the mock screens.
It also drops small JPEG previews in `assets/promo/preview/` so you can eyeball changes quickly;
those previews are debug output and are deliberately left out of the release zip.
Edit the copy, the brand colours or the photos there and re-run it to re-brand the whole set.

The file exports a single async function `({ fs, tools, only })`; `fs.writeFile(path, bytes)` and
`fs.readFile(path)` are the only host capabilities it needs, and `only` is an optional filter
(`"promos"` for the five screenshots, `"play"` for the icon + feature graphic, `"sigtest"` for a signature-pen test sheet). Run it from any environment with `OffscreenCanvas` and
`createImageBitmap` — a page console works if you sink the bytes to downloads instead of disk:

```js
const src = await (await fetch("assets/promo-src/generate-promos.js")).text();
const generate = eval(`(${src})`);
const saved = [];
await generate({ fs: {
  readFile: async (p) => new Uint8Array(await (await fetch(p)).arrayBuffer()),
  writeFile: async (p, b) => { saved.push([p, b]); }
} });
// `saved` holds [path, Uint8Array] pairs — write them out or trigger downloads.
```

Rebrand checklist: colours live in the `CO` object, copy lives in the `promos` array near the
bottom of the script, and the phone-screen scenes are the `sceneJob` / `sceneQuote` /
`sceneMarkup` / `sceneOffline` / `sceneSignature` functions.

## Tests

```bash
./gradlew testDebugUnitTest
```

Covers the money maths (`Money.parseToCents`/`formatPlain` round trips), job and quote totals
including discounts and tax, checklist progress, date helpers and byte formatting.

## Licence / attribution

Your project, your licence. Third-party components are Apache-2.0 (AndroidX, Compose, CameraX,
WorkManager, Coil, Kotlin, kotlinx.serialization, AndroidX Credential Manager).
