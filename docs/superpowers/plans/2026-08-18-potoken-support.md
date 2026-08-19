# PoToken Support Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Make a complete adaptive-format download finish with `bytes === contentLength` instead of failing with HTTP 403 after ~60 seconds of media.

**Architecture:** A dependency-free core module resolves a Proof of Origin Token from one of three sources (caller-supplied, caller-provided async function, or a bundled generator), and the token is appended to streaming URLs as `&pot=` and injected into player payloads as `serviceIntegrityDimensions.poToken`. The BotGuard attestation that mints tokens is isolated in its own lazily loaded module so its heavy, fragile dependencies stay optional.

**Tech Stack:** Node.js (CommonJS), `undici`, `node:test` for tests, `bgutils-js@^4` and `jsdom@^29` as optional dependencies.

**Spec:** `docs/superpowers/specs/2026-08-18-potoken-support-design.md`

## Global Constraints

- **jsdom must be `^29`, never `^30`.** jsdom 30 requires Node `^22.22.2 || ^24.15.0 || >=26.0.0`. This package declares `node >=20.18.1` and the dev runtime is Node 20.19.4. jsdom 29.1.1 supports `^20.19.0`.
- `bgutils-js@^4` — has zero transitive dependencies.
- Both go in `optionalDependencies`, never `dependencies`. The core must install and run with neither present.
- The generator module is loaded with a lazy `require` inside a function, never at module top level.
- Token resolution **never throws**. On any failure it logs a warning and returns `null`, preserving today's behavior.
- Content tokens are bound to the video ID and must not be cached across videos. Session tokens are bound to `visitorData` and may be cached with a TTL.
- The project is CommonJS. `bgutils-js` ships ESM, so it must be pulled in with `await import(...)`, not `require(...)`.
- The codebase uses 2-space indent, double quotes, and semicolons (see `.prettierrc.json`). Run `npm run prettier` before committing.

---

### Task 1: Feasibility spike — prove `&pot=` lifts the 60s ceiling

This task is a **gate**. It writes no production code. Its only output is a decision and a captured token. If it fails, stop and report; the rest of the plan changes.

**Files:**
- Create: `scripts/spike-potoken.js` (throwaway, deleted in Step 6)

**Interfaces:**
- Consumes: nothing.
- Produces: a decision recorded in the task's commit message. Downstream tasks assume IOS/ANDROID streaming URLs honour a `pot` parameter.

- [ ] **Step 1: Install the optional dependencies locally**

```bash
pnpm add -D bgutils-js@^4 jsdom@^29
```

Note: installed as dev dependencies for the spike only. Task 6 moves them to `optionalDependencies`.

- [ ] **Step 2: Write the spike script**

Create `scripts/spike-potoken.js`:

```js
// Throwaway feasibility spike. Deleted at the end of Task 1.
// Question: does appending &pot=<token> to an IOS/ANDROID streaming URL
// lift the ~60s ceiling that currently makes adaptive downloads 403?
const ytdl = require("../lib/index.js");
const { request } = require("undici");

const REQUEST_KEY = "O43z0dpjhgX20SCx4KAo";
const VIDEO_ID = "aqz-KE-bpKQ";

const mintToken = async contentBinding => {
  const { JSDOM } = await import("jsdom");
  const { BotGuardClient, getChallenge } = await import("bgutils-js/botguard");
  const { buildURL, getHeaders, USER_AGENT } = await import("bgutils-js/utils");
  const { WebPoMinter } = await import("bgutils-js/webpo");

  const dom = new JSDOM('<!DOCTYPE html><html lang="en"><head><title></title></head><body></body></html>', {
    url: "https://www.youtube.com/",
    referrer: "https://www.youtube.com/",
    userAgent: USER_AGENT,
  });

  Object.assign(globalThis, {
    window: dom.window,
    document: dom.window.document,
    location: dom.window.location,
    origin: dom.window.origin,
  });
  if (!Reflect.has(globalThis, "navigator")) {
    Object.defineProperty(globalThis, "navigator", { value: dom.window.navigator });
  }

  const challenge = await getChallenge({ fetchFunction: fetch, requestKey: REQUEST_KEY });
  const interpreter = challenge.interpreterJavascript?.privateDoNotAccessOrElseSafeScriptWrappedValue;
  if (!interpreter) throw new Error("Interpreter javascript not available");
  new Function(interpreter)();

  const botGuardClient = await BotGuardClient.create({
    program: challenge.program,
    globalName: challenge.globalName,
    globalObject: globalThis,
  });

  const webPoSignalOutput = [];
  const botguardResponse = await botGuardClient.snapshot({ webPoSignalOutput });

  const res = await fetch(buildURL("GenerateIT", true), {
    method: "POST",
    headers: getHeaders(),
    body: JSON.stringify([REQUEST_KEY, botguardResponse]),
  });
  const [integrityToken, estimatedTtlSecs, mintRefreshThreshold, websafeFallbackToken] = await res.json();

  const minter = await WebPoMinter.create(
    { integrityToken, estimatedTtlSecs, mintRefreshThreshold, websafeFallbackToken },
    webPoSignalOutput,
  );
  return minter.mintAsWebsafeString(contentBinding);
};

const probe = async (url, start, end) => {
  const res = await request(url, { method: "GET", headers: { Range: `bytes=${start}-${end}` } });
  let got = 0;
  try {
    for await (const c of res.body) {
      got += c.length;
      if (got > 32768) break;
    }
  } catch {}
  res.body.destroy();
  return res.statusCode;
};

(async () => {
  const token = await mintToken(VIDEO_ID);
  console.log(`minted token (${token.length} chars): ${token.slice(0, 24)}...`);

  const info = await ytdl.getInfo(VIDEO_ID);
  const format = ytdl.chooseFormat(info.formats, { quality: "highestaudio" });
  const clen = parseInt(format.contentLength);
  const withPot = new URL(format.url);
  withPot.searchParams.set("pot", token);

  // Offset well past the ~60s window that currently 403s.
  const deepStart = Math.floor(clen * 0.8);
  const deepEnd = deepStart + 32767;

  const before = await probe(format.url, deepStart, deepEnd);
  const after = await probe(withPot.toString(), deepStart, deepEnd);

  console.log(`itag ${format.itag}, contentLength=${clen}, probing bytes=${deepStart}-${deepEnd}`);
  console.log(`  without pot: HTTP ${before}`);
  console.log(`  with pot:    HTTP ${after}`);
  console.log(after === 206 ? "\nVERDICT: PASS — pot lifts the ceiling." : "\nVERDICT: FAIL — pot does not help.");

  require("fs").writeFileSync(
    require("path").join(require("os").tmpdir(), "ytdl-spike-token.json"),
    JSON.stringify({ videoId: VIDEO_ID, token }, null, 2),
  );
})();
```

