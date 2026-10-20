# SYP — Implementation Plan

_Status: draft for review · Last updated: 2026-10-20 · Owner: @lgibelli_

SYP (Sync Your Playlists) is a Manifest V3 browser extension that exports, imports and copies playlists between **Spotify**, **YouTube Music**, **Apple Music** and **Amazon Music**. It works inside the web players the user is already logged into and reuses those sessions. There are no developer API keys and no servers, and no user data leaves the browser.

---

## 1. Why an extension (and not an app or a SaaS)

| Constraint (as of Oct 2026) | Impact |
|---|---|
| Spotify Web API development mode is capped at **5 users** and needs the developer to have Premium (Feb 2026). Extended quota needs a registered business with **250k MAU** (May 2025). | An official-API client can't be distributed. |
| Spotify removed the **implicit grant** OAuth flow on 27 Nov 2025. | Older tools that use it, such as spotify2ytmusic and spotify-backup, no longer work. |
| Apple MusicKit needs a paid Apple Developer Program membership ($99/yr) per app. | Possible, but adds cost and a token-signing backend. |
| YouTube Music has **no official API**. Every tool reverse-engineers InnerTube. | It's unofficial whichever form we choose. |
| Amazon Music Web API is a closed program. | No official route. |

An extension runs in the user's own logged-in tabs. It can call the same internal endpoints the web players use, with the user's own session. That gives no per-app user caps, no keys, no backend and no hosting cost. The privacy story is simple: everything stays local.

Trade-off we accept: internal endpoints change without notice. The architecture is built around **isolating and quickly patching** each service adapter (see §9).

## 2. Goals and non-goals

### Goals (v1.0)
1. **Export** any playlist, Liked Songs or library from any supported service to a portable file (SYP JSON, CSV).
2. **Import** a file into any supported service, creating or appending to playlists.
3. **Copy** playlists directly from service A to service B, with high-quality matching and a review step for uncertain matches.
4. **Resumable** long jobs (thousands of tracks) that survive tab or browser restarts.
5. Publishable on the **Chrome Web Store**, with Edge and Firefox as fast follows.

### Non-goals (v1.0)
- Two-way continuous sync (planned for after 1.0, see §11, Phase 5).
- Copying audio, downloading or playback control. **Never**: this is a hard line for store policy and legal reasons.
- Podcasts and audiobooks.
- Mobile.
- Any server component, telemetry or account system.

## 3. Prior art and what we reuse

- **spotify2ytmusic** (CC0, Python): the "find the album, then the exact track" matching idea, and `tests/playliststest.json` (a real 15k-line Spotify export) for golden matching tests. CC0 allows reuse.
- **ytmusicapi** (MIT, Python): the reference for InnerTube request and response shapes for YouTube Music. We port shapes and parsers, not code verbatim, and credit it in the README.
- Existing extensions: Playlist Exporter (Spotify/Apple export only), Youfy (YT→Spotify, 50-track cap), HarmonySync (Firefox, no Amazon, unproven). None covers all four services with import and export.
- Hosted SaaS services: TuneMyMusic, Soundiiz, FreeYourMusic. SYP's pitch against them: free, local, no caps, no third party holding your tokens.

## 4. Architecture

### 4.1 Components

```
┌──────────────────────────── Extension ────────────────────────────┐
│                                                                   │
│  App page (app.html, full tab)          Service worker            │
│  ─ wizard UI, review, reports           ─ opens/finds service tabs│
│  ─ Job orchestrator (long-lived)        ─ permission requests     │
│  ─ Matching engine                      ─ badge/progress, alarms  │
│  ─ Job store (IndexedDB)                                          │
│          │                                                        │
│          │ chrome.scripting.executeScript({ world: "MAIN" })      │
│          ▼                                                        │
│  ┌──────────── Service tab (e.g. music.youtube.com) ───────────┐  │
│  │ Page agent (bundled, MAIN world)                            │  │
│  │  ─ reads session (tokens, ytcfg, MusicKit instance…)        │  │
│  │  ─ performs same-origin / CORS-allowed fetches              │  │
│  │  ─ returns plain JSON                                       │  │
│  │ Token sniffer (MAIN world, document_start), where needed    │  │
│  └─────────────────────────────────────────────────────────────┘  │
└───────────────────────────────────────────────────────────────────┘
```

Key decisions:

