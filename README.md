# TubeClean for Chrome: ad-free YouTube, Tubi and Pluto TV

TubeClean is a Chrome extension (Manifest V3) for laptops and desktops. It removes ads on:
- **YouTube** (www.youtube.com), and keeps it guest-only, with no history.
- **Tubi** (tubitv.com).
- **Pluto TV** (pluto.tv), on-demand titles.

It also works in other Chromium browsers such as Microsoft Edge. It doesn't work in the DuckDuckGo browser, which doesn't support extensions.

It makes no network requests of its own and uses no remote servers.

Sites can't see it or fake its counts. Its page scripts are built into one bundle per site (`page-youtube.js`, `page-tubi.js`, `page-pluto.js`), with the rules and the bridge kept inside the bundle rather than on `window`. The bridge talks to the extension over a private link: an element never put in the page, handed to `src/relay.js` before any of the site's own scripts run. The only thing it stores is daily ad-block counts.

## Install

1. Get the files:
   - **From a release:** download the zip and unzip it somewhere permanent. The extension stops working if the folder is deleted.
   - **From source:** run `node tools/build-extension.mjs` (Node 22 or later). It writes `dist/` and `build/TubeClean-chrome-<version>.zip`.
2. Open `chrome://extensions` (in Edge: `edge://extensions`).
3. Turn on **Developer mode**.
4. Click **Load unpacked** and choose the unzipped folder or `dist/`.
5. Optionally, pin TubeClean from the puzzle-piece menu.

To update, rebuild or unzip the new version over the old folder, then click the reload ↻ icon on TubeClean's card.

Chrome may warn at startup about developer-mode extensions. Close the warning; don't click Disable.

## YouTube

| Layer | How |
|---|---|
| 1. Network blocking | `declarativeNetRequest` rules built from `adUrls` and `trackerUrls`. They only apply to requests made by YouTube pages; video streams (`neverBlock`) are never blocked. |
| 2. Cosmetic hiding | `adskip.js` hides ad tiles and banners (`hideSelectors`). |
| 3. Ad skipper | If an ad still plays, `adskip.js` mutes it, jumps to its end at 16× speed and clicks Skip. |
| 4. Ad-data stripping | `prune.js` deletes `adPlacements`, `playerAds` and similar keys (`adDataKeys`) from the video data before the player reads it, so ads are never scheduled. |

`src/enforcement.js` also removes YouTube's "Ad blockers are not allowed" dialog and resumes the video.

**Guest only:**
- **Sign-in:** YouTube's sign-in pages and endpoints are refused, but only when YouTube requests them. Gmail and other Google sign-ins in Chrome keep working.
- **Wipe:** YouTube's storage and cache are wiped when its last tab closes or leaves YouTube, and when Chrome starts. If you were signed in to YouTube in Chrome, installing signs you out of YouTube only.
- **Kept guest:** YouTube asks a brand-new guest to "sign in to confirm you're not a bot" before some videos, so YouTube's cookies (the guest ID) are kept between sessions.
  - A new guest starts every 10 days, or with **Clean Start** in the popup. Clean Start reloads open YouTube tabs.
  - The popup shows the guest's day ("day 3 of 10").
  - Google sign-in cookies (`SID`, `LOGIN_INFO`, ...) are removed from YouTube the moment they appear, so the kept guest never becomes a signed-in session.
  - If YouTube asks anyway, a note on the page explains it: watch any other video for a minute, then come back.
- **History:** YouTube pages are deleted from Chrome's history as soon as they're added.

## Tubi

Tubi doesn't stitch ads into the movie. Before each ad break, its player asks Tubi's ad service (Rainmaker) for a list of ads, then plays them around the movie.
- **How it's blocked:** `src/tubi.js` empties that list as the player reads it, so the player has nothing to play and carries on with the movie.
- **Ad-blocker check:** Tubi's check (`mcdn.tubitv.com/js/ads.js`) is left alone, so the site still sees "ads allowed".
- **Ad networks:** requests from Tubi pages to known ad networks and Tubi's ad servers (`blockDomains` in `tubi.json`) are refused.
- **Guest mode:** Tubi isn't wiped and sign-in isn't blocked; those are YouTube-only.

## Pluto TV