- [ ] **Step 3: Run the spike**

```bash
node scripts/spike-potoken.js
```

Expected on success: `without pot: HTTP 403`, `with pot: HTTP 206`, and `VERDICT: PASS`.

- [ ] **Step 4: Handle a FAIL verdict**

If the verdict is FAIL, **stop the plan here**. Report to the user which of these it was:
- Minting itself threw — BotGuard flow is broken or the request key rotated.
- Token minted but `with pot` is still 403 — IOS/ANDROID do not honour GVS tokens here. Per the spec's Risks section, the design switches to ANDROID_VR with a session token and this plan must be rewritten.

Do not proceed to Task 2 on a FAIL.

- [ ] **Step 5: Save the token for later tasks**

The script already wrote it to `<tmpdir>/ytdl-spike-token.json`. Tasks 3 and 4 use this real token to test the plumbing without needing the generator. Note that PoTokens expire — re-run the spike script if later tasks see 403s with a token that previously worked.

- [ ] **Step 6: Delete the spike and commit the finding**

```bash
rm scripts/spike-potoken.js
git add -A
git commit -m "chore: verify pot parameter lifts the adaptive format ceiling

Feasibility spike (not retained): minted a content-bound PoToken via
BotGuard and confirmed that appending &pot= to an IOS/ANDROID streaming
URL turns a 403 at 80% depth into a 206."
```

---

### Task 2: Move the test suite into the repo with a failing acceptance test

**Files:**
- Create: `test/helpers.js`
- Create: `test/unit/url-utils.test.js`
- Create: `test/integration/download.test.js`
- Modify: `package.json` (add `test` and `test:unit` scripts)

**Interfaces:**
- Consumes: nothing.
- Produces: `test/helpers.js` exporting `networkTest(name, fn)` — a wrapper that skips the test unless `YTDL_TEST_NETWORK=1` is set. Later tasks use it for every network-dependent test.

- [ ] **Step 1: Write the test helper**

Create `test/helpers.js`:

```js
const test = require("node:test");

// Network tests hit the live YouTube API and are skipped unless opted into,
// so CI and offline runs stay green.
const NETWORK_ENABLED = process.env.YTDL_TEST_NETWORK === "1";

exports.networkTest = (name, fn) =>
  test(name, { skip: NETWORK_ENABLED ? false : "set YTDL_TEST_NETWORK=1 to run" }, fn);

// Downloads a stream to completion, resolving with the total byte count.
exports.collect = stream =>
  new Promise((resolve, reject) => {
    let bytes = 0;
    stream.on("data", chunk => {
      bytes += chunk.length;
    });
    stream.on("end", () => resolve(bytes));
    stream.on("error", reject);
  });
```

- [ ] **Step 2: Write the offline unit test**

Create `test/unit/url-utils.test.js`:

```js
const test = require("node:test");
const assert = require("node:assert");
const ytdl = require("../../lib/index.js");

test("getVideoID parses every supported URL shape", () => {
  assert.strictEqual(ytdl.getVideoID("https://www.youtube.com/watch?v=aqz-KE-bpKQ"), "aqz-KE-bpKQ");
  assert.strictEqual(ytdl.getVideoID("https://youtu.be/aqz-KE-bpKQ"), "aqz-KE-bpKQ");
  assert.strictEqual(ytdl.getVideoID("https://www.youtube.com/shorts/aqz-KE-bpKQ"), "aqz-KE-bpKQ");
});

test("validateURL rejects non-YouTube hosts", () => {
  assert.strictEqual(ytdl.validateURL("https://example.com/watch?v=aqz-KE-bpKQ"), false);
});
```

