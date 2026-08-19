# PoToken support for full-length adaptive downloads

**Date:** 2026-08-18
**Status:** SUPERSEDED — the PoToken route was tested and does not work. See
[Outcome](#outcome-route-b-is-closed) at the end of this document. The problem statement
and measurements below remain accurate and are worth keeping; the proposed solution is not.
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

---

## Outcome: Route B is closed

Three feasibility spikes were run before writing production code. All three failed to lift
the ceiling. BotGuard minting itself worked every time — the tokens were valid, well-formed,
and correctly attached. YouTube simply does not honour them for these streaming URLs.

### Spike 1 — WEB token on an IOS/ANDROID URL

Minted a 116-character content-bound token and appended `&pot=` to an existing IOS/ANDROID
streaming URL.

| Probe | Without pot | With pot |
| --- | --- | --- |
| offset 0 (32 KB) | 206 | 206 |
| offset 80% (8.2 MB) | 403 | **403** |

The offset-0 control proves the URL stayed intact and the token was genuinely attached.
Verdict: FAIL.

### Spike 2 — everything inside one WEB session

Hypothesis: the token must match the client and session that issued the URL. This spike
extracted `INNERTUBE_CLIENT_VERSION`, `VISITOR_DATA` and the signature timestamp from the
live watch page, minted a session token bound to that `visitorData`, and sent a WEB player
request carrying it.

The hypothesis was never testable. **The plain WEB client no longer returns per-format
streaming URLs at all.** All 40 `adaptiveFormats` came back with no `url`, no
`signatureCipher`, and no `cipher`; `streamingData` carried only `serverAbrStreamingUrl`
(SABR/UMP). Control checks confirmed this is unconditional server behaviour — identical
response shape with no `visitorData` and no token, and already present in the raw watch
page HTML.

This rules out the WEB client for this library's architecture, which depends on HTTP Range
requests against direct format URLs. It also explains retroactively why `lib/info.js` never
used the plain WEB client. Verdict: INCONCLUSIVE, hypothesis untestable by this route.

### Spike 3 — IOS/ANDROID player request carrying a session token

The remaining untested variant: rather than patching a token onto a URL that was already
issued restricted, send the token *with* the player request so the URL is minted
authenticated. Both clients, three token variants, with offset-0 controls.

| Client | itag | offset 0 (a/b/c) | offset 80% (a/b/c) |
| --- | --- | --- | --- |
| IOS | 251 | 206 / 206 / 206 | 403 / 403 / 403 |
| ANDROID | 258 | 206 / 206 / 206 | 403 / 403 / 403 |

(a) untouched, (b) session token bound to `visitorData`, (c) content token bound to the
video id. Verdict: FAIL.

### Conclusion

`bgutils-js` mints **WebPO** tokens. The clients that still return usable per-format URLs
in this fork are IOS and ANDROID, which do not accept them, and the WEB client that would
accept them no longer returns per-format URLs. The two halves cannot be joined with the
tools available.

The implementation plan at `docs/superpowers/plans/2026-08-18-potoken-support.md` was
halted at its Task 1 gate and Tasks 2-7 were never executed. `bgutils-js` and `jsdom` were
removed from `devDependencies` after the spikes.

### What remains viable

**Route A: ANDROID_VR with cookies.** Per the yt-dlp PO Token guide, `android_vr` requires
no GVS token for streaming. It is blocked here only at the player stage, with
`LOGIN_REQUIRED` / "Sign in to confirm you're not a bot" — a bot check that authenticated
cookies normally lift. This is untested because it needs real credentials. It remains the
most promising path and costs no new dependencies.

**Known-good today:** progressive formats (itag 18) download completely, and any video
under ~60 seconds downloads completely in any format.

### An operational note for the future

Getting the BotGuard minter working in Node required a fix worth recording: the challenge
interpreter must be evaluated via indirect `eval` into Node's own realm, **not** through
`jsdomWindow.eval`. Running it inside the jsdom realm makes `WebPoMinter.create` fail
cross-realm `instanceof` checks with `APF:Failed`. Anyone revisiting this will hit it.
