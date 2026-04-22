# How SpotiFLAC Works

This document explains how SpotiFLAC retrieves track information from Spotify and uses third-party platform APIs to download the audio in lossless quality.

---

## Overview

SpotiFLAC works in three main phases:

1. **Retrieve track metadata from Spotify** (no account needed)
2. **Resolve the track to another streaming platform** using the ISRC
3. **Download the lossless audio file** from that platform and embed Spotify metadata into it

---

## Phase 1 — Retrieving Track Info from Spotify

### Step 1: Parse the Spotify Track ID

The user provides a Spotify track URL, URI, or raw ID. SpotiFLAC normalises it into a plain 22-character track ID.

Accepted formats:
- `https://open.spotify.com/track/<id>`
- `spotify:track:<id>`
- The bare 22-character ID

### Step 2: Obtain an Anonymous Spotify Access Token

SpotiFLAC does not require a Spotify account. Instead it authenticates as an anonymous web-player session using a TOTP (Time-based One-Time Password) challenge.

- A TOTP code is generated from a reverse-engineered Spotify secret (`spotify_totp.go`)
- This code is sent to `https://open.spotify.com/api/token` together with `productType=web-player`
- Spotify returns a short-lived `accessToken`
- The token is cached on disk and reused until it expires

### Step 3: Fetch Track Metadata

The track ID (base-62) is converted to a GID (hex) by decoding it from base-62 into a 128-bit integer and formatting it as a 32-character hex string.

The GID is used to call the Spotify internal metadata endpoint:

```
GET https://spclient.wg.spotify.com/metadata/4/track/<gid>?market=from_token
Authorization: Bearer <access_token>
```

The response contains:
- **ISRC** — International Standard Recording Code, the global unique ID for this recording (`isrc_finder.go`)
- **Album GID** — used in a follow-up call to fetch the album and extract the **UPC** (Universal Product Code)

Other track info (title, artist, album name, release date, cover art URL, track/disc numbers, copyright, etc.) is taken from the Spotify metadata response provided by the frontend and is later used only for tagging.

### Step 4: ISRC Fallback via Soundplate