- [ ] **Step 3: Write the failing acceptance test**

Create `test/integration/download.test.js`. `aqz-KE-bpKQ` is 635 seconds long, so it is well past the ~60s ceiling — a short video would pass even while the bug exists.

```js
const assert = require("node:assert");
const ytdl = require("../../lib/index.js");
const { networkTest, collect } = require("../helpers");

const LONG_VIDEO = "aqz-KE-bpKQ"; // 635s — must exceed the ~60s ceiling

networkTest("downloads an adaptive audio-only format to completion", async () => {
  const info = await ytdl.getInfo(LONG_VIDEO);
  const format = ytdl.chooseFormat(info.formats, { quality: "highestaudio" });
  const expected = parseInt(format.contentLength);

  assert.ok(expected > 0, "format must declare a contentLength");

  const bytes = await collect(ytdl.downloadFromInfo(info, { quality: "highestaudio" }));
  assert.strictEqual(bytes, expected, `downloaded ${bytes} of ${expected} bytes`);
});

networkTest("downloads via the public audioonly filter", async () => {
  const stream = ytdl(LONG_VIDEO, { filter: "audioonly", quality: "highestaudio" });
  const bytes = await collect(stream);
  assert.ok(bytes > 2 * 1024 * 1024, `expected more than 2MB, got ${bytes}`);
});
```

- [ ] **Step 4: Add the test scripts**

In `package.json`, add to `scripts`:

```json
"test": "node --test test/unit/ test/integration/",
"test:unit": "node --test test/unit/"
```

- [ ] **Step 5: Run the unit tests to verify they pass**

```bash
npm run test:unit
```

Expected: PASS, 2 tests.

- [ ] **Step 6: Run the acceptance test to verify it FAILS**

```bash
YTDL_TEST_NETWORK=1 npm test
```

Expected: the two integration tests FAIL with `Status code: 403`. This is the bug being fixed — confirm the failure is a 403 and not a setup error before continuing.

- [ ] **Step 7: Commit**

```bash
git add test/ package.json
git commit -m "test: add suite with failing adaptive-download acceptance test

The two integration tests fail with HTTP 403 today: adaptive formats stop
serving bytes past ~60s of media. They are the acceptance criterion for
PoToken support."
```

---

### Task 3: Core token resolution module

**Files:**
- Create: `lib/potoken.js`
- Create: `test/unit/potoken.test.js`

**Interfaces:**
- Consumes: nothing.
- Produces:
  - `resolve(videoId, options) => Promise<{ poToken: string, visitorData?: string } | null>`
  - `applyToUrl(url: string, poToken?: string) => string`
  - `selectProvider(options) => Function | null`

**Note on caching.** The spec calls for caching so repeated downloads do not redo the
attestation. That caching lives in the generator (Task 6), which builds the expensive
BotGuard minter once per process. This module deliberately caches nothing: content tokens
are bound to a video id and must not be reused across videos.

- [ ] **Step 1: Write the failing tests**

Create `test/unit/potoken.test.js`:

```js
const test = require("node:test");
const assert = require("node:assert");
const potoken = require("../../lib/potoken.js");

test("applyToUrl appends the pot parameter", () => {
  const out = potoken.applyToUrl("https://example.com/videoplayback?itag=251", "TOKEN123");
  assert.strictEqual(new URL(out).searchParams.get("pot"), "TOKEN123");
});

test("applyToUrl leaves the url untouched without a token", () => {
  const url = "https://example.com/videoplayback?itag=251";
  assert.strictEqual(potoken.applyToUrl(url, undefined), url);
});

test("applyToUrl returns the url unchanged when it cannot be parsed", () => {
  assert.strictEqual(potoken.applyToUrl("not a url", "TOKEN123"), "not a url");
});

test("resolve prefers a caller-supplied token", async () => {
  const result = await potoken.resolve("VIDEO", { poToken: "MANUAL", visitorData: "VD" });
  assert.deepStrictEqual(result, { poToken: "MANUAL", visitorData: "VD" });
});

test("resolve calls a provider with the video id", async () => {
  const seen = [];
  const result = await potoken.resolve("VIDEO", {
    poTokenProvider: async arg => {
      seen.push(arg);
      return { poToken: "FROM_PROVIDER" };
    },
  });
  assert.strictEqual(result.poToken, "FROM_PROVIDER");
  assert.strictEqual(seen[0].videoId, "VIDEO");
});

test("resolve returns null when nothing is configured", async () => {
  assert.strictEqual(await potoken.resolve("VIDEO", {}), null);
});

test("resolve swallows provider errors and returns null", async () => {
  const result = await potoken.resolve("VIDEO", {
    poTokenProvider: async () => {
      throw new Error("boom");
    },
  });
  assert.strictEqual(result, null);
});

test("resolve returns null when a provider yields no token", async () => {
  const result = await potoken.resolve("VIDEO", { poTokenProvider: async () => ({}) });
  assert.strictEqual(result, null);
});
```

- [ ] **Step 2: Run the tests to verify they fail**

