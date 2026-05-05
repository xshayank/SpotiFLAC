# SpotiFLAC — Qobuz & Amazon Downloader Backend

This document covers how the Qobuz and Amazon Music download backends work inside SpotiFLAC, including every API call made, all request/response shapes, and a step-by-step guide for integrating the same logic in your own project.

---

## Table of Contents

1. [Overview](#overview)
2. [Shared Infrastructure](#shared-infrastructure)
   - [HTTP Headers](#http-headers)
   - [Spotify → ISRC Resolution](#spotify--isrc-resolution)
   - [Platform Link Resolution (Songstats / song.link)](#platform-link-resolution)
   - [Provider Priority System](#provider-priority-system)
3. [Qobuz Backend](#qobuz-backend)
   - [Credential Bootstrapping](#credential-bootstrapping)
   - [Request Signing](#request-signing)
   - [Qobuz API Reference](#qobuz-api-reference)
   - [Download Providers](#qobuz-download-providers)
   - [Quality Codes](#quality-codes)
   - [Full Download Flow](#qobuz-full-download-flow)
4. [Amazon Music Backend](#amazon-music-backend)
   - [Amazon API Reference](#amazon-api-reference)
   - [Decryption](#decryption)
   - [Full Download Flow](#amazon-full-download-flow)
5. [Setting It Up in Your Own Project](#setting-it-up-in-your-own-project)
6. [Error Handling & Fallback Strategy](#error-handling--fallback-strategy)

---

## Overview

SpotiFLAC downloads lossless audio from Qobuz and Amazon Music without requiring the user to have an account on either platform. The flow for both services is:

```
Spotify URL/ID
     │
     ▼
Resolve ISRC (via Spotify Web Player reverse engineering)
     │
     ▼
Find matching track on the target platform
     │
     ▼
Obtain a stream/download URL (via third-party APIs)
     │
     ▼
Download the file
     │
     ▼
Decrypt (Amazon only, when the response includes a decryption key)
     │
     ▼
Embed Spotify metadata + cover art
```

---

## Shared Infrastructure

### HTTP Headers

All outbound requests use a common browser `User-Agent` and an `Accept` header:

```
User-Agent: Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/146.0.0.0 Safari/537.36
Accept:     application/json, text/plain, */*
```

Source: `backend/http_headers.go`

---

### Spotify → ISRC Resolution

An ISRC (International Standard Recording Code) is the key used to match a Spotify track to its equivalent on Qobuz or Amazon Music.

SpotiFLAC reverses the Spotify Web Player to obtain ISRCs without a user account. The relevant code is in `backend/spotify_metadata.go` and `backend/spotify_totp.go`.

The resolved ISRC looks like `USUM71703861` — a 12-character alphanumeric string.

---

### Platform Link Resolution

Once an ISRC is known, SpotiFLAC finds the platform-specific URLs (Tidal, Amazon Music, Deezer) using two resolver strategies, tried in order based on a user-configurable preference with automatic fallback.

#### Strategy 1 — Songstats (default)

```
GET https://songstats.com/<ISRC>?ref=ISRCFinder
```

The response is an HTML page. SpotiFLAC scrapes all `<script type="application/ld+json">` blocks and walks the `sameAs` fields looking for URLs that contain:

| Pattern | Assigned to |
|---------|-------------|
| `listen.tidal.com/track` | Tidal URL |
| `music.amazon.com` | Amazon URL |
| `deezer.com` | Deezer URL |

Source: `backend/songstats.go`

#### Strategy 2 — Deezer ISRC API + song.link

**Step 1**: Resolve the Deezer track ID from the ISRC.

```
GET https://api.deezer.com/track/isrc:<ISRC>
```

Response (abbreviated):

```json
{
  "id": 123456789,
  "isrc": "USUM71703861",
  "link": "https://www.deezer.com/track/123456789"
}
```

**Step 2**: Feed the Deezer URL into the song.link API to get equivalent links on other platforms.

```
GET https://api.song.link/v1-alpha.1/links?url=<deezer_url>&userCountry=<region>
```

Response (abbreviated):

```json
{
  "linksByPlatform": {
    "tidal": { "url": "https://listen.tidal.com/track/..." },
    "amazonMusic": { "url": "https://music.amazon.com/albums/.../B0XXXXXXXX" }
  }
}
```

SpotiFLAC reads `linksByPlatform.amazonMusic.url` and `linksByPlatform.tidal.url`.

Source: `backend/songlink.go`, `backend/link_resolver.go`

#### Amazon URL Normalization

Raw Amazon Music URLs can contain the track ASIN in several places. SpotiFLAC normalizes them all to a canonical form:

```
https://music.amazon.com/tracks/<ASIN>?musicTerritory=US
```

Supported input patterns:

| Input format | Extraction method |
|---|---|
| `...?trackAsin=B0XXXXXXXX...` | Query-string split |
| `.../albums/B0XXXXXXXXXX/B0YYYYYYYYYY` | Regex `albumASIN/trackASIN` |
| `.../tracks/B0XXXXXXXX` | Regex `/tracks/<ASIN>` |

Source: `backend/songlink.go` → `normalizeAmazonMusicURL()`

---

### Provider Priority System

For Qobuz, SpotiFLAC uses multiple third-party download APIs. It tracks which ones succeed or fail per session, and automatically prefers recently successful providers over recently failed ones.

The state is stored in a [BoltDB](https://github.com/etcd-io/bbolt) database (`provider_priority.db`) in the app config directory.

Each provider entry stores:

```json
{
  "service": "qobuz",
  "provider": "https://www.musicdl.me/api/qobuz/download",
  "last_outcome": "success",
  "last_attempt": 1714900000,
  "last_success": 1714900000,
  "last_failure": 0,
  "success_count": 12,
  "failure_count": 1
}
```

Sorting priority: `success > unknown > failure`, then `last_success` timestamp descending.

Source: `backend/provider_priority.go`

---

## Qobuz Backend

Source files: `backend/qobuz.go`, `backend/qobuz_api.go`

### Credential Bootstrapping

Every Qobuz API request must be signed with an `app_id` and `app_secret`. SpotiFLAC obtains these without a Qobuz account by scraping the Qobuz open web player:

#### Step 1 — Fetch the shell HTML

```
GET https://open.qobuz.com/track/1
User-Agent: <browser UA>
```

SpotiFLAC scans the HTML for a `<script>` tag whose `src` attribute matches:

```regex
<script[^>]+src="([^"]+/js/main\.js|/resources/[^"]+/js/main\.js)"
```

#### Step 2 — Fetch the JavaScript bundle

```
GET <bundle_url>
User-Agent: <browser UA>
```

SpotiFLAC scans the bundle source for the API config pattern:

```regex
app_id:"(?P<app_id>\d{9})",app_secret:"(?P<app_secret>[a-f0-9]{32})"
```

#### Step 3 — Validate the credentials

SpotiFLAC makes a test `track/search` call (see [API Reference](#qobuz-api-reference)) for a known probe ISRC (`USUM71703861`). If it returns `tracks.total > 0`, the credentials are considered valid.

#### Caching

Scraped credentials are cached to `<app_dir>/qobuz-api-credentials.json` for **24 hours**. Structure:

```json
{
  "app_id": "712109809",
  "app_secret": "589be88e4538daea11f509d29e4a23b1",
  "source": "https://open.qobuz.com/resources/.../js/main.js",
  "fetched_at_unix": 1714900000
}
```

#### Fallback

If scraping fails, SpotiFLAC falls back to embedded default credentials (hardcoded `app_id` and `app_secret`). These may become stale if Qobuz rotates them.

---

### Request Signing

Every call to `https://www.qobuz.com/api.json/0.2` must include a request signature. The algorithm is:

```
1. Normalize the API path:
   Strip leading/trailing slashes, then remove all remaining slashes.
   e.g. "track/search" → "tracksearch"

2. Sort the query parameters alphabetically by key,
   skipping "app_id", "request_ts", and "request_sig".

3. Build the signature payload string:
   <normalized_path><key1><value1><key2><value2>...<unix_timestamp><app_secret>

4. MD5-hash the payload (lowercase hex):
   request_sig = hex(md5(payload))

5. Add to every request:
   ?app_id=<app_id>&request_ts=<unix_timestamp>&request_sig=<signature>&<other_params>
```

Required headers on every signed request:

```
User-Agent: <browser UA>
Accept:     application/json
X-App-Id:   <app_id>
```

Source: `backend/qobuz_api.go` → `qobuzRequestSignature()`, `newQobuzSignedRequestWithCredentials()`

---

### Qobuz API Reference

Base URL: `https://www.qobuz.com/api.json/0.2`

All endpoints below require the signed query parameters described above.

#### Search track by ISRC

```
GET /track/search?query=<ISRC>&limit=1&app_id=...&request_ts=...&request_sig=...
```

Response (abbreviated):

```json
{
  "query": "USUM71703861",
  "tracks": {
    "limit": 1,
    "offset": 0,
    "total": 1,
    "items": [
      {
        "id": 341032040,
        "title": "Track Title",
        "version": "",
        "duration": 213,
        "track_number": 3,
        "media_number": 1,
        "isrc": "USUM71703861",
        "copyright": "© 2023 Label",
        "maximum_bit_depth": 24,
        "maximum_sampling_rate": 96.0,
        "hires": true,
        "hires_streamable": true,
        "release_date_original": "2023-06-01",
        "performer": { "name": "Artist Name", "id": 12345 },
        "album": {
          "title": "Album Title",
          "id": "abc123456789",
          "image": {
            "small":     "https://static.qobuz.com/images/covers/..._230.jpg",
            "thumbnail": "https://static.qobuz.com/images/covers/..._50.jpg",
            "large":     "https://static.qobuz.com/images/covers/..._600.jpg"
          },
          "artist": { "name": "Artist Name", "id": 12345 },
          "label": { "name": "Record Label" }
        }
      }
    ]
  }
}
```

Key fields used downstream:

| Field | Purpose |
|---|---|
| `tracks.items[0].id` | Qobuz numeric track ID, used in download requests |
| `tracks.items[0].hires` | Whether Hi-Res quality is available |
| `tracks.items[0].maximum_bit_depth` | e.g. `24` |
| `tracks.items[0].maximum_sampling_rate` | e.g. `96.0` (kHz) |

#### Get track by Qobuz ID

Used when a Qobuz ID is already known (prefix `qobuz_` on the ISRC field acts as a sentinel).

```
GET /track/get?track_id=<qobuz_id>&app_id=...&request_ts=...&request_sig=...
```

Response shape is a single `QobuzTrack` object (same fields as an item in the search response above).

---

### Qobuz Download Providers

SpotiFLAC uses **two categories** of download API, tried in priority order:

#### Primary — MusicDL

```
POST https://www.musicdl.me/api/qobuz/download
Content-Type: application/json
X-Debug-Key:  <internal key>
User-Agent:   <browser UA>
Accept:       application/json, text/plain, */*

{
  "url": "https://open.qobuz.com/track/<track_id>",
  "quality": "<quality_code>"
}
```

Success response:

```json
{
  "success": true,
  "type": "track",
  "url_type": "direct",
  "track_id": "341032040",
  "quality_label": "Hi-Res 24-bit / 96 kHz",
  "download_url": "https://..."
}
```

Error response:

```json
{
  "success": false,
  "error": "Track not available in this quality",
  "message": ""
}
```

The `X-Debug-Key` header is an internal authentication token that SpotiFLAC derives at runtime from an AES-GCM encrypted blob stored in the binary. It is not a static string you can copy.

#### Fallback — Standard stream APIs

Two interchangeable endpoints:

```
GET https://dab.yeet.su/api/stream?trackId=<qobuz_id>&quality=<quality_code>
GET https://dabmusic.xyz/api/stream?trackId=<qobuz_id>&quality=<quality_code>
```

Both return one of these response shapes:

**Flat:**
```json
{ "url": "https://..." }
```

**Nested:**
```json
{ "data": { "url": "https://..." } }
```

No authentication headers are required for these endpoints.

---

### Quality Codes

| Code | Label | Format |
|------|-------|--------|
| `6`  | CD Lossless | 16-bit / 44.1 kHz FLAC |
| `7`  | Hi-Res 24-bit | 24-bit / up to 96 kHz FLAC |
| `27` | Hi-Res Max | 24-bit / up to 192 kHz FLAC |

SpotiFLAC automatically falls back through `27 → 7 → 6` if a higher quality is unavailable.

---

### Qobuz Full Download Flow

```
Input: Spotify Track ID

1. Resolve ISRC from Spotify Track ID
   └─ Calls Spotify Web Player (reverse engineered, no account needed)

2. Search Qobuz for the ISRC
   └─ GET /track/search?query=<ISRC>&limit=1  (signed)
   └─ Extract: qobuz_track_id, hires flag, quality metadata

3. Get download URL  [multi-provider, with fallback]
   ├─ Try: POST musicdl.me/api/qobuz/download  {url, quality}
   └─ Fallback: GET dab.yeet.su or dabmusic.xyz /api/stream?trackId=...

4. Download the FLAC file
   └─ GET <download_url>  with browser UA

5. Rename file to user-configured filename format

6. Download Spotify cover art  (optional)

7. Embed metadata (from Spotify) and cover art into the FLAC file

Output: /path/to/Artist - Title.flac
```

---

## Amazon Music Backend

Source file: `backend/amazon.go`

### Amazon API Reference

SpotiFLAC uses a proprietary proxy API:

```
Base URL: https://amazon.spotbye.qzz.io
```

#### Get stream info by ASIN

```
GET https://amazon.spotbye.qzz.io/api/track/<ASIN>
User-Agent:  <browser UA>
Accept:      application/json, text/plain, */*
X-Debug-Key: <internal key>
```

The ASIN is the 10-character Amazon Standard Identification Number extracted from the Amazon Music URL (regex: `B[0-9A-Z]{9}`).

Success response:

```json
{
  "streamUrl":     "https://...",
  "decryptionKey": "0123456789abcdef0123456789abcdef"
}
```

- `streamUrl` — direct URL to the encrypted M4A audio file.
- `decryptionKey` — hex key passed to `ffmpeg -decryption_key` to decrypt the file. May be an empty string if the stream is not encrypted.

The `X-Debug-Key` header is derived at runtime from an AES-GCM encrypted blob (same mechanism as the Qobuz MusicDL key), seeded from the string `spotiflac:amazon:spotbye:api:v1`.

---

### Decryption

When `decryptionKey` is non-empty, SpotiFLAC decrypts the downloaded M4A using FFmpeg:

```sh
ffmpeg -decryption_key <hex_key> -i <input.m4a> -c copy -y <output.m4a>
```

After decryption, FFprobe is used to detect the audio codec of the result:

```sh
ffprobe -v quiet -select_streams a:0 \
  -show_entries stream=codec_name \
  -of default=noprint_wrappers=1:nokey=1 \
  <decrypted_file>
```

If the codec is reported as `flac`, the file is renamed to `.flac`. Otherwise it stays as `.m4a`.

---

### Amazon Full Download Flow

```
Input: Spotify Track ID

1. Resolve ISRC from Spotify Track ID

2. Resolve Amazon Music URL from ISRC
   ├─ Fetch Songstats page for ISRC, scrape sameAs links
   │     OR
   └─ Deezer ISRC API → song.link API → extract amazonMusic URL

3. Normalize Amazon URL → canonical track URL
   └─ https://music.amazon.com/tracks/<ASIN>?musicTerritory=US

4. Extract ASIN from URL  (regex: B[0-9A-Z]{9})

5. Call proxy API
   └─ GET https://amazon.spotbye.qzz.io/api/track/<ASIN>
   └─ Extract: streamUrl, decryptionKey

6. Download encrypted M4A
   └─ GET <streamUrl>

7. Decrypt (if decryptionKey is present)
   └─ ffmpeg -decryption_key <key> -i <file> -c copy -y <decrypted_file>

8. Detect codec with ffprobe → rename to .flac if applicable

9. Rename file to user-configured filename format

10. Download Spotify cover art  (optional)

11. Embed metadata (from Spotify) into the file

Output: /path/to/Artist - Title.flac  (or .m4a)
```

---

## Setting It Up in Your Own Project

### Prerequisites

- **Go 1.21+** (the module uses `go 1.26` syntax)
- **FFmpeg and FFprobe** binaries accessible on the system (required for Amazon decryption only)
- Internet access to the third-party APIs listed above

### Step 1 — Copy the relevant source files

The minimum set of files you need:

| File | Purpose |
|---|---|
| `backend/http_headers.go` | Shared HTTP helper |
| `backend/qobuz_api.go` | Qobuz credential scraping and signed request helpers |
| `backend/qobuz.go` | Qobuz downloader |
| `backend/amazon.go` | Amazon downloader |
| `backend/provider_endpoints.go` | API base URLs |
| `backend/provider_priority.go` | Provider priority tracking (BoltDB) |
| `backend/songlink.go` | song.link / Deezer URL resolution |
| `backend/songstats.go` | Songstats scraping |
| `backend/link_resolver.go` | Orchestrates Songstats + song.link resolvers |
| `backend/spotify_metadata.go` | Spotify track metadata + ISRC |
| `backend/spotify_totp.go` | Spotify Web Player token auth |
| `backend/metadata.go` | FLAC/MP4 tag embedding |
| `backend/cover.go` | Cover art download |
| `backend/ffmpeg.go` + platform variants | FFmpeg path helpers |
| `backend/config.go` | App-dir and settings helpers |

### Step 2 — Initialize the database (for provider priority)

```go
if err := backend.InitProviderPriorityDB(); err != nil {
    log.Printf("warning: provider priority DB unavailable: %v", err)
}
defer backend.CloseProviderPriorityDB()
```

### Step 3 — Download a Qobuz track

```go
downloader := backend.NewQobuzDownloader()

filePath, err := downloader.DownloadTrack(
    spotifyID,       // "4uLU6hMCjMI75M1A2tKUQC"
    outputDir,       // "/tmp/music"
    "27",            // quality: "6", "7", or "27"
    "{artist} - {title}", // filename format (see below)
    false,           // includeTrackNumber
    0,               // playlist position (0 = single track)
    trackName,       // from Spotify metadata
    artistName,
    albumName,
    albumArtist,
    releaseDate,     // "2023-06-01"
    false,           // useAlbumTrackNumber
    coverURL,        // Spotify cover image URL
    false,           // embedMaxQualityCover
    trackNumber,     // e.g. 3
    discNumber,      // e.g. 1
    totalTracks,     // e.g. 12
    totalDiscs,      // e.g. 1
    copyright,
    publisher,
    composer,
    ", ",            // metadataSeparator
    spotifyURL,      // full Spotify track URL
    true,            // allowFallback (try lower qualities if 27 fails)
    false,           // useFirstArtistOnly
    false,           // useSingleGenre
    false,           // embedGenre (requires MusicBrainz lookup)
)
```

The return value is the path to the downloaded `.flac` file, or `"EXISTS:<path>"` if the file was already present.

### Step 4 — Download an Amazon Music track

```go
downloader := backend.NewAmazonDownloader()

// Option A: you already have an Amazon Music URL
filePath, err := downloader.DownloadByURL(
    amazonURL,       // "https://music.amazon.com/tracks/B0XXXXXXXX?musicTerritory=US"
    outputDir,
    "",              // quality (not used for Amazon, pass empty string)
    "{artist} - {title}",
    "", "",          // playlistName, playlistOwner
    false, 0,        // includeTrackNumber, position
    trackName, artistName, albumName, albumArtist, releaseDate, coverURL,
    trackNumber, discNumber, totalTracks, false, totalDiscs,
    copyright, publisher, composer,
    ", ",            // metadataSeparator
    "",              // isrcOverride (empty = auto-resolve)
    spotifyURL,
    false, false, false, // useFirstArtistOnly, useSingleGenre, embedGenre
)

// Option B: start from a Spotify track ID
filePath, err := downloader.DownloadBySpotifyID(
    spotifyTrackID,
    outputDir,
    "",
    // ... same remaining args as DownloadByURL ...
)
```

### Filename Format Tokens

Both downloaders support a template-based filename format string. Available tokens:

| Token | Value |
|---|---|
| `{title}` | Track title |
| `{artist}` | Track artist(s) |
| `{album}` | Album name |
| `{album_artist}` | Album artist |
| `{year}` | 4-digit release year |
| `{date}` | Full release date (`YYYY-MM-DD`) |
| `{track}` | Zero-padded track number (`01`, `02`, …) |
| `{disc}` | Disc number |
| `{isrc}` | ISRC code |

Legacy named formats (no `{` braces) are also supported: `"artist-title"`, `"title"`, default is `"title - artist"`.

### Step 5 — Check Qobuz MusicDL availability

Before starting a download you can probe whether the primary MusicDL provider is reachable:

```go
ok := backend.CheckQobuzMusicDLStatus(nil) // nil = uses a 4-second-timeout client
if !ok {
    log.Println("MusicDL is down, will rely on standard stream APIs")
}
```

---

## Error Handling & Fallback Strategy

### Qobuz

| Scenario | Behaviour |
|---|---|
| Scraped credentials fail validation | Use disk-cached credentials, then in-memory cached, then embedded defaults |
| HTTP 400 / 401 from Qobuz API | Force-refresh credentials and retry once |
| Quality `27` unavailable | Retry with quality `7` |
| Quality `7` unavailable | Retry with quality `6` |
| MusicDL fails | Try dab.yeet.su, then dabmusic.xyz |
| All providers fail | Return error with last provider's error message |

### Amazon

| Scenario | Behaviour |
|---|---|
| Songstats scrape fails | Fall back to Deezer ISRC API + song.link |
| Amazon proxy API returns non-200 | Return error immediately (no alternative provider) |
| `decryptionKey` is empty | Skip decryption; use the raw M4A as-is |
| FFmpeg decryption fails | Return error with FFmpeg's last 500 bytes of output |
| Decrypted file is missing or zero-size | Return error |

### Both backends

When a file with the expected name already exists and `RedownloadWithSuffix` is disabled, the download is skipped and `"EXISTS:<path>"` is returned instead of downloading again. This lets callers detect the skip condition cheaply.
