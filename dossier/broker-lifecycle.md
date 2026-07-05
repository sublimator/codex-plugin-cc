<!--
rendered_from: broker-lifecycle.md.j2
rendered_at: 2026-07-05T10:10:40Z
branch: fix/shared-broker-upstream
commit: 67d5a6b
commit_message: docs: marker at setup-auth reuseExistingBroker (dossier bug 3 citation)
-->

---

<sub>Last updated: 2026-07-05 | branch: fix/shared-broker-upstream | commit: 67d5a6b (docs: marker at setup-auth reuseExistingBroker (dossier bug 3 citation))</sub>

---

# The Broker Lifecycle Dossier

**Three interlocking bugs in `codex-plugin-cc`'s shared-runtime management** — observed,
reproduced, and forensically documented on 2026-07-05 against plugin v1.0.5
(upstream `80c31f9`), macOS (Darwin 25.5.0), multiple concurrent Claude Code sessions.

## TL;DR

| # | Bug | Severity | One line |
|---|-----|----------|----------|
| 1 | **SessionEnd murders the shared broker** | High | Any Claude session ending tears down the cwd-shared broker with no ownership check — killing other sessions' running Codex jobs mid-turn and leaving their job records zombied at `"running"` forever. |
| 2 | **The test suite leaks a process pair per broker** | Medium | Every `npm test` run leaves ~25–30 orphaned `app-server-broker` + `codex app-server` processes. We found **184** accumulated, the oldest 30+ hours old. |
| 3 | **Tests read live workspace state** | Medium | The setup tests run in the real repo cwd and talk to the real `broker.json` — one live broker for the repo made five setup tests fail (one on `'shared' !== 'direct'`, four on auth assertions contaminated via `reuseExistingBroker`). |

Every code reference below is a permalink into this fork at a commit where the
relevant lines are fenced with `//@@` extraction markers — click through and read
the real source. Related upstream issues: #380, #402, #286, #416.

---

## Bug 1 — SessionEnd murders the shared broker

Every Claude Code session in the same project directory shares one Codex broker
(`broker.json` is keyed by cwd). When **any** of those sessions ends, the
SessionEnd hook resolves the broker by cwd and unconditionally shuts it down,
kills its process tree, and deletes the session record:

📍 [`plugins/codex/scripts/session-lifecycle-hook.mjs:101-114`](https://github.com/sublimator/codex-plugin-cc/blob/67d5a6bbb087983e03d687de1cbc968c87af0bd3/plugins/codex/scripts/session-lifecycle-hook.mjs#L101-L114)
```javascript
 101   if (brokerEndpoint) {
 102     await sendBrokerShutdown(brokerEndpoint);
 103   }
 104 
 105   cleanupSessionJobs(cwd, input.session_id || process.env[SESSION_ID_ENV]);
 106   teardownBrokerSession({
 107     endpoint: brokerEndpoint,
 108     pidFile,
 109     logFile,
 110     sessionDir,
 111     pid,
 112     killProcess: terminateProcessTree
 113   });
 114   clearBrokerSession(cwd);
```

There is no check for other live sessions or their in-flight jobs. The
consequences split by victim:

- **The broker and its `codex app-server`** die immediately (`terminateProcessTree`).
- **Other sessions' running jobs** die with them — but their job records are
  *not* cleaned up, because `cleanupSessionJobs` filters strictly by the ending
  session's id. The surviving records freeze at `status: "running"` with a dead
  pid: a zombie that `status` reports as live work indefinitely.

<details>
<summary><b>Field incident — timeline of one murder (2026-07-05)</b></summary>

An adversarial code-review task was running in workspace `hooks-testing`:

```
07:36:25.023Z  job task-mr7h9br3-iu7pib created (kind: task, "rescue")
07:36:25.046Z  started, pid 91472, threadId 019f3135-06f0-…
07:38:59.824Z  last job-record update
07:39:45.744Z  last log line (mid-investigation, reading a source file)
   —  broker socket dir (cxc-cbf6Da) vanished; broker pid gone
   —  another Claude session in the same cwd had ended at that moment
11m 01s        status still "running"; ps -p 91472 → no such process
   —  no macOS crash report for node: the broker was killed, it didn't crash
```

Recovery was only possible because Codex *threads* survive broker death:
`codex-companion cancel <job>` + `task --resume` continued the review with
full context. The job record itself never recovered — it had to be cancelled
manually.

</details>

<details>
<summary><b>Why the zombie: cleanupSessionJobs only reaps the ending session</b></summary>

📍 [`plugins/codex/scripts/session-lifecycle-hook.mjs:42-75`](https://github.com/sublimator/codex-plugin-cc/blob/67d5a6bbb087983e03d687de1cbc968c87af0bd3/plugins/codex/scripts/session-lifecycle-hook.mjs#L42-L75)
```javascript
  42 function cleanupSessionJobs(cwd, sessionId) {
  43   if (!cwd || !sessionId) {
  44     return;
  45   }
  46 
  47   const workspaceRoot = resolveWorkspaceRoot(cwd);
  48   const stateFile = resolveStateFile(workspaceRoot);
  49   if (!fs.existsSync(stateFile)) {
  50     return;
  51   }
  52 
  53   const state = loadState(workspaceRoot);
  54   const removedJobs = state.jobs.filter((job) => job.sessionId === sessionId);
  55   if (removedJobs.length === 0) {
  56     return;
  57   }
  58 
  59   for (const job of removedJobs) {
  60     const stillRunning = job.status === "queued" || job.status === "running";
  61     if (!stillRunning) {
  62       continue;
  63     }
  64     try {
  65       terminateProcessTree(job.pid ?? Number.NaN);
  66     } catch {
  67       // Ignore teardown failures during session shutdown.
  68     }
  69   }
  70 
  71   saveState(workspaceRoot, {
  72     ...state,
  73     jobs: state.jobs.filter((job) => job.sessionId !== sessionId)
  74   });
  75 }
```

The filter `job.sessionId === sessionId` is correct for its purpose — but it
means the collateral damage of the broker teardown (other sessions' jobs) is
invisible: killed processes, untouched records.

</details>

**Fix shape** (implemented and field-tested on a private branch, 90/90 tests):
before tearing down, consult the workspace job state; if any *other* session
has `queued`/`running` jobs, skip the teardown entirely — last session out
turns off the lights. Treat unattributable jobs (missing session id on either
side) as someone else's: the worst case flips from "killed live work" to "an
idle orphan broker", which the next session reuses or restarts harmlessly.

---

## Bug 2 — the test suite leaks a broker + app-server pair per run

The test harness creates workspaces with a recognizable prefix:

📍 [`tests/helpers.mjs:7-9`](https://github.com/sublimator/codex-plugin-cc/blob/67d5a6bbb087983e03d687de1cbc968c87af0bd3/tests/helpers.mjs#L7-L9)
```javascript
   7 export function makeTempDir(prefix = "codex-plugin-test-") {
   8   return fs.mkdtempSync(path.join(os.tmpdir(), prefix));
   9 }
```

Tests then drive the real companion, which — by design — auto-starts a broker
for the workspace. This test *proves* a broker was started (it checks
`loadBrokerSession(repo)`), runs one more `task` command against it, and ends:

📍 [`tests/runtime.test.mjs:907-915`](https://github.com/sublimator/codex-plugin-cc/blob/67d5a6bbb087983e03d687de1cbc968c87af0bd3/tests/runtime.test.mjs#L907-L915)
```javascript
 907   const review = run("node", [SCRIPT, "review"], {
 908     cwd: repo,
 909     env
 910   });
 911   assert.equal(review.status, 0, review.stderr);
 912 
 913   if (!loadBrokerSession(repo)) {
 914     return;
 915   }
```

Nothing stops that broker. Ever. The harness demonstrably knows how to clean up
after itself — here it is conscientiously reaping a throwaway `sleep` process —
it just never extends the courtesy to brokers:

📍 [`tests/runtime.test.mjs:1560-1570`](https://github.com/sublimator/codex-plugin-cc/blob/67d5a6bbb087983e03d687de1cbc968c87af0bd3/tests/runtime.test.mjs#L1560-L1570)
```javascript
1560   t.after(() => {
1561     try {
1562       process.kill(-sleeper.pid, "SIGTERM");
1563     } catch {
1564       try {
1565         process.kill(sleeper.pid, "SIGTERM");
1566       } catch {
1567         // Ignore missing process.
1568       }
1569     }
1570   });
```

Those are **2** `t.after()` cleanup hooks in a ~2,100-line test file where
dozens of tests start real broker processes.

<details>
<summary><b>Census — 184 leaked processes on one workstation (2026-07-05)</b></summary>

```
$ ps -axo command | grep -c "app-server-broker.mjs serve"     → 91
$ ps -axo command | grep -c "codex app-server$"               → 93

broker --cwd histogram:
  ~87  /private/var/folders/…/T/codex-plugin-test-*   ← makeTempDir() prefix
    1  ~/projects/hooks-testing                        ← live session
    1  ~/projects/catalogue-tools                      ← live session
    1  ~/projects/quickjs-wasm-compilation             ← live session
    1  ~/projects/codex-plugin-cc                      ← live session

age clusters:  ~16 min and ~27 min   (two `npm test` runs that day)
               1h43m, 3h26m          (earlier runs)
               oldest: 1 day 6.5 h

per-run leak rate: ~25–30 broker + app-server pairs per full `npm test`
```

93 orphaned `cxc-*` socket/pid directories accompanied them in `$TMPDIR`.
Reaping required nothing more than SIGTERM by cwd pattern — confirming these
were unowned.

</details>

**Fix shape**: track brokers started during a test (or sweep
`codex-plugin-test-*` workspaces in a global `after`), and tear them down with
the same `SessionEnd` path the plugin already ships. As a bonus this would have
surfaced Bug 1 in CI: the teardown path is currently exercised by zero tests
that involve a real second session.

---

## Bug 3 — tests read live workspace state, so real brokers fail fake tests

The setup tests run the companion **in the actual repo root**, with only PATH
and HOME-ish variables faked:

📍 [`tests/runtime.test.mjs:36-45`](https://github.com/sublimator/codex-plugin-cc/blob/67d5a6bbb087983e03d687de1cbc968c87af0bd3/tests/runtime.test.mjs#L36-L45)
```javascript
  36   const result = run("node", [SCRIPT, "setup", "--json"], {
  37     cwd: ROOT,
  38     env: buildEnv(binDir)
  39   });
  40 
  41   assert.equal(result.status, 0);
  42   const payload = JSON.parse(result.stdout);
  43   assert.equal(payload.ready, true);
  44   assert.match(payload.codex.detail, /advanced runtime available/);
  45   assert.equal(payload.sessionRuntime.mode, "direct");
```

That final assertion — `sessionRuntime.mode === "direct"` — reaches this code,
whose fallback consults the **real** `broker.json` for the cwd it was handed:

📍 [`plugins/codex/scripts/lib/codex.mjs:908`](https://github.com/sublimator/codex-plugin-cc/blob/67d5a6bbb087983e03d687de1cbc968c87af0bd3/plugins/codex/scripts/lib/codex.mjs#L908)
```javascript
 908   const endpoint = env?.[BROKER_ENDPOINT_ENV] ?? loadBrokerSession(cwd)?.endpoint ?? null;
```

That explains one failing test. The other four setup tests are contaminated
through a second door: setup's auth check *connects to the live broker* rather
than the faked `codex` on PATH, so their auth/ready assertions are judged
against the real runtime's state instead of the fixture's:

📍 [`plugins/codex/scripts/lib/codex.mjs:945-948`](https://github.com/sublimator/codex-plugin-cc/blob/67d5a6bbb087983e03d687de1cbc968c87af0bd3/plugins/codex/scripts/lib/codex.mjs#L945-L948)
```javascript
 945     client = await CodexAppServerClient.connect(cwd, {
 946       env: options.env,
 947       reuseExistingBroker: true
 948     });
```

So if any real broker exists for the repo directory — say, because a developer
ran `codex-companion task` from the repo root an hour ago — five setup tests
fail, and nothing in the failure output hints that the cause is a background
process from earlier in the day.

<details>
<summary><b>Controlled repro — 90/90 → 85/5 → 90/90 without touching a line of code</b></summary>

```
1. baseline:            node --test tests/*.test.mjs   → # pass 90, # fail 0
2. codex-companion task "…" launched from the repo root
   (auto-starts a broker; broker.json now exists for the repo cwd)
3. same suite:          node --test tests/*.test.mjs   → # pass 85, # fail 5

   the five failures are all setup tests; the mode assertion reads:
     AssertionError: 'shared' !== 'direct'
       expected: 'direct'   actual: 'shared'
       at tests/runtime.test.mjs (setup test, sessionRuntime.mode)
   the remaining four fail their auth/ready assertions — setup's auth
   check reached the live broker instead of the faked codex binary.

4. SessionEnd teardown of that broker (the plugin's own hook)
5. same suite:          setup tests → 8/8; full suite → 90/90
```

The diff between a green and a red CI run here is one background process.

</details>

**Fix shape**: point the setup tests at a `makeTempDir()` cwd (they only assert
on binary/auth detection, not on repo contents), or let tests inject an
isolated state/broker directory the way they already inject PATH.

---

## How the three compound

Bug 2 breeds orphan brokers on every developer machine that runs the tests —
in temp workspaces, so they don't trip Bug 3 directly, but they bury the *one*
broker that matters in a haystack of dozens and make any process-level
diagnosis miserable. Bug 3 turns any legitimate repo-root broker — the natural
consequence of a developer using the companion in their own checkout — into
five phantom test failures. And while you're debugging *that*, Bug 1 is
killing your long-running Codex jobs whenever an unrelated Claude session
exits, leaving zombie job records that report `"running"` from beyond the
grave. Each bug manufactures evidence that misdirects the investigation of
the others — which is how all three survived to v1.0.5.

---

*Generated with [projected-source](https://github.com/sublimator/projected-source):
every snippet above is extracted from the marked source in this fork at render
time, with GitHub permalinks — the `//@@start`/`//@@end` markers in the source
are the anchors. Permalinks deliberately pin the markers commit (the dossier
commit lands after it, so the two shas differ by design). Rendered from
`dossier/broker-lifecycle.md.j2`.*