```bash
node --test test/unit/potoken.test.js
```

Expected: FAIL — `Cannot find module '../../lib/potoken.js'`.

- [ ] **Step 3: Write the implementation**

Create `lib/potoken.js`:

```js
// Content tokens are bound to a video id, so nothing is cached at this layer.
// The expensive BotGuard attestation is cached inside lib/potoken-generator.js.

/**
 * Picks the token source, in priority order: an explicit provider function,
 * then the bundled generator when the caller opted into it.
 *
 * @param {Object} options
 * @returns {Function|null}
 */
exports.selectProvider = options => {
  if (typeof options.poTokenProvider === "function") return options.poTokenProvider;
  // Lazily required so jsdom is never loaded unless the generator is actually used.
  if (options.generatePoToken) return require("./potoken-generator").generate;
  return null;
};

/**
 * Resolves a Proof of Origin Token for a video. Never throws: without a token
 * the caller keeps today's behaviour, where only the first ~60s of an adaptive
 * format is reachable.
 *
 * @param {string} videoId
 * @param {!Object} options
 * @returns {Promise<{poToken: string, visitorData: (string|undefined)}|null>}
 */
exports.resolve = async (videoId, options = {}) => {
  if (options.poToken) {
    return { poToken: options.poToken, visitorData: options.visitorData };
  }

  const provider = exports.selectProvider(options);
  if (!provider) return null;

  try {
    const result = await provider({
      videoId,
      visitorData: options.visitorData,
    });
    if (!result || !result.poToken) return null;
    return result;
  } catch (err) {
    console.warn(`Failed to obtain a PoToken, falling back to restricted formats: ${err.message}`);
    return null;
  }
};

/**
 * Appends the token to a streaming URL. YouTube rejects requests past ~60s of
 * media without it.
 *
 * @param {string} url
 * @param {string} [poToken]
 * @returns {string}
 */
exports.applyToUrl = (url, poToken) => {
  if (!poToken) return url;
  try {
    const parsed = new URL(url);
    parsed.searchParams.set("pot", poToken);
    return parsed.toString();
  } catch {
    return url;
  }
};
```

- [ ] **Step 4: Run the tests to verify they pass**

```bash
node --test test/unit/potoken.test.js
```

Expected: PASS, 8 tests.

- [ ] **Step 5: Commit**

```bash
git add lib/potoken.js test/unit/potoken.test.js
git commit -m "feat: add PoToken resolution module

Resolves a token from a caller-supplied value, a provider function, or the
bundled generator. Never throws — a missing token degrades to the current
restricted-format behaviour."
```

---

### Task 4: Append the token to streaming URLs

**Files:**
- Modify: `lib/sig.js:59` (`setDownloadURL` signature) and `lib/sig.js:110-116` (`decipherFormats`)
- Modify: `lib/info.js:267` and `lib/info.js:279` (pass the video id through)
- Test: `test/unit/sig.test.js` (create), `test/integration/download.test.js` (existing)

**Interfaces:**
- Consumes: `potoken.applyToUrl(url, poToken)` from Task 3.
- Produces: `sig.decipherFormats(formats, html5player, options, poToken)` — a fourth optional parameter. `sig.setDownloadURL(format, decipherScript, nTransformScript, poToken)` — likewise.

- [ ] **Step 1: Write the failing test**

Create `test/unit/sig.test.js`:

```js
const test = require("node:test");
const assert = require("node:assert");
const sig = require("../../lib/sig.js");

test("setDownloadURL appends pot to an unciphered format url", () => {
  const format = { url: "https://r1.googlevideo.com/videoplayback?itag=251" };
  sig.setDownloadURL(format, null, null, "TOKEN123");
  assert.strictEqual(new URL(format.url).searchParams.get("pot"), "TOKEN123");
});

test("setDownloadURL leaves the url alone when no token is given", () => {
  const format = { url: "https://r1.googlevideo.com/videoplayback?itag=251" };
  sig.setDownloadURL(format, null, null, undefined);
  assert.strictEqual(new URL(format.url).searchParams.has("pot"), false);
});
```

- [ ] **Step 2: Run it to verify it fails**

```bash
node --test test/unit/sig.test.js
```

Expected: FAIL — the first test finds no `pot` parameter.

- [ ] **Step 3: Thread the token through `sig.js`**

In `lib/sig.js`, change the `setDownloadURL` signature at line 59 from:

```js
exports.setDownloadURL = (format, decipherScript, nTransformScript) => {
```

to:

```js
exports.setDownloadURL = (format, decipherScript, nTransformScript, poToken) => {
```

Then, still inside `setDownloadURL`, replace the assignment:

```js
      format.url = urlObj.toString();
```

with:

```js
      format.url = potoken.applyToUrl(urlObj.toString(), poToken);
```

and replace the fallback branch:

```js
    } else {
      format.url = rawUrl;
    }
```

with:

```js
    } else {
      format.url = potoken.applyToUrl(rawUrl, poToken);
    }
```

Add the require at the top of `lib/sig.js`, next to the existing requires:

```js
const potoken = require("./potoken");
```

