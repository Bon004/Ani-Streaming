# ani-cli Fork — Claude Instructions

## Overview

POSIX sh script (v4.14.0). Streams anime via AllAnime API, plays with mpv.exe. Windows-native (MSYS2/Git Bash).

## Sync After Every Change

```sh
cp ~/ani-cli/ani /c/Users/esteb/scoop/apps/ani-cli/current/ani
```

Both files must be kept identical. The live path is what actually runs.

## Fork Additions (not in upstream)

**1. AniList thumbnail preview in fzf** (`launcher()`)
- Preview script at `$_ani_preview_script`, generated into `$anilist_cache_dir/`
- Uses chafa `--symbols=all` (symbols mode only — sixels/kitty produce too much output for fzf)
- WPF preview window spawned via powershell.exe with stdin AND stdout redirected to /dev/null (otherwise hangs the fzf pipeline)
- Sentinel file `$anilist_cache_dir/current_preview` controls WPF window lifetime
- Disable with `ANI_CLI_THUMBNAILS=0`

**2. AllAnime CDN fix** (`decode_tobeparsed()`)
- AllAnime's clock.json CDN is broken; fast4speed signed URLs (`stype:"t"`) still work
- Extracts `https://tools.fast4speed.rsvp/...` directly from tobeparsed JSON
- Written to `$cache_dir/yt`, consumed by `get_episode_url()` which opens it directly

**2b. mkissa migration + aaReq token** (`get_crypto()` / `get_aa_req()`, migrated 2026-07-19)
- AllAnime's backend moved to **mkissa** (site drama / host takeover). Live hosts: token'd episode + search API `allanime_api="https://api.mkissa.net"`, referrer/Origin `https://mkissa.to`. BUT the internal clock endpoint (Default/wixmp provider) still lives at `allanime.day/apivtwo/clock.json` — only allanime.day's *root/graphql* is Cloudflare-walled, the clock path returns 200. So `allanime_base="allanime.day"` (clock) and `allanime_api` is set explicitly to mkissa, NOT derived from base
- Episode queries require an encrypted `aaReq` extension, else `AA_CRYPTO_MISSING`; wrong epoch/build → `AA_CRYPTO_STALE`
- Token = base64(0x01 || iv || AES-256-GCM(payload) || tag); iv = first 12 bytes of sha256("epoch:buildId:queryHash:ts"); ts = unix time floored to 300s, in ms. `buildId` is currently **49**. Encryption done with `node -e` (upstream uses botan — not on Windows/scoop)
- **Split key** (`get_crypto()`): `key = partB XOR mask`.
  - `partB` + `epoch` rotate ~every 3 days, served in mkissa.to SSR HTML as `window.__aaCrypto={...}`
  - `mask` is a bare 64-hex literal baked into the JS chunk that also references `__aaCrypto` (changes on site redeploy). Found by walking `app.js`'s `../chunks/*.js` imports — NOT in the homepage modulepreload list, so stl3's `bd="..."` regex misses it
  - Derived key cached at `$aa_crypto_cache` (`~/.cache/ani-cli/aa_crypto`) for `$aa_crypto_ttl` (21600s / 6h). On any `AA_CRYPTO` error, `get_episode_url` busts the cache and retries once
  - Env pins still win: `ANI_CLI_AA_KEY` + `ANI_CLI_AA_EPOCH` (both) skip the fetch; `ANI_CLI_AA_BUILD` overrides buildId
- Same derived key decrypts `tobeparsed` responses (`process_response`/`decode_tobeparsed`)