1. **Requests run in the page context, not from the extension origin.** The services check `Origin`, origin-bound auth hashes (YouTube's `SAPISIDHASH` includes the origin) and first-party cookies. Requests from `chrome-extension://` would fail or look suspicious. Running in the page's MAIN world makes our requests indistinguishable from the web player's own.
2. **RPC through `chrome.scripting.executeScript` with `world: "MAIN"`.** It returns the awaited result of an async function, so no custom `postMessage` bridge is needed. The injected functions are bundled in the extension package, which complies with the "no remote code" policy. A small `window.__syp` page agent is installed once per tab and then called by operation name.
3. **The orchestrator lives in an extension page (`app.html`), not the service worker.** MV3 service workers are killed after about 30s idle, which breaks long jobs. The app tab is long-lived and visible, and the user sees progress. Job state is checkpointed to IndexedDB after every batch, so closing the tab only pauses the job.
4. **Service tabs:** the orchestrator finds an existing tab for the service or opens one in the background (`active: false`). It detects "not logged in" and asks the user to log in, then resumes.
5. **Session capture per service, always passive:**
   - **Spotify:** a manifest-registered MAIN-world content script at `document_start` wraps `fetch`/`XMLHttpRequest` to capture the player's own tokens. SYP never mints tokens.
   - **YouTube Music:** reads `ytcfg` and cookies on demand.
   - **Apple Music:** reads the `MusicKit` instance.
   - **Amazon Music:** reads `window.amznMusic.appConfig`, or hooks the Redux store at `document_start`.

   See §7.

### 4.2 Service adapter interface

Every service implements the same contract. Capabilities are declared so the UI can grey out unsupported actions per service.

```ts
// Sketch only. Not final code.
type ServiceId = "spotify" | "ytmusic" | "apple" | "amazon";

interface Capabilities {
  readPlaylists: boolean;
  readLiked: boolean;
  writePlaylists: boolean;
  writeLiked: boolean;
  isrcLookup: boolean;          // exact lookup by ISRC on this service
  maxAddBatch: number;          // tracks per add request
}

interface ServiceAdapter {
  id: ServiceId;
  capabilities: Capabilities;
  getSession(): Promise<Session>;                       // logged in? user, market/storefront
  listPlaylists(): AsyncIterable<PlaylistSummary>;
  getTracks(ref: PlaylistRef | "liked"): AsyncIterable<Track>;
  findByIsrc?(isrc: string): Promise<Candidate[]>;
  search(q: TrackQuery): Promise<Candidate[]>;          // returns raw candidates; scoring is central
  createPlaylist(p: NewPlaylist): Promise<PlaylistRef>;
  addTracks(ref: PlaylistRef, ids: string[]): Promise<AddResult>; // adapter batches internally
  like?(ids: string[]): Promise<AddResult>;
}
```

Adapters are split into:
- `endpoints.ts`: all URLs, GraphQL operation names and hashes, and client versions **in one file per service**, so a breakage is usually a one-file fix.
- `page-agent.ts`: the code that runs in the MAIN world (session read, raw fetch).
- `parse.ts`: pure functions that turn raw responses into the domain model. Unit-tested against recorded fixtures.
- `adapter.ts`: pagination, batching, retries and error mapping.

### 4.3 Throttling, retries and errors

- A per-service **token-bucket rate limiter** with conservative defaults, adjustable in Advanced settings.
- Retry with exponential backoff and jitter on 429/5xx/network errors, honouring `Retry-After`.
- Typed errors: `NotLoggedIn`, `SessionExpired`, `RateLimited`, `EndpointChanged` (unexpected shape or 4xx on a known-good call), `NotAvailableInRegion`, `PlaylistFull`, `Unknown`. `EndpointChanged` shows a "this service may have changed, check for an SYP update" banner with a one-click diagnostic report the user can copy.
- **Idempotent writes:** before appending, read the destination playlist and skip tracks already present. A crashed job can be safely re-run.
- **Verify after write:** Spotify and Apple can return success while silently dropping tracks. After every batch, re-read the destination count or contents and re-queue the shortfall. This is a core part of the write loop, not an optional extra.
- **Bisect on batch failure:** on YouTube Music (one bad videoId fails the batch) and Apple (some recordings return 500), a failed batch is split in half recursively. The offending track(s) are isolated and marked unmatched, and the rest succeed.
- **Hard-stop conditions** pause the job instead of retrying: the YouTube Music creation gate or "too many playlists" response, a network-wide "automated queries" 429, an exhausted Apple hourly budget, and repeated 429s on Spotify. The job resumes automatically after a cool-down, or manually.
- **Request-budget scheduler:** each adapter declares its limits (for example, Apple at about 250 requests per hour). The orchestrator plans the job against them and shows an honest ETA. Long jobs keep running in the background and survive restarts (§4.1.3).

## 5. Data model and file formats

### 5.1 Domain model

```ts
interface Track {
  title: string;
  artists: string[];          // all credited artists, in order
  album?: string;
  durationMs?: number;
  isrc?: string;
  explicit?: boolean;
  ids: Partial<Record<ServiceId, string>>;  // native IDs we know (track URI, videoId, catalog id, ASIN)
  addedAt?: string;           // ISO-8601, when available
}

interface Playlist {
  name: string;
  description?: string;
  kind: "playlist" | "liked";
  source: { service: ServiceId; id: string; url?: string };
  tracks: Track[];
}
```

### 5.2 SYP JSON (`*.syp.json`), the canonical export

```json
{
  "format": "syp",
  "version": 1,
  "exportedAt": "2026-10-20T12:00:00Z",
  "generator": "SYP 1.0.0",
  "playlists": [ { "name": "…", "kind": "playlist", "source": {…}, "tracks": [ … ] } ]
}
```

Versioned and validated on import with a schema (zod). Unknown fields are preserved.

### 5.3 Interop formats
- **CSV export/import**, columns compatible with Exportify/TuneMyMusic where possible (Track name, Artist name(s), Album, ISRC, Duration).
- **Import of spotify2ytmusic `playlists.json`** (spotify-backup format). It's cheap to support and gives users of the old tool a migration path.
- Plain-text "Artist - Title" lists, with best-effort parsing for import.

## 6. Matching engine

The quality of the product depends on this. It runs centrally, not per adapter, so every service pair benefits equally.

### 6.1 Pipeline for each source track and destination service
1. **Known-ID cache:** if this track (by ISRC, or by normalized `artist|title|durationBucket`) was matched to this destination before, reuse the result. The cache is stored in IndexedDB and makes re-syncs nearly free.
2. **Native ID:** if `track.ids[dest]` is already known (for example, re-importing an SYP file), use it.
3. **ISRC lookup** where the destination supports exact ISRC search (expected: Apple `filter[isrc]`; see §7.3). Accept if exactly one result, or pick the best result by score. **Spotify has no usable ISRC search** (the public API is off-limits and Pathfinder doesn't expose ISRC; see §7.1), so it skips this step, and so does YouTube Music. Amazon has ISRC search only through the GraphQL route (§7.4). ISRCs are still worth *capturing* on export wherever a source provides them.
4. **Metadata search:** query `"{primary artist} {clean title}"`, then fall back to `"{clean title} {album}"` and, for YouTube Music, the album-first strategy taken from spotify2ytmusic.
5. **Score candidates** (0–1) from:
   - title similarity after normalization (token-set ratio / Jaro-Winkler),
   - artist overlap across all credited artists,
   - duration delta (strong signal: ≤2s good, ≤5s acceptable, >10s penalized hard),
   - album similarity (weak signal),
   - version-marker agreement: penalize `live`, `remix`, `acoustic`, `instrumental`, `karaoke`, `cover`, `sped up`, `slowed`, `remaster` when present on only one side,
   - explicit/clean agreement (weak),
   - for YouTube Music, prefer official audio tracks over music videos and user uploads.
6. **Classify:** `≥ 0.85` auto-match · `0.60–0.85` needs review · `< 0.60` unmatched. Thresholds are tuned on the golden set.

### 6.2 Normalization
Lowercase, Unicode NFKD with diacritics stripped, remove `(feat. …)`/`[feat. …]`/`ft.`, strip trailing ` - 2011 Remaster`, ` - Radio Edit` and similar into a separate "version" field, unify `&`/`and`/`,` in artist lists, and collapse whitespace and punctuation.

### 6.3 Review UI
Uncertain tracks show the top 3 candidates side by side: title, artists, album, duration and a link to open in the service. Actions: pick one, search manually, or skip. Unmatched tracks are exported as a CSV report.

### 6.4 Evaluation
A **golden set** of about 300 hand-verified track pairs per service pair, built from `playliststest.json` plus deliberately hard cases: classical, live albums, K-pop/J-pop with non-Latin titles, remasters and very short titles like "Survival" by Yes. CI runs the scorer against recorded search fixtures and fails if precision drops. Target: **≥ 97% precision on auto-matches** and **≥ 90% recall** (auto plus review).

## 7. Service adapters

> Findings below come from public sources (open-source projects, issue trackers, blogs) as of Oct 2026. Everything must be confirmed against live traffic in Phase 0 before implementation.

### 7.1 Spotify (open.spotify.com)

Confidence: **medium**. The read path is well trodden. Writes work but have sharp edges. Spotify is the most actively hostile of the four.

**Session / auth**
- The web player gets its bearer token from `GET /api/token?...&totp=…&totpVer=N`. The TOTP secret is obfuscated in the JS bundle and **rotates often** (v12 in mid-2025, v61 by 2026-10-16). Spotify sent a cease-and-desist that took down a public secret-scraper repo in Oct 2025. **SYP must not mint tokens.**
- **Approach:** a MAIN-world content script registered in the manifest (`"world": "MAIN"`, `run_at: document_start`). An inline script would be blocked by Spotify's CSP. It wraps `fetch` and `XMLHttpRequest.setRequestHeader` and captures `authorization`, `client-token` and `spotify-app-version` from the player's own requests. Tokens live about 1h and the page refreshes them itself, so we always take the latest. Fallback: read headers with `chrome.webRequest.onBeforeSendHeaders` (needs the `webRequest` permission, so we'd prefer to avoid it). Several current extensions (hearted.music, chorus, Segue) use one of these two approaches.

**Do not use `api.spotify.com/v1` with the web-player token.** Since late 2025 it returns 429 on the first request (many independent reports through Oct 2026), and retrying lands in an hours-long penalty box. Everything goes through the internal APIs below.

**Read (Pathfinder GraphQL):** `POST https://api-partner.spotify.com/pathfinder/v2/query` with `{operationName, variables, extensions.persistedQuery.sha256Hash}` and headers `Authorization`, `App-Platform: WebPlayer`, plus `client-token` and `spotify-app-version` when captured.
- `libraryV3`: user's playlists (page size 50). Liked Songs is `spotify:collection:tracks`.
- `fetchPlaylist` / `fetchPlaylistContents`: `{uri, offset, limit}`. Use a page size of about 300; very large pages time out with 504.
- `fetchLibraryTracks`: Liked Songs, `{offset, limit}`.
- `searchDesktop` / `searchTracks`: free-text search only.
- `profileAttributes`: session check and display name.

**Write**
- **Create playlist** is not GraphQL: `POST https://spclient.wg.spotify.com/playlist/v2/playlist` with an `UPDATE_LIST_ATTRIBUTES` op (name) returns the new URI. Then `addItemsToRootlist` (GraphQL) adds it to the user's library.
- **Add tracks:** `addToPlaylist` with `{playlistUri, playlistItemUris[], newPosition: BOTTOM_OF_PLAYLIST}`, about 100 URIs per call (the real maximum is unknown; confirm in Phase 0).
- ⚠️ **Writes can fail silently:** after about 5–10 rapid `addToPlaylist` calls Spotify returns 200 but drops the write. SYP must **pace writes 4–7s with jitter** and **re-read `totalCount` after each batch**, re-queuing any shortfall.

**Persisted-query hashes rotate** (read operations roughly weekly to monthly; `libraryV3`, `fetchPlaylist` and `searchTracks` all changed between Oct 8 and Oct 16, 2026). Old hashes often keep working for a while. Strategy:
1. Ship known-good hashes in `endpoints.ts`.
2. On `PersistedQueryNotFound` (often HTTP 200 with an error body) or HTTP 412, re-scan the loaded web-player bundle with the regex `"<op>","query","<64-hex>"`, using the bundle's chunk table for lazily loaded search chunks. Cache the result and retry once.
3. Still failing: raise `EndpointChanged`.

The hash is data read from Spotify's own page, so this doesn't count as remote code.

**Rate limits:** space reads about 450ms apart. Pathfinder is fairly tolerant. On a 429, back off exponentially from 30s up to 300s and stay in "slow mode" for the rest of the job. The page can't read `Retry-After` because of CORS, so we use fixed backoff steps.

**Matching metadata:** Pathfinder returns name, URI, `duration.totalMilliseconds`, artists, album, `contentRating` (explicit), `playability.playable` and `addedAt`. **It has no ISRC.** A possible way to get ISRC is `spclient.wg.spotify.com/metadata/4/track/{gid}?market=from_token` (some tools get `external_id[type=isrc]`, others get 404 or protobuf), to be **tested in Phase 0**. Either way, **Spotify has no usable ISRC search**, so Spotify as a destination always uses fuzzy matching.

**Risks:** Spotify's User Guidelines explicitly forbid automated access, scraping, reverse-engineering and "circumventing any technology". The practical risk is to the **user's account** (Spotify tightened enforcement after the Anna's Archive scrape in Dec 2025). Mitigations: gentle defaults, write pacing, no mass operations beyond the user's own library, and a clear in-app warning.

**References:** hearted.music, chorus, Segue, folia-major (full read/write incl. create), songmirror, jbaruch/spotify-playlist-driver (write pacing), BitChord.

### 7.2 YouTube Music (music.youtube.com)

Confidence: **high**. ytmusicapi (1.12.3, Sep 2026) is a maintained reference for every request and response shape, and in-page execution avoids its main pain point. Since Aug 2025 OAuth is broken again and the maintainer considers it unwinnable. Cookie auth tied to the browser's own session still works.

**Session / auth (all from the MAIN world of the music.youtube.com tab)**
- Endpoint: `POST https://music.youtube.com/youtubei/v1/<endpoint>?prettyPrint=false`, `fetch(..., { credentials: "include" })`.
- Body: `{ context: ytcfg.get("INNERTUBE_CONTEXT"), ...payload }`, using the page's own `WEB_REMIX` client and version. We never invent a client version.
- Headers from `ytcfg`: `X-Goog-AuthUser` (= `SESSION_INDEX`), `X-Goog-Visitor-Id` (= `VISITOR_DATA`), `X-Goog-PageId` (= `DELEGATED_SESSION_ID`, for brand accounts), `X-Origin`, `X-Youtube-Client-Name/Version`.
- `Authorization`: computed per request from cookies readable in `document.cookie`. Use the current 3-hash form, `SAPISIDHASH`, `SAPISID1PHASH` and `SAPISID3PHASH`, built from `SAPISID`, `__Secure-1PAPISID` and `__Secure-3PAPISID`, including `USER_SESSION_ID` in the hash input when present. Fall back to the legacy single `SAPISIDHASH`, which ytmusicapi still uses successfully.
- Running in-page avoids the problem of copied cookies dying when used from another IP (`__Secure-*PSIDTS` is tied to the IP that captured it).
- PO tokens / attestation currently apply only to playback (GVS/player), not to browse, search or edit. **Watch item.**

**Read**
- Library playlists: `browse { browseId: "FEmusic_liked_playlists" }`. Skip grid item 0 (the "New playlist" tile). Continue with `ctoken`/`continuation` URL params.
- Playlist tracks: `browse { browseId: "VL" + id }`, 100 items per page, continuation via `continuationItemRenderer` token → `appendContinuationItemsAction`. `musicEditablePlaylistDetailHeaderRenderer` present means the user owns the playlist.
- Liked Music: `browseId: "VLLM"`, same parser.
- Search (songs filter): `search { query, params: "EgWKAQIIAWoMEA4QChADEAQQCRAF" }`. There's also a videos filter.

**Write**
- Create: `playlist/create { title, description, privacyStatus, videoIds? }` → `playlistId`. Sanitize `<`/`>` out of titles. HTML in descriptions is stripped. We can pass the first batch of `videoIds` inline.
- Add: `browse/edit_playlist { playlistId, actions: [{ action: "ACTION_ADD_VIDEO", addedVideoId, dedupeOption: "DEDUPE_OPTION_SKIP" }] }`. **Dedupe on our side first.** Batches of about 100 (101 tested; no documented max).
- Like: `like/like { target: { videoId } }` (optional "Liked Songs → likes" mode, §14).

**Quirks and limits SYP must handle**
- **Playlist cap of 5,000 items** (400 "Maximum playlist size exceeded"). Auto-split into `Name (part 2)`, … and warn first. Liked Music is reportedly about 10k.
- **Settle window after create:** the first edit can return 409 Conflict or 400 "Precondition check failed" for about 20s. Retry up to 8 times, 5s apart.
- **One bad or removed videoId fails the whole batch** (400 Precondition). On a batch failure, split the batch in half and retry each half to find and skip the bad ID(s).
- **Creation gating (Jul 2026):** `playlist/create` may return HTTP 200 with only a `showEngagementPanelEndpoint` (`PAfeature_enablement`) and no `playlistId`. It's account-scoped and lasts more than an hour. Older form: "You are creating too many playlists" (429/400, multi-hour). **Treat either as a hard stop:** pause the job, tell the user, and resume later.
- Pacing defaults: 1–3s jittered gap between write calls, **≥ 30–60s between playlist creates**, a per-session create cap (default 20, configurable), backoff from 5s to 60s. A network-wide 429 ("automated queries") pauses the job.

**Matching metadata:** videoId, title, artists[{name,id}], album, duration (seconds), isExplicit, isAvailable, and **`musicVideoType`**: `ATV` is the official audio "song", `OMV` the official music video, `UGC` a user upload. Prefer `ATV`. Allow `OMV` and `UGC` only as fallbacks (opt-in for UGC). The `next` endpoint can map OMV to its ATV counterpart. **No ISRC** anywhere, so YouTube Music as a destination always uses fuzzy matching, and as a source it exports without ISRC.

**Risks:** YouTube's ToS forbid accessing the service "using any automated means". As with Spotify, the main risk is to the user's account (an unconfirmed report of a spam-policy termination after mass playlist creation). Google also runs the Chrome Web Store, so the takedown risk is asymmetric. Expect parser churn every few months (ytmusicapi had about 5 shape fixes from 2025 to 2026). Store listing: "for YouTube Music" wording only, no YouTube logos.

**References:** ytmusicapi (source of truth for shapes), yt-dlp (`_base.py`, 3-hash auth), PlaylistPlus userscript (in-page approach, 100 per batch with about 1.2s pacing).

### 7.3 Apple Music (music.apple.com)

Confidence: **medium-high** on the mechanics, which come from Apple's own API docs and today's live bundle. **Low** on throughput, because the rate limit is tight and poorly understood.

**Session / auth**
- **Developer token:** a JWT (`iss: "AMPWebPlay"`, `kid: "WebPlayKid"`, ES256, origin-bound to apple.com) hard-coded in the web app bundle and passed to `MusicKit.configure()`. It currently rotates about every 70 days with bundle releases, so we **never hard-code it**.
- **User token:** the `media-user-token` cookie (not HttpOnly). Storefront is in the `itua` cookie.
- **Approach:** a MAIN-world agent that waits until `MusicKit.getInstance()` exposes `developerToken`, `musicUserToken`, `isAuthorized` and `storefrontId`. applemusic-mcp tested this live. Fallback: read the cookie and regex the bundle for the `AMPWebPlay` JWT. That bundle scraping broke twice in 2025–26 when files were renamed, so it's only a fallback.
- Requests from the page context get `Origin: https://music.apple.com`, which this token requires. Use the app's own headers: `authorization: Bearer <jwt>`, `media-user-token`, `x-apple-client-version`.
- Hosts: `amp-api.music.apple.com` for library and catalog, and `amp-api-edge.music.apple.com` for search.
- **Subscription:** assume an active Apple Music subscription is required for library and playlist access. Check `GET /v1/me/account?meta=subscription` (`meta.subscription.active`), and show a clear message on 403.

**Read**
- Library playlists: `GET /v1/me/library/playlists?limit=100`, follow `next`. Attributes include `canEdit` and `playParams.globalId`.
- Playlist tracks: `GET /v1/me/library/playlists/{p.id}/tracks?limit=100&include=catalog`. Library songs have **no ISRC** of their own, so `include=catalog` gives ISRC, `durationInMillis` and the catalog ID.
- Catalog (Apple-curated or shared) playlists: `GET /v1/catalog/{sf}/playlists/{id}?include=tracks&limit[tracks]=300`.
- Storefront: `GET /v1/me/storefront`.

**Search / lookup**
- **ISRC batch lookup:** `GET /v1/catalog/{sf}/songs?filter[isrc]=A,B,…` takes **up to 25 ISRCs per request**, the best exact-ID path of the four services. One ISRC can return several songs; pick by score. Misses are silent.
- Text search: `GET /v1/catalog/{sf}/search?types=songs&term=…&limit=25`.
- Storefront equivalents: `songs?filter[equivalents]={id}` maps a song to the version playable in the user's storefront (useful for SYP files exported in another country).

**Write**
- Create: `POST /v1/me/library/playlists` `{ attributes: { name, description, isPublic }, relationships: { tracks: { data: [{ id, type: "songs" }] } } }` → 201. We can pass the first batch inline.
- Add: `POST /v1/me/library/playlists/{id}/tracks` `{ data: [{ id, type: "songs" }] }` → 204, appends.
- The web token also allows deleting playlists or tracks, renaming, and a full `PUT` replace. The public API doesn't. We **don't need these for v1**. They would be useful for Phase 5 sync (removal propagation), to be explored then.
- ⚠️ **Only playlists with `canEdit: true` are writable.** Playlists made in the Music desktop/iOS app return 500 even from the page. **SYP always creates a new playlist on Apple Music** and only appends to playlists it created.
- ⚠️ **Silent drops and 500s:** adds can return 2xx while dropping some tracks, and some recordings reproducibly fail with 500 "Unable to update tracks". Large creates (thousands of tracks) have failed with 500. Use batches of about 100, **re-read and verify after each batch**, and on a 500 split the batch in half to isolate the problem track.
- Library IDs (`p.…`, `i.…`) differ from catalog IDs and change if items are removed and re-added. New items can take a few seconds to appear. Store catalog IDs in SYP files, never library IDs.

**Rate limits (the main constraint)**
- One report (Jul 2026, web token used from outside the browser) saw 429 `{"code":"42900"}` after about **250–280 requests per rolling hour**, with no `Retry-After` header, and a 15-minute pause didn't clear it. The quota may be shared per token. In-page behaviour is unknown, so **measure in Phase 0**.
- Planning implication: with ISRCs, 1,000 tracks need about 40 lookup requests plus about 10 write requests. **Without ISRCs it's about 1,000 searches, which could take ~4h** at that rate. So:
  - (a) get ISRCs from the source wherever possible (Spotify metadata endpoint, §7.1, Phase 0),
  - (b) a request-budget scheduler that shows an honest ETA and runs jobs in the background across hours,
  - (c) aggressive caching,
  - (d) treat a 429 as "unknown, retry later", never as "not found".

**Matching metadata:** catalog songs have `isrc`, `durationInMillis`, `name`, `artistName`, `albumName` and `contentRating`. Uploaded or unmatched library songs probably have no catalog link: export them by metadata only.

**Risks:** the Apple Media Services Terms forbid automated scraping and say to "access our Services only using Apple's software". Apple says other methods "may be blocked at any time". No enforcement against this approach has been reported. Token and bundle rotation is handled by reading at runtime.

**References:** applemusic-mcp (in-page MusicKit token capture, live-tested writes), gamdl (token extraction history), Apple Music API docs (request and response shapes match amp-api).

### 7.4 Amazon Music (music.amazon.com + regional domains)

Confidence: **low**. Several plausible routes exist, none verified with a logged-in account, and the player may be rewritten soon. This is why Amazon is last in the roadmap and has an export-only fallback.

**Official API: not an option.** Web API v1 and v2 are closed betas ("limited to already approved developers", access via an Amazon business development contact). The Program Requirements even forbid integrating "with a third-party music service". That doesn't bind non-participants, but it signals Amazon's stance.

**Session (all variants start here)**
- The player (codename "Skyfire") POSTs `/config.json?skipToken=false&clientApplication=skyfire` and stores the result in `window.amznMusic.appConfig`: `accessToken`, `accessTokenExpiresIn`, `csrf { token, ts, rnd }`, `deviceId`, `deviceType`, `sessionId`, `customerId`, `marketplaceId`, `musicTerritory`, `siteRegion`, `benefits`. The token is refreshed via `/pandaToken`.
- **Approach:** a MAIN-world agent reads `window.amznMusic.appConfig` (or awaits `amznMusic.dynamicConfig.loaderPromise`). It's no longer inline in the HTML, so tools that regex the page broke. We never scrape the HTML.
- Always use the account's **home marketplace** domain and its `siteRegion` (NA/EU/FE). Using the wrong marketplace fails. Host permissions must cover the regional domains (.com, .co.uk, .de, .fr, .it, .es, .ca, .com.au, .co.jp, .in, .com.br, .com.mx).

**Candidate routes, to be tried in this order in Phase 0**
1. **GraphQL (`gql.music.amazon.dev`)**, as used by SongMirror and confirmed working by a user on music.amazon.fr in Sep 2026. `authorization: AmznMusic base64({deviceId, deviceType, accessToken})` plus a public client `x-api-key`. Has clean operations: `user.playlists(roleFilter: OWNER)`, `playlist(id).tracks` (cursor, 100), **`searchTracks` with an ISRC filter**, `createPlaylist`, `appendTracks(rejectDuplicateTracks)`. It returns ISRC and duration. **Best route if it works.** Unknowns: whether CORS allows the call from the music.amazon.com page (if not, the extension origin can call it with host permission, since auth is a header token, not cookies), and whether `appendTracks` accepts batches (SongMirror adds one track at a time).
2. **The player's own "mesk skill" endpoints (`{region}.mesk.skill.music.a2z.com/api/…`)**: `showLibraryHome`, `showHome` (playlist deeplink), `showSearch`, `create_playlist`, `add_tracks_to_playlist { playlistId, selectedIds: [ASIN…], rejectDuplicate }`. Headers: `x-amzn-authentication`, `x-amzn-csrf`, `x-amzn-device-id`, `x-amzn-session-id`, and others. ⚠️ The responses are **server-driven UI** (`{ methods: [templates/widgets/items] }`), not data. Parsing them is brittle, and list items carry only title, artist, album and an "mm:ss" duration (**no ISRC**).
3. **Dispatch through the app's own Redux store**: hook `__REDUX_DEVTOOLS_EXTENSION_COMPOSE__` at `document_start` to capture the store, then dispatch `InvokeHttpSkillMethod`. SynQApp's MV3 extension does this. This lets the app build its own headers and CSRF, which is robust to header changes but tied to internal action names.
4. **Fallback (export only):** DOM scraping of the virtualized list (shadow-DOM `music-image-row` components, auto-scroll, track ASIN from `href ?trackAsin=`). Import by clicking through the UI is **out of scope**: too slow and fragile.

**Decision rule:** if route 1 or 2 supports reliable writes in Phase 0, Amazon gets full read/write. Otherwise ship **Amazon = export-only** (route 1, 2 or 4) and say so in the listing.

**Matching metadata:** tracks are ASINs (`B0…`), which differ between territories. Playlists are UUIDs. ISRC is available via GraphQL. Otherwise it's only on per-track page JSON-LD (`isrcCode`, one request per track, too expensive). Store ASIN plus marketplace in SYP files.

**Open questions:** Prime (shuffle-only, full catalog since Nov 2022) versus Unlimited, and whether Prime-only accounts can add arbitrary tracks to playlists. Page-size and pagination behaviour. Rate limits (unknown, 429s reported).

**Risks:** highest breakage risk of the four. `config.json` changed to POST-only for CSRF in Sep 2026, inline config was removed, and a React Native Web rewrite ("Dragonfly") is in the bundle behind an internal flag, which would likely invalidate routes 2–4. Same terms-of-service posture as the others.

**References:** SongMirror (GraphQL route), lib-origin (mesk endpoints), SynQApp/Extension (Redux hook, MV3), Mornye extension (config.json POST fix, regional ASINs), save_amazon_music_playlist (DOM export).

### 7.5 Expected capability matrix (to be confirmed in Phase 0)

| | Spotify | YouTube Music | Apple Music | Amazon Music |
|---|---|---|---|---|
| Read playlists | ✅ Pathfinder | ✅ InnerTube | ✅ amp-api | 🟡 GraphQL / mesk; DOM fallback |
| Read Liked / favourites | ✅ | ✅ (`LM`) | 🟡 to verify | 🟡 to verify |
| Create playlist | ✅ spclient | ✅ (beware creation gate) | ✅ | 🟡 |
| Add tracks | ✅ ~100/call, **4–7s pacing, verify** | ✅ ~100/call, bisect on bad ID | ✅ ~100/call, **verify**, bisect on 500 | 🟡 possibly 1 per call |
| ISRC available on export | 🟡 metadata endpoint (Phase 0) | ❌ | ✅ via `include=catalog` | 🟡 GraphQL only |
| Exact lookup by ISRC (as destination) | ❌ | ❌ | ✅ 25 per request | 🟡 GraphQL `searchTracks(isrc)` |
| Binding limit | silent write drops | creation gate, 5k items per playlist | ~250–280 req/hour | unknown |
| Confidence | Medium | **High** | Medium-high | **Low** |

### 7.6 Phase 0 test checklist (the unknowns that decide the plan)

- **Spotify:**
  - Does `metadata/4/track/{gid}` return ISRC as JSON from the page? This decides Spotify→Apple speed.
  - What's the maximum batch size for `addToPlaylist`?
  - Does the `spclient` create call work, and does bundle-hash discovery find the mutation hashes?
- **YouTube Music:**
  - Is the 3-hash `Authorization` required, or is the legacy single hash enough?
  - Does `edit_playlist` work with batches larger than 100?
  - Behaviour of the creation gate on a fresh test account.
- **Apple Music:**
  - What's the real in-page rate limit (requests per hour before a 429)?
  - Is a subscription required to read the library?
  - How do we read the Favorite Songs list?
  - Is the add-then-verify flow reliable?
- **Amazon Music:**
  - Which route works from the page (GraphQL called from the page, GraphQL from the extension origin, mesk, the Redux store)?
  - Can it write? Does Prime differ from Unlimited?
  - Pagination and ISRC availability.
- **All services:** record sanitized fixtures and measure throughput for a 1,000-track playlist.

## 8. UX

1. **Popup:** shows which services are detected as logged in (by checking open tabs or probing), and an "Open SYP" button.
2. **App page wizard:**
   1. *Source:* pick a service or "Import file". The playlist list loads with checkboxes, track counts and a search box.
   2. *Destination:* pick a service or "Export file" (SYP JSON / CSV). For a service, choose "create new playlists" (default, with a name template like `{name}`) or "append to existing".
   3. *Matching:* live progress (matched / review / unmatched), pause and resume, and an ETA from the request-budget scheduler. Before starting, warn when a job will be slow (for example, "about 3h because Apple Music limits requests; you can close this tab and it will continue later").
   4. *Review:* the uncertain matches (§6.3).
   5. *Write:* progress per playlist.
   6. *Report:* summary plus downloadable unmatched CSV, with a "Copy again later" pairing saved for future sync.
3. **Jobs list:** resume or discard paused jobs.
4. **Settings:** rate-limit presets (Gentle / Normal), match thresholds (advanced), clear caches, diagnostics.
   **About:** the AGPL-3.0 "Appropriate Legal Notices" (copyright, no warranty, license link) and a link to the source repository.
5. **Permissions:** host permissions are **optional and requested per service** the first time the user enables it. A fresh install shows minimal warnings, and users only grant access to the services they use. MAIN-world `document_start` scripts (the Spotify token capture, the Amazon store hook) are registered dynamically with `chrome.scripting.registerContentScripts` only after that service's permission is granted.
6. **Account-safety notice** shown once per service before the first write: SYP acts like a fast user on your own account, the services' terms discourage automation, and SYP paces requests gently to stay well within normal use.

## 9. Robustness and maintenance strategy

Internal endpoints *will* break. The plan is to make breakages cheap:
- All volatile values (endpoint paths, GraphQL hashes, client versions, header names) live in one `endpoints.ts` per service.
- Where possible, **discover volatile values at runtime from the page** instead of hard-coding them: client version from `ytcfg`, Spotify GraphQL hashes from loaded JS bundles, Apple developer token from the MusicKit instance.
- **Self-test / diagnostics:** a "Check services" button runs read-only probes per service (session, list one playlist, one search) and shows which step failed. This is the first thing to ask for in bug reports, and the issue template includes it.
- **Recorded-fixture tests** for every parser, so a changed response shape is caught by updating a fixture plus parser rather than by debugging live.
- **Manual live smoke tests** (Playwright with a persistent, logged-in local profile, never in CI) before each release.
- **Release pipeline** so a fix can reach the Chrome Web Store the same day: tag, CI build, zip, upload with `chrome-webstore-upload-cli`. Review time is outside our control, typically 1–3 days.
- No remote config that changes behaviour (store policy). Fixes ship as extension updates.

## 10. Tech stack and repo layout

- **Language:** TypeScript (strict).
- **Extension framework:** [WXT](https://wxt.dev) (Vite-based, MV3, builds for Chrome/Edge/Firefox from one codebase, HMR in dev). Pin an exact version; WXT is still 0.x and minor bumps can break.
- **UI:** Preact + a small CSS layer (no heavy component library), to keep the bundle small and the review simple. _Open decision: React if contributor familiarity matters more than size._
- **Storage:** IndexedDB (via `idb`) for jobs and the match cache, `chrome.storage.local` for settings.
- **Validation:** zod for file formats and defensive parsing of service responses.
- **Fuzzy matching:** a small in-house implementation (Jaro-Winkler + token-set) or `fastest-levenshtein`. Keep dependencies minimal.
- **Testing:** Vitest (unit, parsers, matching, golden set), Playwright (e2e on the unpacked extension: UI flows with mocked page agents in CI, live smoke tests locally).
- **Quality:** ESLint + Prettier, `tsc --noEmit`, GitHub Actions CI (lint, typecheck, test, build, zip artifacts).
- **Package manager:** npm.

```
syp/
├─ entrypoints/
│  ├─ background.ts            # service worker
│  ├─ popup/                   # launcher
│  ├─ app/                     # main UI (wizard, review, jobs, settings)
│  └─ spotify-sniffer.content.ts  # MAIN world, document_start (only where needed)
├─ src/
│  ├─ core/                    # model, normalize, matching, scoring, jobs, throttle, errors
│  ├─ io/                      # syp-json, csv, spotify-backup importer
│  ├─ rpc/                     # executeScript-based page RPC, tab management
│  └─ services/
│     ├─ types.ts              # ServiceAdapter, Capabilities
│     ├─ spotify/  { endpoints, page-agent, parse, adapter }.ts + fixtures/
│     ├─ ytmusic/  …
│     ├─ apple/    …
│     └─ amazon/   …
├─ tests/                      # unit + golden set + e2e
├─ docs/
│  ├─ services/*.md            # reverse-engineering notes per service (Phase 0 output)
│  └─ PRIVACY.md
└─ PLAN.md
```

## 11. Phased roadmap

Each phase ends with a demo-able result. Estimates assume one developer working part-time with AI assistance and are rough.

### Phase 0: Reverse-engineering spikes (1–2 weeks)
For each service, using a throwaway test playlist on Luca's own accounts (ideally secondary accounts, given the account-risk findings):
- capture auth/session material from inside the page,
- list playlists, read one playlist fully (with pagination), search, create a playlist, add tracks,
- work through the **§7.6 checklist**,
- record sanitized fixtures (strip tokens and personal data),
- write `docs/services/<service>.md` (endpoints, headers, limits, quirks).

**Go/no-go per service.** Expected order of confidence: YouTube Music > Spotify ≈ Apple Music > Amazon Music. Amazon's outcome decides read/write versus export-only.

### Phase 1: Foundation (1–2 weeks)
WXT scaffold, manifest with optional host permissions, CI, domain model, SYP JSON and CSV import/export, page-RPC layer, tab manager, job store with checkpointing, rate limiter, error types, app shell UI.

### Phase 2: MVP: Spotify ⇄ YouTube Music (2–3 weeks)
Both adapters (read and write), matching engine v1 with golden set, wizard plus review UI, export/import files. **Milestone:** copy a 1,000-track Spotify playlist to YouTube Music and back with ≥ 97% auto-match precision. Publish as an **unlisted** Chrome Web Store item for testers.

### Phase 3: Apple Music (1–2 weeks)
Adapter, batched ISRC lookup (25 per request), storefront handling and equivalents, subscription-required messaging, "always create new playlist" rule, and **request-budget scheduler hardening** for the hourly limit (multi-hour background jobs).

### Phase 4: Amazon Music (2–3 weeks, highest risk)
Read/export first. Write only if Phase 0 found a workable path; otherwise ship Amazon as **export-only** (see §7.4).

### Phase 5: Sync and polish (2+ weeks)
Saved pairings ("keep this Spotify playlist mirrored to YouTube Music"), incremental add-only sync on demand, optional removal propagation with confirmation, Liked Songs ⇄ likes, Firefox and Edge builds, i18n scaffolding.

### Phase 6: Public launch and maintenance
Public Chrome Web Store listing, Edge Add-ons, Firefox AMO, a GitHub Pages site with privacy policy, issue templates with diagnostics, and a release checklist.

## 12. Chrome Web Store compliance checklist

- **Single purpose:** "Transfer and back up your music playlists between streaming services." Everything in the extension serves that.
- **Permissions:** `storage`, `scripting`, `unlimitedStorage` (match cache; drop if not needed). **Optional** host permissions per service domain (including Amazon regional domains), requested at runtime. Each one justified in the dashboard.
- **No remote code:** all logic bundled; only data is exchanged with the services' own endpoints.
- **User data:** handled locally only, nothing sent anywhere except the user's own service accounts at the user's request. Privacy policy plus Limited Use disclosure hosted on GitHub Pages. Privacy practices form filled accordingly.
- **No impersonation:** the SYP name and icon are our own. Service names appear only as plain text (nominative use), no service logos as the app icon, and a "not affiliated" disclaimer in the listing and README.
- **Accurate listing:** list known limitations (for example, "Amazon Music: export only").
- **Account hygiene:** one-time $5 developer registration, 2FA, a dedicated publisher email.

## 13. Risks

| Risk | Likelihood | Impact | Mitigation |
|---|---|---|---|
| Internal endpoint change breaks a service | High (ongoing) | Medium | §9: isolated adapters, runtime discovery, diagnostics, fast releases |
| A service hardens auth (token attestation, bot detection) | Medium | High | Page-context execution looks identical to the web player; conservative rate limits; worst case, degrade that service to export via DOM |
| Account action against a user for automated use | Low–Medium | High | Gentle default rate limits, human-like pacing, clear warning in the UI, no bulk ops beyond the user's own data |
| Chrome Web Store rejection or takedown (for example, on a complaint from a service) | Low–Medium | High | Strict policy compliance (§12); also distribute via GitHub releases, Edge and Firefox |
| Terms of service: automated access is likely against each service's ToS | High (certain for some) | Medium | Personal-use tool acting only on the user's own data at their request; no content copying; disclaimer; respond promptly to any notice |
| Matching quality disappoints | Medium | High | Golden-set CI gate, review step, ISRC path |
| Apple/Amazon write paths unavailable | Medium | Medium | Ship read/export-only for that service and say so in the listing |
| Apple's ~250 requests/hour limit makes transfers without ISRC take hours | High | Medium | Get ISRCs at the source (Spotify metadata endpoint), batch ISRC lookups, cache, background jobs with honest ETA |
| Amazon player rewrite (React Native Web "Dragonfly" already in the bundle) | Medium | High (for Amazon) | Prefer the GraphQL route, which is separate from the player UI; keep the Amazon adapter thin; export-only fallback |
| Silent write drops (Spotify, Apple) produce incomplete playlists | High | High | Mandatory verify-after-write plus re-queue (§4.3); final report compares source and destination counts |

## 14. Open decisions for Luca

1. **UI library:** Preact (smaller) or React (more familiar). Recommendation: Preact.
2. **Firefox at launch** or after Chrome is stable? Recommendation: after Phase 2.
3. **Liked Songs semantics** on YouTube Music (likes affect recommendations). Copy into a playlist named "Liked from Spotify" by default, or like the songs? Recommendation: playlist by default, liking as an option.
4. **Monetization:** none or donations only? This affects store listing and ToS exposure. Recommendation: free and open source, donations link at most.
5. **Branding:** icon and name check ("SYP" collisions in the Chrome Web Store).
6. **Contributions: DCO or CLA?** The project is AGPL-3.0. A CLA (contributors grant the right to relicense) keeps open the option to dual-license commercially later. A DCO (sign-off only) is friendlier to contributors but makes the AGPL permanent for contributed code. Recommendation: decide before accepting the first external PR.

## Appendix A: Key sources (research of 2026-10-20)

**Platform and policy**
- Spotify: [Extended access criteria (Apr 2025)](https://developer.spotify.com/blog/2025-04-15-updating-the-criteria-for-web-api-extended-access) · [OAuth migration / implicit grant removal](https://developer.spotify.com/blog/2025-10-14-reminder-oauth-migration-27-nov-2025) · [Feb 2026 migration guide](https://developer.spotify.com/documentation/web-api/tutorials/february-2026-migration-guide) · [User Guidelines](https://www.spotify.com/us/legal/user-guidelines/)
- [Chrome Web Store program policies](https://developer.chrome.com/docs/webstore/program-policies/policies)
- [YouTube Terms](https://www.youtube.com/t/terms) · [Apple Media Services Terms](https://www.apple.com/legal/internet-services/itunes/us/terms.html) · [Amazon Music API overview (closed beta)](https://www.developer.amazon.com/docs/music/API_web_overview.html)

**Spotify internals**
- [librespot #1475](https://github.com/librespot-org/librespot/issues/1475), [#1562](https://github.com/librespot-org/librespot/discussions/1562) (TOTP history)
- [hearted.music SPOTIFY-API.md](https://github.com/fabiogaliano/hearted.music/blob/main/extensions/SPOTIFY-API.md) · [chorus](https://github.com/chorus-studio/chorus) · [Segue](https://github.com/SysAdminDoc/Segue) · [BitChord](https://github.com/kushagrasinghx/BitChord)
- [folia-major spotifyApiBridge](https://github.com/chthollyphile/folia-major/blob/main/electron/spotify/spotifyApiBridge.cjs) (create plus add) · [songmirror cookie write backend](https://github.com/ahnafnafee/songmirror/blob/main/docs/design/2026-07-20-spotify-cookie-write-backend.md) · [spotify-playlist-driver](https://github.com/jbaruch/spotify-playlist-driver) (write pacing)

**YouTube Music internals**
- [ytmusicapi](https://github.com/sigma67/ytmusicapi) (request and response shapes; issues [#813](https://github.com/sigma67/ytmusicapi/issues/813), [#962](https://github.com/sigma67/ytmusicapi/issues/962), PR [#963](https://github.com/sigma67/ytmusicapi/pull/963))
- [yt-dlp `_base.py`](https://github.com/yt-dlp/yt-dlp/blob/master/yt_dlp/extractor/youtube/_base.py), PR [#11989](https://github.com/yt-dlp/yt-dlp/pull/11989) (3-hash auth) · [PlaylistPlus userscript](https://greasyfork.org/scripts/574930-playlistplus/code)

**Apple Music internals**
- [applemusic-mcp](https://github.com/epheterson/applemusic-mcp) (in-page MusicKit tokens, live-tested writes) · [gamdl](https://github.com/glomatico/gamdl)
- Apple docs: [create library playlist](https://developer.apple.com/documentation/applemusicapi/create-a-new-library-playlist) · [add tracks](https://developer.apple.com/documentation/applemusicapi/add-tracks-to-a-library-playlist) · [songs by ISRC](https://developer.apple.com/documentation/applemusicapi/get-multiple-catalog-songs-by-isrc)
- Forums: [702228](https://developer.apple.com/forums/thread/702228), [813068](https://developer.apple.com/forums/thread/813068), [805461](https://developer.apple.com/forums/thread/805461)

**Amazon Music internals**
- [songmirror `amazon_music_web.py`](https://github.com/ahnafnafee/songmirror) (GraphQL route) · [lib-origin](https://github.com/Illusion137/lib-origin) (mesk endpoints) · [SynQApp/Extension](https://github.com/SynQApp/Extension) (Redux hook) · [Mornye-Extension](https://github.com/mornyeapp/Mornye-Extension) · [save_amazon_music_playlist](https://github.com/projecteurlumiere/save_amazon_music_playlist) (DOM export)