- [ ] **Step 4: Thread the token through `decipherFormats`**

In `lib/sig.js`, change line 110 from:

```js
exports.decipherFormats = async (formats, html5player, options) => {
```

to:

```js
exports.decipherFormats = async (formats, html5player, options, poToken) => {
```

and line 116 from:

```js
      exports.setDownloadURL(format, decipherScript, nTransformScript);
```

to:

```js
      exports.setDownloadURL(format, decipherScript, nTransformScript, poToken);
```

- [ ] **Step 5: Resolve the token once per `getInfo` call**

In `lib/info.js`, add the require near the other `lib` requires at the top:

```js
const potoken = require("./potoken");
```

Inside `exports.getInfo`, immediately after the line `const formatPromises = [];`, insert:

```js
  // One content-bound token per video: it is bound to the video id, so it is
  // resolved once here and reused for every format of this video. Writing it
  // back onto options follows the existing pattern (see options.visitorId) and
  // is what lets the player fetchers below pick it up in Task 5.
  const tokenResult = await potoken.resolve(id, options);
  const poToken = tokenResult?.poToken;
  if (poToken) {
    options.poToken = poToken;
    if (tokenResult.visitorData) options.visitorData = tokenResult.visitorData;
  }
```

This must sit **above** the `clientPromises` block, so the player fetchers see
`options.poToken` before they build their payloads. `const formatPromises = [];` is the
correct insertion point for that ordering.

Then update **both** `decipherFormats` call sites to pass it. Line 267 becomes:

```js
          formatPromises.push(sig.decipherFormats(formats, info.html5player, options, poToken));
```

and line 279 (inside the `catch` block) becomes:

```js
      formatPromises.push(sig.decipherFormats(formats, info.html5player, options, poToken));
```

- [ ] **Step 6: Run the unit tests to verify they pass**

```bash
npm run test:unit
```

Expected: PASS, all tests including the 2 new `sig` tests.

- [ ] **Step 7: Verify against the live API with a real token**

Using the token captured in Task 1 Step 5:

```bash
YTDL_TEST_NETWORK=1 node -e '
const fs = require("fs"), os = require("os"), path = require("path");
const { token, videoId } = JSON.parse(fs.readFileSync(path.join(os.tmpdir(), "ytdl-spike-token.json")));
const ytdl = require("./lib/index.js");
ytdl.getInfo(videoId, { poToken: token }).then(async info => {
  const f = ytdl.chooseFormat(info.formats, { quality: "highestaudio" });
  console.log("pot present:", new URL(f.url).searchParams.has("pot"));
  let bytes = 0;
  const s = ytdl.downloadFromInfo(info, { quality: "highestaudio", poToken: token });
  s.on("data", c => { bytes += c.length; });
  s.on("end", () => console.log(`downloaded ${bytes} of ${f.contentLength}`, bytes === parseInt(f.contentLength) ? "COMPLETE" : "INCOMPLETE"));
  s.on("error", e => console.log("ERROR:", e.message));
});'
```

Expected: `pot present: true` and `COMPLETE`. If the token has expired, re-run the Task 1 spike script to mint a fresh one.

- [ ] **Step 8: Commit**

```bash
npm run prettier
git add lib/sig.js lib/info.js test/unit/sig.test.js
git commit -m "feat: append PoToken to streaming urls

Threads a content-bound token through decipherFormats into setDownloadURL,
resolved once per getInfo call since the token is bound to the video id."
```

---

### Task 5: Send the session token in player requests

**Files:**
- Modify: `lib/info.js:395` (`fetchAndroidVRPlayer`), `lib/info.js:495` (`fetchIosJsonPlayer`), `lib/info.js:560` (`fetchAndroidJsonPlayer`)
- Test: `test/unit/potoken.test.js` (extend)

**Interfaces:**
- Consumes: `options.poToken`, written back onto the options object by Task 4 Step 5 before the player fetchers run.
- Produces: `buildServiceIntegrityDimensions(poToken) => Object` exported from `lib/potoken.js`, returning `{ serviceIntegrityDimensions: { poToken } }` or `{}`.

- [ ] **Step 1: Write the failing test**

Append to `test/unit/potoken.test.js`:

```js
test("buildServiceIntegrityDimensions wraps a token", () => {
  assert.deepStrictEqual(potoken.buildServiceIntegrityDimensions("TOKEN123"), {
    serviceIntegrityDimensions: { poToken: "TOKEN123" },
  });
});

test("buildServiceIntegrityDimensions is empty without a token", () => {
  assert.deepStrictEqual(potoken.buildServiceIntegrityDimensions(undefined), {});
});
```

- [ ] **Step 2: Run it to verify it fails**

```bash
node --test test/unit/potoken.test.js
```

Expected: FAIL — `potoken.buildServiceIntegrityDimensions is not a function`.

- [ ] **Step 3: Implement the helper**

Append to `lib/potoken.js`:

```js
/**
 * Player requests carry the session token under serviceIntegrityDimensions.
 * Spreading an empty object keeps the payload byte-identical when absent.
 *
 * @param {string} [poToken]
 * @returns {Object}
 */
exports.buildServiceIntegrityDimensions = poToken => (poToken ? { serviceIntegrityDimensions: { poToken } } : {});
```