**2c. Provider layer** (mkissa serves a different provider mix than old AllAnime)
- `decode_tobeparsed` and `get_episode_url` capture the FULL `sourceUrl` (both `--`-internal and absolute). Capturing only `--` (the old fork bug) dropped every absolute provider
- The old `--`-internal providers (Luf-Mp4, Ak, Uv-mp4…) are mostly dead — their clock host is gone. **Default (wixmp)** still works via the allanime.day clock but is absent from most current shows
- **ok.ru (`Ok`) is the widest-coverage provider** (~all popular shows offer it) — resolved by handing the `https://ok.ru/videoembed/...` embed to **yt-dlp** (`get_links` `*ok.ru*` branch, provider slot 6). yt-dlp does NOT support filemoon/streamwish/streamlare/streamsb/vidnest, so ok.ru is the only viable yt-dlp host
- Working provider chain: wixmp (slot 1, rare) → mp4upload (slot 4, often dead/deleted) → ok.ru (slot 6, primary). Coverage ~75%; remaining failures are per-video copyright takedowns of the ok.ru source (nothing code can fix)
- Reference: stl3's `ani-cli-rpi` `cdntesting` branch pioneered the mkissa crypto (but only wixmp+mp4upload providers — it also fails on modern shows). Upstream pystardust hasn't merged a fix. When it breaks again, re-check the buildId/mask first, then provider availability

**3. Manga support** (`manga_main()` and helpers, ~line 420)
- `--manga` flag, isolated behind `[ "$_manga_mode" = "1" ] && manga_main && exit 0`
- Source: MangaDex public REST API (no auth)
- Viewer: chafa terminal rendering, one page at a time; `s` key for double-page spread
- History: `$hist_dir/manga-hsts` (format: `chapter_id\tmanga_id\ttitle\tchapter_display`)
- External chapters (MangaPlus etc.) tagged `[ext]` and opened in browser
- AniList sync via `anilist_sync_manga_progress()` — progress synced as integer chapter number

## Key Functions

| Function | Location | Purpose |
|----------|----------|---------|
| `launcher()` | ~line 12 | fzf/rofi wrapper; thumbnail preview wired here |
| `decode_tobeparsed()` | ~line 300 | Extracts video URLs from AllAnime JSON; fast4speed path |
| `get_episode_url()` | ~line 330 | Assembles final stream URL; reads `$cache_dir/yt` |
| `manga_search()` | ~line 420 | MangaDex title search |
| `manga_chapters_list()` | ~line 435 | Paginated chapter feed (handles 1000+ chapter series) |
| `manga_read_chapter()` | ~line 465 | Page viewer with spread mode |
| `manga_main()` | ~line 580 | Manga flow orchestrator |
| `anilist_query()` | ~line 600 | Shared GraphQL wrapper for all AniList calls |
| `nth()` | ~line 50 | fzf selection wrapper; accepts `linenum\tid\tdisplay` TSV |

## What NOT to Change

- `nth()` TSV format — it's `linenum\tid\tdisplay`; manga uses same convention
- `anilist_query()` — shared by anime and manga sync; don't duplicate it
- `get_links` / `play_episode` / `download` — anime playback chain; manga is fully isolated from these
- The `powershell.exe ... < /dev/null > /dev/null 2>&1 &` redirect pattern in `launcher()` — critical for pipeline integrity

## Dependencies

| Dep | Required by | Install |
|-----|-------------|---------|
| curl, sed, grep, openssl | core | system |
| node | aaReq token (AES-GCM) | nodejs.org / scoop |
| yt-dlp | ok.ru provider (primary playback path) | `scoop install yt-dlp` |
| fzf | selection UI | `scoop install fzf` |
| mpv | playback | `scoop install mpv` |
| jq | AniList sync + manga | `scoop install jq` |
| chafa | thumbnails + manga viewer | `scoop install chafa` |
| aria2c + ffmpeg | `--download` | `scoop install aria2 ffmpeg` |

## AniList Token Security

Token stored at `$anilist_token_file` (chmod 600). Never log, display, or store the raw token string. When guiding setup, instruct user to copy only the token value — not `&token_type=Bearer&expires_in=31536000`.

## Testing

```sh
bash -n ~/ani-cli/ani              # syntax check
ani "classroom of the elite"       # anime smoke test
ani --manga "berserk"              # manga smoke test
ani --continue                     # history smoke test
```