Pluto stitches its ads into the video. A title's playlist (a DASH `.mpd` file) lists the movie in pieces, with ad pieces in between. A separate schedule (`session.json`) drives the player's ad countdown and locks its controls during a break.
- **How it's blocked:** `src/pluto.js` removes the ad pieces from both, and re-times the movie's own pieces to follow each other, so the movie plays straight through. Tested on *The Ring*: 46–51 ads, about 22 minutes, were removed from a 2 h 17 min stream.
- **Live channels** are left alone.
- **Ad networks:** requests from Pluto pages to known ad networks and its pause-screen ad images (`blockDomains` in `pluto.json`) are refused.
- **Guest mode:** none; Pluto isn't wiped.

This replaces the separate "Tubi, Pluto & YouTube Ad Blocker" extension. Don't run both: that one blocks YouTube's bot check (`api/stats/atr`), which makes YouTube ask you to sign in.

## Popup and badge

The toolbar badge shows the ads blocked in the current video, or **!** when the rules need revising. The popup shows:
- ads blocked in this video
- today's totals, average per video and requests refused
- a 30-day chart

Counts are the only data kept: per day, the number of videos, ads blocked per video, and requests refused. No video IDs, titles or searches are stored.

## Revising the rules

Sites change. When something stops matching, the popup shows **Blocker needs revision** and names the key to fix:

| Code | Meaning | Fix |
|---|---|---|
| `PLAYER_NOT_FOUND` | A YouTube video plays but no `playerSelectors` match | `shared/rules.json` → `desktop` |
| `SKIP_FAILED` | A YouTube ad is still on screen 8 s after the skip attempt | `adClassNames`, `skipButtonSelectors` |
| `AD_DATA_CHANGED` | A YouTube ad played despite stripping | `adDataKeys`, `adUrls` |
| `AD_UNDETECTED` | New ad-like fields in YouTube's video data | `adDataKeys` or `ignoreAdLikeKeys` |
| `SITE_CHANGED` | YouTube showed its ad-blocker warning | `enforcementSelectors`, `hideSelectors` |
| `TUBI_CHANGED` | Tubi's ad-service answer changed, or a title played without one | `tubi.json` |
| `PLUTO_CHANGED` | Pluto's playlist or schedule format changed | `pluto.json` |

To revise:
1. Inspect the site with DevTools (Elements for selectors, Network for requests).
2. Edit the file named above. Desktop-only YouTube selectors go in the `desktop` section of `shared/rules.json`.
3. Bump that file's `"version"`; this also clears the warnings.
4. Run `node tools/build-extension.mjs` and reload the extension.

`shared/rules.json`, `shared/prune.js` and `shared/adskip.js` are shared with the TubeClean Android app (Fire TV and phones). Keep them in step with the app's copies.

## Tests

```
node --test tools/*.test.mjs
```

They check:
- the generated network rules against the real `shared/rules.json`: what gets blocked, that video streams never are, and that query strings can't cause a block
- the desktop profile
- the stats and health monitor
- the Tubi blocker, fed ad lists in Tubi's own format
- the Pluto blocker, fed playlists and schedules shaped like Pluto's
- the page bundles: no TubeClean names on `window`, and the bridge only offers its private link

## Layout

```
manifest.json            extension manifest; the build fills in site matches and permissions
tubi.json                Tubi's ad-service patterns and ad domains
pluto.json               Pluto's playlist patterns and ad domains
shared/rules.json        YouTube rules (shared with the app; "desktop" section for this extension)
shared/prune.js          YouTube layer 4: strips ad data at document start
shared/adskip.js         YouTube layers 2 + 3, health probes, per-video counting
src/background.js        service worker: stats, health, badge, YouTube guest-only wipe
src/bridge-shim.js       stands in for the app's TubeCleanBridge in the page
src/relay.js             passes bridge calls from the page to the service worker over a private link
src/enforcement.js       removes YouTube's anti-adblock dialog
src/tubi.js              Tubi: empties the ad list before the player reads it
src/pluto.js             Pluto TV: removes ad pieces from on-demand playlists and the player schedule
src/stats.js, health.js  30-day ad counts and revision detection
src/guest.js             kept guest: 10-day renewal rule, sign-in cookie list
popup.*                  toolbar popup
tools/build-extension.mjs  builds dist/ and the zip
tools/extension-lib.mjs    desktop profile merge, rules.json -> declarativeNetRequest rules
```

## License

Copyright (C) 2026 srispace1645

This program is free software: you can redistribute it and/or modify it under the terms of the GNU General Public License as published by the Free Software Foundation, either version 3 of the License, or (at your option) any later version.

This program is distributed in the hope that it will be useful, but WITHOUT ANY WARRANTY; without even the implied warranty of MERCHANTABILITY or FITNESS FOR A PARTICULAR PURPOSE. See the [GNU General Public License](LICENSE) for more details.