- [ ] **Step 4: Run the test to verify it passes**

```bash
node --test test/unit/potoken.test.js
```

Expected: PASS, 10 tests.

- [ ] **Step 5: Wire it into the three player fetchers**

Each fetcher builds a `payload` object. Add the spread as the last property of each.

In `fetchAndroidVRPlayer` (line ~396), change:

```js
  const payload = {
    context: ANDROID_VR_CONTEXT,
    videoId,
    playbackContext: await getPlaybackContext(info.html5player, options),
    ...CHECK_FLAGS,
  };
```

to:

```js
  const payload = {
    context: ANDROID_VR_CONTEXT,
    videoId,
    playbackContext: await getPlaybackContext(info.html5player, options),
    ...CHECK_FLAGS,
    ...potoken.buildServiceIntegrityDimensions(options.poToken),
  };
```

In `fetchIosJsonPlayer` (line ~496), the payload ends with the `context` object. Add the spread after it, so the object literal ends:

```js
      user: {
        lockedSafetyMode: false,
      },
    },
    ...potoken.buildServiceIntegrityDimensions(options.poToken),
  };
```

In `fetchAndroidJsonPlayer` (line ~561), apply the same change: add `...potoken.buildServiceIntegrityDimensions(options.poToken),` as the final property of the `payload` object literal.

- [ ] **Step 6: Verify nothing regressed against the live API**

```bash
YTDL_TEST_NETWORK=1 node -e '
require("./lib/index.js").getInfo("aqz-KE-bpKQ").then(
  i => console.log("formats:", i.formats.length),
  e => { console.log("FAILED:", e.message); process.exit(1); });'
```

Expected: a non-zero format count, proving the payload change did not break clients when no token is configured.

- [ ] **Step 7: Commit**

```bash
npm run prettier
git add lib/potoken.js lib/info.js test/unit/potoken.test.js
git commit -m "feat: send session PoToken in player requests

Adds serviceIntegrityDimensions to the IOS, ANDROID and ANDROID_VR player
payloads when a token is configured. Payloads are unchanged without one."
```

---

### Task 6: Bundled token generator

**Files:**
- Create: `lib/potoken-generator.js`
- Modify: `package.json` (`optionalDependencies`, `exports`, `files`)
- Test: `test/unit/potoken-generator.test.js` (create), `test/integration/potoken.test.js` (create)

**Interfaces:**
- Consumes: the `{ videoId, visitorData }` object passed by `potoken.resolve` from Task 3.
- Produces: `generate({ videoId, visitorData }) => Promise<{ poToken, visitorData }>` and `mint(contentBinding) => Promise<string>`, both exported from `lib/potoken-generator.js`. `mint` builds the BotGuard minter once per process and reuses it — this is the caching the spec calls for.

- [ ] **Step 1: Write the failing dependency-guard test**

Create `test/unit/potoken-generator.test.js`:

```js
const test = require("node:test");
const assert = require("node:assert");
const generator = require("../../lib/potoken-generator.js");

test("exports a generate function", () => {
  assert.strictEqual(typeof generator.generate, "function");
});

test("requiring the module does not pull in jsdom", () => {
  // The generator must load its heavy dependencies lazily, so merely
  // requiring it must leave jsdom out of the module cache.
  const loaded = Object.keys(require.cache).some(p => p.includes(`${require("path").sep}jsdom${require("path").sep}`));
  assert.strictEqual(loaded, false, "jsdom was loaded at require time");
});
```

- [ ] **Step 2: Run it to verify it fails**

```bash
node --test test/unit/potoken-generator.test.js
```

Expected: FAIL — `Cannot find module '../../lib/potoken-generator.js'`.

- [ ] **Step 3: Write the generator**

Create `lib/potoken-generator.js`:

