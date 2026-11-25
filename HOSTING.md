# Hosting cirro-cloud.com

A static landing page: one HTML file, a poster image, and a silent looping background video. Ready to deploy free with no ads on **GitHub Pages** (or any static host such as Cloudflare Pages, Netlify, Vercel, or S3 + CloudFront).

## Deploying on GitHub Pages (free, no ads)

This folder already includes:
- `CNAME` containing `cirro-cloud.com`
- `.nojekyll` so GitHub Pages serves static files directly without Jekyll processing

Steps for GitHub account `RobWalkersGit`:

1. Create a public repository named `cirro-cloud` at `https://github.com/new`.
2. Push the contents of this folder to the `main` branch root:
   - `index.html`, `hero.mp4`, `hero.webm`, `hero-mobile.mp4`, `hero-mobile.webm`, `poster.jpg`, `CNAME`, `.nojekyll`, `HOSTING.md`
3. In the repository go to **Settings → Pages**:
   - **Source**: Deploy from a branch
   - **Branch**: `main`, folder `/ (root)` → **Save**
   - Your default live URL will be `https://robwalkersgit.github.io/cirro-cloud/` until the custom domain DNS resolves, at which point `https://cirro-cloud.com` serves the root.
4. Confirm **Custom domain** shows `cirro-cloud.com` (populated from `CNAME`).
5. Once DNS resolves (see below) and GitHub issues the TLS certificate, check **Enforce HTTPS**.

### GitHub Pages notes & limits
- GitHub Pages serves over Fastly CDN with proper `video/mp4`, `video/webm`, `image/jpeg` MIME types and HTTP `206 Partial Content` (`Accept-Ranges: bytes`) support for video seeking/streaming.
- Total site size here is ~6.8 MB (well below GitHub Pages' 1 GB site limit and GitHub's 100 MB per-file limit).
- Bandwidth soft limit is 100 GB/month; if traffic outgrows that, move the video files to Cloudflare R2 (zero egress) or Cloudflare Pages (unlimited static bandwidth).

## DNS records for `cirro-cloud.com`

At your domain registrar, configure these DNS records (remove any conflicting parking A/CNAME records first):

### Apex domain (`cirro-cloud.com` / `@`)

| Type | Host / Name | Value |
| --- | --- | --- |
| `A` | `@` | `185.199.108.153` |
| `A` | `@` | `185.199.109.153` |
| `A` | `@` | `185.199.110.153` |
| `A` | `@` | `185.199.111.153` |
| `AAAA` | `@` | `2606:50c0:8000::153` |
| `AAAA` | `@` | `2606:50c0:8001::153` |
| `AAAA` | `@` | `2606:50c0:8002::153` |
| `AAAA` | `@` | `2606:50c0:8003::153` |

### `www` subdomain

| Type | Host / Name | Value |
| --- | --- | --- |
| `CNAME` | `www` | `robwalkersgit.github.io` |

GitHub Pages automatically redirects `www.cirro-cloud.com` to `cirro-cloud.com` when `CNAME` contains `cirro-cloud.com`.

After DNS propagates (usually 5–30 minutes, check with `dig cirro-cloud.com +short`), return to **Settings → Pages** and enable **Enforce HTTPS**.

## Self-hosting / other CDNs (if not using GitHub Pages)

### MIME types
| File | Content-Type |
| --- | --- |
| `index.html` | `text/html; charset=utf-8` |
| `hero.webm`, `hero-mobile.webm` | `video/webm` |
| `hero.mp4`, `hero-mobile.mp4` | `video/mp4` |
| `poster.jpg` | `image/jpeg` |

### Byte-range requests & faststart
Origins/CDNs must answer `Range` requests with `206 Partial Content` and `Accept-Ranges: bytes`. Both MP4 files are encoded with `-movflags +faststart` (`moov` atom at the start) so playback begins immediately.

### Cache headers
```
index.html                              Cache-Control: public, max-age=300, must-revalidate
hero*.mp4, hero*.webm, poster.jpg       Cache-Control: public, max-age=31536000, immutable
```
If you replace a clip, rename the file (e.g. `hero-v3.mp4`) and update `<source>` in `index.html` or purge the CDN.

## Video specs & iOS autoplay behavior

Current encodes (14.5 s, silent, 24 fps):
- `hero.mp4` — 1920×1080, H.264, ~3.3 MB
- `hero.webm` — 1920×1080, VP9, ~2.1 MB
- `hero-mobile.mp4` — 960×540, ~0.7 MB
- `hero-mobile.webm` — 960×540, ~0.6 MB
- `poster.jpg` — first frame of the loop, ~161 KB

The `<video>` element sets `autoplay muted loop playsinline`, which Safari requires for silent autoplay. Under iOS Low Power Mode or `prefers-reduced-motion: reduce`, playback is paused/blocked and `poster.jpg` remains visible as a still background (matching `background-color: #0a427e`).

## Swapping in a different clip

```bash
# MP4 (H.264, yuv420p, faststart, CRF 23–26, under ~8 MB)
ffmpeg -i loop.mp4 -c:v libx264 -profile:v high -pix_fmt yuv420p \
  -crf 24 -preset slow -movflags +faststart -an hero.mp4

# WebM (VP9)
ffmpeg -i loop.mp4 -c:v libvpx-vp9 -b:v 0 -crf 33 -pix_fmt yuv420p \
  -row-mt 1 -deadline good -an hero.webm

# Mobile variant (960×540)
ffmpeg -i loop.mp4 -vf scale=960:540 -c:v libx264 -profile:v high \
  -pix_fmt yuv420p -crf 25 -preset slow -movflags +faststart -an hero-mobile.mp4

# Poster frame
ffmpeg -i loop.mp4 -frames:v 1 -q:v 3 poster.jpg
```
