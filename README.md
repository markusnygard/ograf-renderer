<!--
SPDX-FileCopyrightText: 2026 Yle
SPDX-License-Identifier: CC-BY-4.0
-->

# OGraf renderer

> **This is a server, not a desktop app or an editing plug-in.** It runs in Docker or on a Linux server, and other systems send it render jobs over HTTP. It was made for the Open Live project. Looking for OGraf graphics in Avid Media Composer, Adobe Premiere Pro or DaVinci Resolve? Those are Yle's OGraf NLE tools, a separate project with its own installers (not public yet). They don't use this renderer.

A headless renderer for [EBU OGraf](https://ograf.ebu.io) graphics. Give it a graphic, its data, a format and a length, and you get a frame-accurate, transparent PNG sequence. It can also make a ProRes 4444 or VP9 WebM file with alpha. Run it as a command-line tool, or as an HTTP service in a container.

It also has a **web editor** for applications: an "Import OGraf" command opens it, the user picks a graphic, fills it in and previews it, and the render runs in the background. A signed webhook tells the application when it is done ([docs/editor.md](docs/editor.md)).

It is the cloud edition of the renderer in Yle's OGraf NLE tools (the Electron app behind the Avid Media Composer and Adobe Premiere Pro plug-ins). Both use the same OGraf runtime, timing logic and virtual clock. This one drives headless Chromium instead of Electron windows, so it runs on a Linux server without a display.

## What it renders

- **Non-realtime graphics** (`"supportsNonRealTime": true`) seek themselves with `goToTime()`, frame by frame.
- **Realtime-only graphics** (SPX templates, most live graphics) run on a **virtual clock**: `Date`, timers, `requestAnimationFrame`, `performance.now()` and CSS/Web animations are driven by the renderer, so the result is frame-accurate and the same on every run.
- **Self-timed graphics** (Lottie exports that play their own in and out) are detected. Their in-animation is held until the out marker.
- **Schedule:** the out-animation ends on the last frame (`"auto"`), or starts at an out marker. Multi-step graphics take "continue" markers. A list of actions can also be passed as is.

| Format | Size | Notes |
|---|---|---|
| `1080p25`, `1080p50` | 1920×1080 | |
| `1080i50` | 1920×1080 | 25 woven frames/s (upper field first), plus the 50 fields/s as `fields/` |
| `2160p25`, `2160p50` | 3840×2160 | 1920×1080 layout at twice the pixel density |
| `1080x1920p25/p50`, `1080x1080p25/p50` | vertical, square | only for graphics whose manifest declares the size in `renderRequirements` |

Output, per render:

```
frames/000000.png …   timeline frames, straight (unpremultiplied) alpha, sRGB
fields/000000.png …   interlaced formats only: the temporal samples
graphic.mov           with encode "prores": ProRes 4444 + alpha (interlaced stays interlaced, TFF)
graphic.webm          with encode "webm": VP9 + alpha (interlaced formats as 50p from the fields)
render.json           what was rendered: graphic, data, format, schedule, timing, frame counts
```

## Run it with Docker

Ready-made images for amd64 and arm64 are on GitHub Container Registry: `ghcr.io/markusnygard/ograf-renderer`. Tags: `latest` and `X.Y.Z` for releases, `main` for the newest commit.

```sh
docker run --rm -p 8080:8080 \
  -e OGRAF_API_TOKEN=change-me \
  -v /path/to/graphics:/data/graphics \
  -v ograf-renders:/data/renders \
  --shm-size=1g \
  ghcr.io/markusnygard/ograf-renderer:latest
```

To build it yourself: `docker build -t ograf-renderer .`. Or run `docker compose up --build`, which serves the sample graphics in `fixtures/graphics`.

```sh
TOKEN=change-me
curl -H "Authorization: Bearer $TOKEN" localhost:8080/api/graphics

curl -H "Authorization: Bearer $TOKEN" -H 'Content-Type: application/json' localhost:8080/api/renders -d '{
  "graphic": "lower-third",
  "format": "1080i50",
  "duration": 6,
  "data": { "name": "Jane Doe", "title": "Reporter" },
  "encodes": ["prores"]
}'
# -> 202 { "id": "01M88…", "state": "queued", … }

curl -H "Authorization: Bearer $TOKEN" localhost:8080/api/renders/01M88…        # state, progress, links
curl -H "Authorization: Bearer $TOKEN" -o graphic.mov localhost:8080/api/renders/01M88…/files/graphic.mov
curl -H "Authorization: Bearer $TOKEN" -o frames.tar localhost:8080/api/renders/01M88…/frames.tar
```

The full API is in [docs/api.md](docs/api.md). To let users make graphics from your application with the web editor, see [docs/editor.md](docs/editor.md).

### Settings (environment variables)

| Variable | Default | |
|---|---|---|
| `PORT` | `8080` | HTTP port |
| `OGRAF_API_TOKEN` | *(none)* | Bearer token for every `/api` call. **Set it.** Without it the API is open to anyone who can reach the port. |
| `OGRAF_GRAPHICS_DIR` | `/data/graphics` | The graphics library: one folder per graphic, sub-folders allowed. Mount it, fill it by upload, or both. |
| `OGRAF_OUTPUT_DIR` | `/data/renders` | One folder per render |
| `OGRAF_ALLOW_UPLOADS` | `true` | `false` turns off `PUT`/`DELETE /api/graphics` (for a read-only, mounted library) |
| `OGRAF_CONCURRENCY` | `1` | Renders run at the same time. Each one already uses several pages in parallel. |
| `OGRAF_PARALLEL` | CPUs ÷ 2, max 4 | Pages per render |
| `OGRAF_RETENTION_HOURS` | `24` | Finished renders are deleted after this (`0` keeps them) |
| `OGRAF_MAX_FRAMES` | `15000` | Longest render, in frames (10 min at 25 fps) |
| `OGRAF_MAX_UPLOAD_MB` | `200` | Largest graphic zip |
| `OGRAF_NETWORK` | *(blocked)* | `allow` lets graphics load remote resources (web fonts, images). See *Security*. |
| `OGRAF_CORS_ORIGIN` | *(none)* | Allowed browser origin, for a web UI calling the API directly |
| `OGRAF_PUBLIC_URL` | from the request | The renderer's address as browsers see it (`https://ograf.example.com`), for editor links |
| `OGRAF_EDITOR_ORIGINS` | *(none)* | Application origins that may embed the editor, get its messages and be returned to |
| `OGRAF_EDITOR_SESSION_HOURS` | `4` | How long an editor link works |
| `OGRAF_WEBHOOK_ALLOW` | *(none: webhooks off)* | Where webhooks may go: origins, or `*.example.com` (https) |
| `OGRAF_WEBHOOK_SECRET` | *(none)* | Signs webhooks (`X-Ograf-Signature`). Set it. |
| `OGRAF_CHROME` | `/usr/bin/chromium` in the image | Chromium to use |
| `OGRAF_CHROME_SANDBOX` | off in Docker | `1` keeps Chromium's own sandbox (needs a container allowed to create user namespaces) |
| `OGRAF_FFMPEG` | `ffmpeg` | FFmpeg to use for the encodes |
| `OGRAF_DEBUG` | | `1`: graphic console output and timings in the log |

Sizing: rendering uses CPU, not a GPU. One render of a 1080p lower third takes about 50 ms per frame with 4 pages. Give the container 2–4 CPUs and 2–4 GB of memory per concurrent render, and `--shm-size=1g` (or more). UHD needs about four times as much.

## Command line

Needs Node.js 20 or later. Use a local Chrome, or `npm install` (which downloads one for development).

```sh
npm install
node bin/ograf-renderer.js render --graphic fixtures/graphics/lower-third --out out/lt \
  --format 1080p50 --seconds 5 --data '{"name":"Jane Doe","title":"Reporter"}' --encode prores,webm
node bin/ograf-renderer.js probe --graphic fixtures/graphics/self-timed
node bin/ograf-renderer.js serve --graphics fixtures/graphics --output out/renders
node bin/ograf-renderer.js help
```

From your own Node code:

```js
import { renderGraphic, closeBrowser } from 'ograf-renderer'

const info = await renderGraphic({ graphicDir: 'graphics/lower-third', format: '1080p25', duration: 5, data: { name: 'Jane' }, outDir: 'out', encodes: ['webm'] })
await closeBrowser()
```

## Tests

```sh
npm run test:unit   # no browser needed
npm test            # everything, incl. real renders and the editor in a headless browser (~4 min)
```

The tests also run in the image, against its Chromium, FFmpeg and fonts (CI does both):

```sh
docker run --rm -v "$PWD:/src:ro" --tmpfs /tmp --shm-size=1g ograf-renderer sh /src/tools/test-in-image.sh
```

## Security

Graphics are third-party code, and they run in Chromium inside your network. So:

- **No network for graphics by default.** Chromium sends every request except those to the renderer's own local file server to a proxy that doesn't exist. Graphics can't reach cloud metadata endpoints (`169.254.169.254`), internal services or the internet. If your graphics need web fonts or remote images, set `OGRAF_NETWORK=allow`, and add an egress policy in your platform for the private ranges.
- Each render gets its own browser context, and a graphic is only served its own folder. Links and junctions out of it are refused.
- Uploaded zips are checked for path traversal and zip bombs. Graphic names are limited to `[A-Za-z0-9._-]`.
- The container runs as the unprivileged `node` user. Chromium's own sandbox is off in Docker, because Docker's default seccomp profile doesn't allow it. The container is the sandbox: don't run it privileged, and keep it on its own network.
- Put the API behind TLS (ingress, load balancer) and always set `OGRAF_API_TOKEN`.
- **The editor** never gets the API token. Its link carries a one-time session key in the URL fragment, which only allows that session's preview and one render. In the editor, graphics run in sandboxed frames with an opaque origin, so they can't read the page, its key or the API. The page has a strict Content-Security-Policy, and only `OGRAF_EDITOR_ORIGINS` may embed it.
- **Webhooks** go only to `OGRAF_WEBHOOK_ALLOW`, don't follow redirects, and are signed with `OGRAF_WEBHOOK_SECRET`.

## Licence

Code is Apache-2.0 ([LICENSE](LICENSE)), documentation CC-BY-4.0 ([LICENSES/CC-BY-4.0.txt](LICENSES/CC-BY-4.0.txt)). Every file carries an SPDX header (checked by the tests). Attributions are in [NOTICE](NOTICE). [THIRD-PARTY.md](THIRD-PARTY.md) lists the third-party components, including the Debian packages in the Docker image.

The OGraf runtime in `runtime/render.js` is adapted from [SuperFlyTV/ograf-devtool](https://github.com/SuperFlyTV/ograf-devtool) (MIT).