If the Spotify internal API fails (e.g. due to rate limiting), SpotiFLAC falls back to [Soundplate](https://phpstack-822472-6184058.cloudwaysapps.com/):

```
GET https://phpstack-822472-6184058.cloudwaysapps.com/api/spotify.php?q=https://open.spotify.com/track/<id>
```

Soundplate returns the ISRC in a JSON payload. The resolved ISRC is cached locally so repeat lookups are instant.

---

## Phase 2 — Resolving the Track on Other Platforms

With the ISRC in hand, SpotiFLAC resolves the track to one or more streaming platforms using configurable **link resolvers**.

### Resolver A: Songstats

`songstats.go` fetches the Songstats page for the ISRC:

```
GET https://songstats.com/<ISRC>?ref=ISRCFinder
```

The HTML response contains `<script type="application/ld+json">` blocks. SpotiFLAC parses the structured data and extracts `sameAs` URLs pointing to:
- **Tidal** (`listen.tidal.com/track/…`)
- **Amazon Music** (`music.amazon.com/…`)
- **Deezer** (`deezer.com/track/…`)

### Resolver B: Deezer + Song.link/Odesli

`songlink.go` uses two APIs in sequence:

1. **Deezer ISRC lookup** — resolves the ISRC to a Deezer track URL:
   ```
   GET https://api.deezer.com/track/isrc:<ISRC>
   ```

2. **Song.link cross-platform lookup** — uses the Deezer URL to find equivalents on other platforms:
   ```
   GET https://api.song.link/v1-alpha.1/links?url=<deezer_url>&userCountry=<region>
   ```
   This returns a `linksByPlatform` map containing `tidal`, `amazonMusic`, and `deezer` URLs.

The two resolvers can be configured to run in either order, with automatic fallback to the other if the preferred one fails.

### Qobuz Availability Check

For **Qobuz**, availability is checked separately by searching the Qobuz internal API with the ISRC:

```
GET https://www.qobuz.com/api.json/0.2/track/search?query=<ISRC>&limit=1
```

(The request is signed with Qobuz's app credentials — see `qobuz_api.go`.)

---

## Phase 3 — Downloading the Audio

Once SpotiFLAC has the platform URL(s), it downloads the lossless audio using provider-specific logic.

### Tidal (`tidal.go` / `tidal_alt.go`)

**Standard path:**
1. Extract the numeric track ID from the Tidal URL
2. Call a third-party Tidal stream API:
   ```
   GET <tidal_api>/track/?id=<trackId>&quality=<quality>
   ```
   Multiple community-maintained API endpoints are tried in rotation (`tidal_api_list.go`)
3. The API may return:
   - A **direct download URL** (FLAC file) — downloaded directly
   - A **Base64-encoded BTS manifest** — decoded to extract segment URLs, then the segments are concatenated into a single FLAC file

**Alternative path (Tidal Alt.):**

The Spotify track ID is sent directly to a dedicated proxy:
```
GET https://tidal.spotbye.qzz.io/get/<spotifyTrackId>
```
The proxy returns a JSON object with a direct download link, avoiding the need to resolve a Tidal track ID first.

---

### Amazon Music (`amazon.go`)

1. Extract the **ASIN** (Amazon Standard Identification Number) from the Amazon Music track URL
2. Call the Amazon Music proxy:
   ```
   GET https://amazon.spotbye.qzz.io/api/track/<ASIN>
   ```
3. The proxy returns a JSON object with:
   - `streamUrl` — URL to the raw audio file (`.m4a`)
   - `decryptionKey` *(optional)* — if present, the downloaded file is encrypted
4. Download the `.m4a` file
5. If a decryption key was provided, FFmpeg is used to decrypt it:
   ```
   ffmpeg -decryption_key <key> -i <encrypted.m4a> -c copy <decrypted>
   ```
6. If the decrypted codec is detected as `flac` (via `ffprobe`), the output is saved with a `.flac` extension

---

### Qobuz (`qobuz.go`)

1. Search the Qobuz API for the track using the ISRC (resolved in Phase 1)
2. Obtain a signed stream URL from one of the available Qobuz stream APIs:
   ```
   GET https://dab.yeet.su/api/stream?trackId=<id>&quality=<quality>
   GET https://dabmusic.xyz/api/stream?trackId=<id>&quality=<quality>
   GET https://qobuz.spotbye.qzz.io/api/track/<id>?quality=<quality>
   ```
3. Quality levels fall back automatically if the requested tier is unavailable:
   - `27` (Hi-Res Max) → `7` (24-bit Standard) → `6` (16-bit Lossless)
4. The FLAC file is downloaded directly from the signed URL

---

## Phase 4 — Metadata Embedding

After downloading, SpotiFLAC embeds the full Spotify metadata into the audio file:

| Tag | Source |
|-----|--------|
| Title, Artist, Album | Spotify track/album metadata |
| Album Artist | Spotify |
| Release Date | Spotify |
| Track / Disc Number | Spotify |
| Total Tracks / Discs | Spotify |
| Cover Art | Downloaded from Spotify CDN URL |
| ISRC | Spotify internal API (Phase 1) |
| UPC | Spotify album metadata (Phase 1) |
| Copyright, Publisher, Composer | Spotify |
| Genre *(optional)* | [MusicBrainz](https://musicbrainz.org) — queried by ISRC |
| Comment / URL | Original Spotify track URL |

### Genre via MusicBrainz

If genre embedding is enabled, SpotiFLAC queries:
```
GET https://musicbrainz.org/ws/2/recording?query=isrc:<ISRC>&inc=tags&fmt=json
```
The top-voted community tags are used as the genre. Requests are rate-limited to respect MusicBrainz's 1 request/second policy.

---

## API / Service Summary

| Service | Purpose |
|---------|---------|
| Spotify Web Player (`open.spotify.com/api/token`) | Anonymous access token via TOTP |
| Spotify Client API (`spclient.wg.spotify.com`) | Internal track/album metadata, ISRC, UPC |
| Soundplate | Fallback ISRC lookup from Spotify URL |
| Deezer API (`api.deezer.com`) | ISRC → Deezer track URL cross-reference |
| Song.link / Odesli (`api.song.link`) | Cross-platform URL resolution |
| Songstats (`songstats.com`) | ISRC → platform URLs via structured data |
| Tidal stream APIs (community-hosted) | Tidal track stream URL |
| `tidal.spotbye.qzz.io` | Alternative Tidal download proxy |
| `amazon.spotbye.qzz.io` | Amazon Music stream URL + decryption key |
| Qobuz stream APIs (`dab.yeet.su`, `dabmusic.xyz`, `qobuz.spotbye.qzz.io`) | Qobuz signed stream URL |
| MusicBrainz (`musicbrainz.org`) | Genre tags by ISRC |
| FFmpeg | Audio decryption, format conversion, metadata embedding |
