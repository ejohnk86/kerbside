# Kerbside

A personal home-screen web app for Singapore ride-hailing. It does two things:
launches the five ride apps (Grab, Gojek, TADA, Ryde, CDG Zig) with the destination
copied to the clipboard, and logs the fares the user actually saw so it can learn which
app to open first. Built for one user (John), hosted on GitHub Pages, added to the
home screen from the mobile browser.

The launcher is platform-aware. On Android it fires `intent://` URLs keyed by package
name; on iOS it opens a custom URL scheme with an App Store link as timed fallback
(`visibilitychange` cancels the fallback when the app actually opened). Detection is
the `IOS` flag at the top of the script. John's Android phone is in repair
(may not return); the current device is an iPhone 11 Pro Max on iOS 26, so the iOS
path is the one that matters right now.

## Hard constraints — do not violate

- **Single self-contained file.** Everything lives in `index.html`. No build step, no
  bundler, no npm, no frameworks, no external JS dependencies. (Google Fonts is the one
  permitted external resource.)
- **Never estimates fares.** The app only records and ranks fares the user actually saw.
  No fare models, no distance-based formulas, no surge guessing. This is a design
  decision, not an omission: an estimate can be wrong; an observation can't.
- **No scraping of operator endpoints.** No calls to Grab/Gojek/TADA/Ryde/Zig APIs,
  no impersonating their clients. The launcher opens apps via Android `intent://` URLs
  with a Play Store fallback; that's the whole integration surface. The one permitted
  network API is OneMap (SG government geocoder, no token) — used only when the user
  pins a saved place, never per-ride.
- **Works offline once cached; data never leaves the phone.** localStorage only
  (`ks.rides`, `ks.favs`, `ks.pkgs`, `ks.ios`), with an in-memory fallback when
  storage is blocked. Export/import is manual JSON.
- **Ranking honesty.** Only rides with ≥2 logged fares count as comparisons. Fallback
  tiers: this route + time band → this route any time → all routes same day-type +
  band → all rides; a tier is used only when it has ≥3 comparisons, and the UI always
  states which sample ("basis") it used.

## Design tokens — keep a restyle from drifting

Colours (CSS custom properties in `:root`):

| token | value | role |
|---|---|---|
| `--asphalt` | `#14171A` | page background |
| `--kerb` | `#1E2328` | card background |
| `--kerb-hi` | `#272D34` | raised/selected card |
| `--meter` | `#FFB020` | accent, active tab, hero readout |
| `--fare` | `#7FD1AE` | win/cheapest highlight |
| `--paper` | `#E8E6DF` | receipt card background |
| `--ink` | `#101316` | text on paper |
| `--dim` | `#7A858F` | secondary text |
| `--line` | `#2C333A` | borders |
| `--warn` | `#E8825A` | unlinked/warning |

Type: `--mono` = "Martian Mono" (labels, numbers, all-caps microcopy),
`--sans` = "Instrument Sans" (body). Max content width 520px, mobile-first.

## Current state / open questions (needs a real device)

- iOS (current device): `grab://` is **confirmed working** on the iPhone (tested
  2026-08-12). `gojek://` is untested (app not installed on the device). TADA, Ryde
  and CDG Zig have no known scheme; they launch via their App Store links, which are
  prefilled and verified for all five apps (SG store ids: Grab 647268330,
  Gojek 944875099, TADA 1412329684, Ryde 979806982, CDG Zig 954951647). Prefill
  defaults merge into `ks.ios` only for fields the user has never touched.
- Android (if the phone returns): only Grab (`com.grabtaxi.passenger`) is confirmed.
  Gojek is prefilled as `com.gojek.app` but unverified. TADA, Ryde, CDG Zig are blank —
  the Setup tab extracts the package from a pasted Play Store share link.
- Untested: whether `intent://` URLs fire correctly from an installed PWA, and whether
  any of the five apps accept route parameters in a deep link. Each iOS app row has a
  "deep link template" field (`{dest}` = URL-encoded destination, `{lat}`/`{lng}` =
  coordinates of a pinned saved place); when set and a destination is typed, it is
  tried instead of the plain scheme. Templates containing `{lat}` only fire when the
  destination matches a pinned place; otherwise plain scheme. Saved places can carry
  coordinates (OneMap geocode at add-time; legacy string entries still work unpinned).
  **Deep-link prefill is a confirmed dead end for Grab** (two device tests, Aug 2026):
  `screenType=BOOKING` opens the Transport tab but `dropOffAddress` text is ignored,
  and the `dropOffLatitude/Longitude` variant fails to open Grab at all (bounced to the
  App Store fallback). No prefilled Grab template ships now; a startup migration
  (`RETIRED_DL`) strips the old auto-prefilled ones from stored `ks.ios`. The template
  field remains for any future user-supplied link. Do not re-add a guessed Grab deep
  link without a fresh working example from the device.
- The iOS App Store fallback uses an elapsed-time guard (iOS suspends page timers when
  an app opens, so a late-firing callback = app opened = no redirect) plus
  `pagehide`/`visibilitychange` cancels. This replaced a naive 1.6s timer that wrongly
  bounced slow app-opens to the App Store. Keep the guard if you touch `launch()`.
- John does not use the fare log (ranking known by intuition: TADA, Grab, CDG Zig) —
  launcher friction is what matters.
- `navigator.clipboard` requires a secure context. Testing over plain-http LAN
  (`python -m http.server`) silently breaks the copy-destination feature — test through
  GitHub Pages or a cloudflared https tunnel.

## Workflow

Edit `index.html` → push to `main` → GitHub Pages serves it → pull-to-refresh in Chrome
on the phone. No local dev server needed; when one is used anyway, remember the https
caveat above.
