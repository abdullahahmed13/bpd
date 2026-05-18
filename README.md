# gcore-videoplayer-js XSS — GitHub Pages bundle

Seven static files. Drop them at the root of a public GitHub repo, enable Pages, and you have a public URL to paste into the Gcore demo's `/source` field.

| File | Purpose |
|---|---|
| `master.mpd` | **DASH manifest** — `<Label>` in audio + subtitle `<AdaptationSet>` carries `alert(document.domain)`. Use this for the demo's MPD input. |
| `master.m3u8` | **HLS manifest** — `#EXT-X-MEDIA NAME=` carries the same payload. Use if the demo also accepts m3u8. |
| `video.m3u8` `audio.m3u8` `subs.m3u8` | Stub HLS variant playlists referenced by `master.m3u8`. |
| `subs.vtt` | Malicious WebVTT — cue text contains the payload too (finding 2.2). Used by both manifests. |
| `README.md` | This file. |

All five payload files use the canonical safe XSS probe `<img src=x onerror=alert(document.domain)>`. Three sinks fire:
- audio dropdown label (`<%= track.label %>` in `audio-tracks/template.ejs`)
- CC dropdown label (`<%= t.name %>` in `cc/combobox.ejs`)
- live caption line (`ClosedCaptions.setSubtitleText` → `.html(getCueAsHTML().textContent)`)

## Deploy to GitHub Pages

```bash
cd gcore-xss-gh-pages

# create an empty repo on github first, then:
git init
git add .
git commit -m "PoC for gcore-videoplayer-js manifest XSS (CVE pending)"
git branch -M main
git remote add origin git@github.com:<you>/<repo>.git
git push -u origin main
```

Now enable Pages:
1. GitHub → repo → **Settings → Pages**
2. **Build and deployment** → Source = **Deploy from a branch**
3. Branch = `main`, folder = `/` (root) → **Save**
4. Wait ~30 s. The URL appears at the top: `https://<you>.github.io/<repo>/`

Verify the files are reachable (CORS headers are sent automatically by Pages):

```bash
curl -sI https://<you>.github.io/<repo>/master.mpd | head -3
# expect HTTP/2 200 and content-type that starts with application/dash+xml or application/octet-stream — either works
```

## Fire it at the Gcore demo

1. Open `https://gcore-videoplayer-js-nuxt.vercel.app/source`.
2. Into the source/URL field, paste:
   ```
   https://<you>.github.io/<repo>/master.mpd
   ```
3. Click whatever Load/Apply/Play button the demo offers.
4. Watch for the modal showing `gcore-videoplayer-js-nuxt.vercel.app` — that string in the dialog is the killshot for the bounty report. Screenshot it.

If the demo also accepts HLS, repeat with `master.m3u8` for a second screenshot from a different code path (HLS parser, same EJS sinks). Two-format coverage strengthens the report.

## Optional — convert to a clickable one-shot link

If the demo accepts a `?source=` / `?src=` / `?url=` query parameter (common for Nuxt demos — inspect the form's submit handler or the page URL after you click load), you can collapse the repro to a single link:

```
https://gcore-videoplayer-js-nuxt.vercel.app/source?source=https%3A%2F%2F<you>.github.io%2F<repo>%2Fmaster.mpd
```

That URL in a bounty report is gold — triager clicks once, sees the alert, accepts the report.

## What if nothing fires

Check Network tab for the manifest fetch:

- **CORS failure** (`net::ERR_BLOCKED_BY_CORS` or red entry in Network) — shouldn't happen with GitHub Pages, but if it does, switch to `raw.githubusercontent.com` URLs which also send `Access-Control-Allow-Origin: *`.
- **CSP violation in console** — demo origin has a CSP that blocks inline event handlers. Open Elements → Ctrl+F → `XSS` or `alert` — the injected `<img>` is still in the DOM. That alone is the bug; the CSP merely mitigates the consequences. Note it in the report.
- **Demo iframes the player with `sandbox="allow-scripts"` and no `allow-same-origin`** — XSS fires in a null origin and is less impactful. Still demonstrable, but mention it; severity drops.

## Cleanup after the report is filed

Delete the repo or make it private. The PoC files are harmless on their own (no real exfil, just a `document.domain` alert) but it's good hygiene not to leave reachable attack hosts up.

## Bug bounty submission

`bugbounty@gcore.com` per the repo's `SECURITY.md`.

Subject: **`Stored XSS in @gcorevideo/player via unescaped HLS/DASH manifest metadata`**

Include:
- Clickable one-shot URL (or paste-step instructions if no query param)
- Screenshot of the alert showing `gcore-videoplayer-js-nuxt.vercel.app`
- Code-level root cause: `<%= %>` (unescaped) in `packages/player/assets/audio-tracks/template.ejs`, `cc/combobox.ejs`; `.html(text)` in `ClosedCaptions.setSubtitleText`
- Affected: `@gcorevideo/player` ≤ 2.30.3 (published bundle at `player.gvideo.co/v2/assets/latest/`)
- Fix: every `<%=` → `<%-`; replace `.html(text)` with `.text(text)` in `ClosedCaptions.setSubtitleText`
- CVSS 3.1: `AV:N/AC:L/PR:N/UI:R/S:C/C:H/I:L/A:N` ≈ 8.3 High