```js
// BotGuard attestation. Isolated here because it carries the only heavy,
// optional dependencies in the package and because YouTube changes this
// machinery often — swapping it out must not touch core code.
const REQUEST_KEY = "O43z0dpjhgX20SCx4KAo";

const MISSING_DEPS_MESSAGE =
  "PoToken generation requires the optional dependencies 'bgutils-js' and 'jsdom'. " +
  "Install them with: npm install bgutils-js@^4 jsdom@^29 — or supply a token yourself " +
  "via the `poToken` option.";

let modulesPromise = null;

// bgutils-js is ESM, so it is pulled in with await import() from this CJS module.
const loadModules = () => {
  if (!modulesPromise) {
    modulesPromise = (async () => {
      try {
        const [{ JSDOM }, botguard, utils, webpo] = await Promise.all([
          import("jsdom"),
          import("bgutils-js/botguard"),
          import("bgutils-js/utils"),
          import("bgutils-js/webpo"),
        ]);
        return { JSDOM, ...botguard, ...utils, ...webpo };
      } catch (err) {
        modulesPromise = null;
        throw new Error(`${MISSING_DEPS_MESSAGE} (original error: ${err.message})`);
      }
    })();
  }
  return modulesPromise;
};

let minterPromise = null;

const createMinter = async () => {
  const { JSDOM, BotGuardClient, getChallenge, buildURL, getHeaders, USER_AGENT, WebPoMinter } = await loadModules();

  const dom = new JSDOM('<!DOCTYPE html><html lang="en"><head><title></title></head><body></body></html>', {
    url: "https://www.youtube.com/",
    referrer: "https://www.youtube.com/",
    userAgent: USER_AGENT,
  });

  Object.assign(globalThis, {
    window: dom.window,
    document: dom.window.document,
    location: dom.window.location,
    origin: dom.window.origin,
  });
  if (!Reflect.has(globalThis, "navigator")) {
    Object.defineProperty(globalThis, "navigator", { value: dom.window.navigator });
  }

  const challenge = await getChallenge({ fetchFunction: fetch, requestKey: REQUEST_KEY });
  const interpreter = challenge.interpreterJavascript?.privateDoNotAccessOrElseSafeScriptWrappedValue;
  if (!interpreter) throw new Error("BotGuard interpreter javascript not available");
  new Function(interpreter)();

  const botGuardClient = await BotGuardClient.create({
    program: challenge.program,
    globalName: challenge.globalName,
    globalObject: globalThis,
  });

  const webPoSignalOutput = [];
  const botguardResponse = await botGuardClient.snapshot({ webPoSignalOutput });

  const response = await fetch(buildURL("GenerateIT", true), {
    method: "POST",
    headers: getHeaders(),
    body: JSON.stringify([REQUEST_KEY, botguardResponse]),
  });
  if (!response.ok) throw new Error(`Integrity token request failed: HTTP ${response.status}`);

  const [integrityToken, estimatedTtlSecs, mintRefreshThreshold, websafeFallbackToken] = await response.json();

  return WebPoMinter.create(
    { integrityToken, estimatedTtlSecs, mintRefreshThreshold, websafeFallbackToken },
    webPoSignalOutput,
  );
};

/**
 * Mints a token for an arbitrary content binding. The minter is expensive to
 * build, so it is created once and reused for the life of the process.
 *
 * @param {string} contentBinding a video id, visitor id or data sync id
 * @returns {Promise<string>}
 */
const mint = (exports.mint = async contentBinding => {
  if (!minterPromise) {
    minterPromise = createMinter().catch(err => {
      minterPromise = null;
      throw err;
    });
  }
  const minter = await minterPromise;
  return minter.mintAsWebsafeString(contentBinding);
});

/**
 * Provider entry point used by lib/potoken.js. Content tokens are bound to the
 * video id, so they are minted per call and never cached across videos.
 *
 * @param {{videoId: string, visitorData: (string|undefined)}} params
 * @returns {Promise<{poToken: string, visitorData: (string|undefined)}>}
 */
exports.generate = async ({ videoId, visitorData }) => ({
  poToken: await mint(videoId),
  visitorData,
});
```

- [ ] **Step 4: Run the unit test to verify it passes**

```bash
node --test test/unit/potoken-generator.test.js
```

Expected: PASS, 2 tests.

- [ ] **Step 5: Declare the dependencies and entry point**

Move `bgutils-js` and `jsdom` out of `devDependencies` (where Task 1 put them) and into a new `optionalDependencies` block in `package.json`:

```json
"optionalDependencies": {
  "bgutils-js": "^4.0.3",
  "jsdom": "^29.1.1"
}
```

Do **not** use `^30` for jsdom — see Global Constraints.

Add an `exports` map so the generator has a public subpath, keeping `main` for backwards compatibility:

```json
"exports": {
  ".": "./lib/index.js",
  "./potoken": "./lib/potoken-generator.js",
  "./package.json": "./package.json"
}
```

- [ ] **Step 6: Write the end-to-end integration test**

Create `test/integration/potoken.test.js`:

```js
const assert = require("node:assert");
const ytdl = require("../../lib/index.js");
const { networkTest, collect } = require("../helpers");

const LONG_VIDEO = "aqz-KE-bpKQ";

networkTest("generatePoToken produces urls carrying a pot parameter", async () => {
  const info = await ytdl.getInfo(LONG_VIDEO, { generatePoToken: true });
  const format = ytdl.chooseFormat(info.formats, { quality: "highestaudio" });
  assert.ok(new URL(format.url).searchParams.has("pot"), "format url must carry pot");
});

networkTest("generatePoToken downloads an adaptive format to completion", async () => {
  const info = await ytdl.getInfo(LONG_VIDEO, { generatePoToken: true });
  const format = ytdl.chooseFormat(info.formats, { quality: "highestaudio" });
  const expected = parseInt(format.contentLength);
  const bytes = await collect(ytdl.downloadFromInfo(info, { quality: "highestaudio", generatePoToken: true }));
  assert.strictEqual(bytes, expected, `downloaded ${bytes} of ${expected} bytes`);
});

networkTest("no token configured still returns formats without throwing", async () => {
  const info = await ytdl.getInfo(LONG_VIDEO);
  assert.ok(info.formats.length > 0);
});

networkTest("the BotGuard minter is built once and reused", async () => {
  const generator = require("../../lib/potoken-generator.js");
  // The first mint runs the full attestation; the second must reuse the minter,
  // so it returns far faster than a cold build ever could.
  const coldStart = Date.now();
  await generator.mint(LONG_VIDEO);
  const coldMs = Date.now() - coldStart;

  const warmStart = Date.now();
  const second = await generator.mint(LONG_VIDEO);
  const warmMs = Date.now() - warmStart;

  assert.ok(typeof second === "string" && second.length > 0, "second mint returned a token");
  assert.ok(warmMs < coldMs / 2, `expected reuse to be much faster: cold ${coldMs}ms, warm ${warmMs}ms`);
});
```

