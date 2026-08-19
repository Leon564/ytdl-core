# PoToken support for full-length adaptive downloads

**Date:** 2026-08-18
**Status:** Approved, pending implementation
**Branch base:** `fix/update-innertube-client-versions`

## Problem

Adaptive formats (audio-only and video-only) stop serving bytes after roughly 60
seconds of media. Any byte offset past that window returns HTTP 403.

This was verified against the live API, not inferred. On a freshly fetched URL the
*first* request at offset 9.5 MB already returns 403 while offset 0 returns 206, which
rules out rate limiting. Measured accessible windows on `aqz-KE-bpKQ` (635 s):

| itag | kind | contentLength | reachable | approx media |
| --- | --- | --- | --- | --- |
| 251 | audio 160k | 10,258,925 | 0.96 MB (8.9%) | ~60 s |
| 315 | video 2160p60 | 1,362,269,481 | 122.93 MB (9.5%) | ~60 s |
| 18 | progressive 360p | 28,523,658 | 100% | full |

The cause is that no format URL carries a `pot` (Proof of Origin Token) parameter —
0 of 72 formats on that video. Progressive itag 18 is exempt and downloads completely,
which is why some downloads still work today.

Practical impact: `ytdl(url, { filter: "audioonly" })` fails with `Status code: 403`.
The downstream consumer of this fork (`E:\Dev\chat\bot`) requests exactly that, so it
is broken until this is fixed.

## Goal

A complete adaptive-format download finishes with `bytes === contentLength`.

## Non-goals

- Changing the default player clients. The one exception is the contingency in Risks: if
  the feasibility spike shows IOS/ANDROID do not honour a GVS token, switching the
  default to ANDROID_VR comes back into scope and this non-goal is withdrawn.
- Rewriting the chunked-download logic. An earlier attempt to size chunks by bitrate
  was reverted: the ceiling is absolute, not per-request, so chunking cannot fix it.
- Supporting age-restricted or members-only videos.

## Background: which clients need a token

Per the yt-dlp PO Token guide, GVS (streaming) tokens are required by `web`, `mweb`,
`ios`, `android`, and others, and are **not** required by `android_vr` or `web_embedded`.
Most tokens are bound to the video ID.

This fork currently gets its formats from IOS and ANDROID — both of which require a
token. ANDROID_VR would need none, but its player request is refused on unauthenticated
IPs with `LOGIN_REQUIRED` / "Sign in to confirm you're not a bot", regardless of client
version (tested 1.57.29, 1.61.48, 1.62.27, 1.68.53).

So there are two complementary routes, and the design supports both:

- **Route A (no token):** unblock ANDROID_VR with cookies via the existing
  `createAgent`. Its streaming URLs then need no `pot`. Zero new code — documentation only.
- **Route B (token):** mint a content-bound token and append `&pot=` to IOS/ANDROID URLs.

## Token flow

Confirmed from the BgUtils reference example:

1. Set up a DOM (`window`, `document`, `location`, `origin`, `navigator`) via jsdom.
2. `getChallenge({ fetchFunction, requestKey })` with request key `O43z0dpjhgX20SCx4KAo`.
3. Evaluate `challenge.interpreterJavascript.privateDoNotAccessOrElseSafeScriptWrappedValue`.
4. `BotGuardClient.create({ program, globalName, globalObject })`.
5. `botGuardClient.snapshot({ webPoSignalOutput })`.
6. `POST` the `[requestKey, botguardResponse]` payload to the WAA `GenerateIT` endpoint,
   yielding `[integrityToken, estimatedTtlSecs, mintRefreshThreshold, websafeFallbackToken]`.
7. `WebPoMinter.create(integrityTokenData, webPoSignalOutput)`.
8. `minter.mintAsWebsafeString(contentBinding)` returns the token.

Two bindings matter:

- **Session token** — bound to `visitorData`. Sent in the player request as
  `serviceIntegrityDimensions.poToken`. Lifts bot checks.
- **Content token** — bound to the video ID. Appended to the streaming URL as `&pot=`.
  Must not be cached long-term; YouTube mints a new one per video.

## Architecture

Three pieces, deliberately separated so the fragile part is isolated and optional.

### 1. Core wiring — `lib/potoken.js` (no new dependencies)

Resolves a token from whichever source is configured, in priority order:

1. `options.poToken` + `options.visitorData` — supplied by the caller.
2. `options.poTokenProvider` — an async function
   `({ videoId, visitorData, client }) => { poToken, visitorData }`.
3. The bundled generator, if enabled and its optional dependencies are installed.
4. Nothing — current behavior, no `pot` appended, 60 s ceiling stands.

Responsibilities:

- Cache session tokens keyed by `visitorData` with a TTL derived from
  `estimatedTtlSecs`; cache content tokens per video ID for the life of the request only.
- Never throw on failure. A token that cannot be minted degrades to today's behavior
  with a warning, because a partial download beats a hard crash for callers that only
  need the first minute.

### 2. Integration points

- `lib/sig.js` — `setDownloadURL` appends `&pot=<encoded>` when a GVS token is present,
  alongside the existing signature and `n` handling.
- `lib/info.js` — player payloads gain `serviceIntegrityDimensions.poToken` when a
  session token is present. Applies to the IOS, ANDROID, and ANDROID_VR fetchers.

### 3. Bundled generator — `lib/potoken-generator.js`, exposed as a subpath export

Implements the eight-step flow above. Loaded **lazily**: the core requires it only when
the generator is actually selected, so callers who supply their own token never load
jsdom.

- Dependencies: `bgutils-js@^4` (zero transitive deps) and `jsdom@^29` as
  `optionalDependencies`.
- **jsdom must be pinned to `^29`.** jsdom 30 requires Node `^22.22.2 || ^24.15.0 || >=26`,
  which breaks this package's declared `node >=20.18.1` engine and the current dev
  runtime (Node 20.19.4). jsdom 29.1.1 supports `^20.19.0`.
- If either dependency is missing, throw one clear error naming both packages and
  pointing at the manual `poToken` option.

## Testing

The existing scratchpad suite becomes `test/` in the repo, run with `npm test`.

**Acceptance criterion.** A new test downloads an adaptive audio-only format end to end
and asserts `bytes === parseInt(format.contentLength)`. This test must fail before the
change and pass after it. Today it fails at ~60 s with a 403.

Supporting coverage:

- `filter: "audioonly"` through the public `ytdl()` API completes — the downstream bot's
  exact call.
- Format URLs carry a `pot` parameter once a provider is configured.
- With no token configured, behavior is unchanged and nothing throws.
- Token caching: two downloads of the same video mint the session token once.

Network-dependent tests are marked so they can be skipped in CI.

## Risks

- **BotGuard changes frequently.** The `requestKey` and challenge format may rotate. This
  is why the generator is isolated behind a provider interface — swapping it out or
  pointing at an external provider must not touch core code.
- **jsdom is heavy** (~20 transitive dependencies) and version-sensitive against Node 20.
  Optional and lazily loaded for exactly this reason.
- **The token may not lift the ceiling for IOS/ANDROID specifically.** The guide says
  those clients accept GVS tokens, but this has not been verified in this environment.
  Step 1 of implementation is a feasibility spike that mints a real token and confirms
  `&pot=` lifts the 60 s cap **before** any production code is written. If it does not,
  the design changes to prefer ANDROID_VR with a session token, and the spike is what
  tells us so.

## Open question

Whether cookies alone unblock ANDROID_VR cannot be tested without real credentials. If
they do, Route A is a zero-dependency path worth documenting prominently.