- [ ] **Step 7: Run the full suite**

```bash
YTDL_TEST_NETWORK=1 npm test
```

Expected: PASS. **This is the acceptance criterion** — the two Task 2 tests that failed with 403 must now pass, because `downloadFromInfo` resolves a token when `generatePoToken` is set.

If the Task 2 tests still fail while the Task 6 tests pass, the cause is that Task 2's tests do not opt into token generation. That is correct and expected: update them to pass `{ generatePoToken: true }` and note in the commit that token generation is opt-in.

- [ ] **Step 8: Commit**

```bash
npm run prettier
git add lib/potoken-generator.js package.json test/
git commit -m "feat: add bundled BotGuard PoToken generator

Lazily loads bgutils-js and jsdom (optional dependencies, jsdom pinned to
^29 for Node 20 support) and mints content-bound tokens. Exposed at the
./potoken subpath; the core never imports it."
```

---

### Task 7: Types and documentation

**Files:**
- Modify: `typings/index.d.ts:347-358` (`getInfoOptions`)
- Modify: `README.md`

**Interfaces:**
- Consumes: the option names established in Tasks 3–6: `poToken`, `visitorData`, `poTokenProvider`, `generatePoToken`.
- Produces: nothing consumed by later tasks.

- [ ] **Step 1: Extend the typings**

In `typings/index.d.ts`, inside `interface getInfoOptions`, add:

```ts
      poToken?: string;
      visitorData?: string;
      generatePoToken?: boolean;
      poTokenProvider?: (params: {
        videoId: string;
        visitorData?: string;
      }) => Promise<{ poToken: string; visitorData?: string }>;
```

While in this interface, correct the stale `playerClients` union — the code supports `ANDROID_VR` and has no `WEB` fetcher. Change:

```ts
      playerClients?: Array<"WEB_EMBEDDED" | "TV" | "IOS" | "ANDROID" | "WEB">;
```

to:

```ts
      playerClients?: Array<"ANDROID_VR" | "WEB_EMBEDDED" | "TV" | "IOS" | "ANDROID">;
```

- [ ] **Step 2: Verify the typings compile**

```bash
npx tsc --noEmit typings/index.d.ts
```

Expected: no errors.

- [ ] **Step 3: Document it in the README**

Add a section after the existing proxy/agent documentation:

````markdown
## PoToken (full-length downloads)

YouTube serves only the first ~60 seconds of adaptive (audio-only and video-only)
formats unless the streaming URL carries a Proof of Origin Token. Without one,
`ytdl(url, { filter: "audioonly" })` fails with `Status code: 403` partway through.
Progressive formats such as itag 18 are unaffected.

### Option 1: let ytdl-core generate the token

Install the optional dependencies, then opt in:

```bash
npm install bgutils-js@^4 jsdom@^29
```

```js
const stream = ytdl(url, { filter: "audioonly", generatePoToken: true });
```

> jsdom must be `^29`. Version 30 requires Node 22 or newer, while this package
> supports Node 20.18.1 and up.

### Option 2: supply your own token

```js
ytdl.getInfo(url, { poToken: "YOUR_TOKEN", visitorData: "YOUR_VISITOR_DATA" });
```

### Option 3: provide a token source

```js
ytdl.getInfo(url, {
  poTokenProvider: async ({ videoId }) => ({ poToken: await myTokenService(videoId) }),
});
```

### Option 4: no token at all

Use cookies with the `ANDROID_VR` client, which does not require a PoToken for
streaming:

```js
const agent = ytdl.createAgent(cookies);
ytdl.getInfo(url, { agent, playerClients: ["ANDROID_VR"] });
```
````

- [ ] **Step 4: Verify the documented examples are accurate**

Re-read each snippet against the implemented option names in `lib/potoken.js`. Every option shown (`generatePoToken`, `poToken`, `visitorData`, `poTokenProvider`) must exist in `resolve` and `selectProvider`.

- [ ] **Step 5: Commit**

```bash
npm run prettier
git add typings/index.d.ts README.md
git commit -m "docs: document PoToken options and fix playerClients typing

Also corrects the playerClients union, which listed a non-existent WEB
client and omitted ANDROID_VR."
```

---

## Verification

After all tasks, from a clean checkout:

```bash
npm run test:unit                    # must pass with no optional deps installed
YTDL_TEST_NETWORK=1 npm test         # full suite including the acceptance test
```

The acceptance criterion is the Task 2 / Task 6 test asserting
`bytes === parseInt(format.contentLength)` for an adaptive audio-only format on a
video longer than 60 seconds. That test fails with HTTP 403 before this work and
passes after it.
