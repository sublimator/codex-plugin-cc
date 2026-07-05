<!--
rendered_from: broker-lifecycle.md.j2
rendered_at: 2026-07-05T12:35:17Z
branch: fix/shared-broker-upstream
commit: 86ea7dd
commit_message: docs: dossier wording — setup tests fake only PATH (codex re-review)
-->

---

<sub>Last updated: 2026-07-05 | branch: fix/shared-broker-upstream | commit: 86ea7dd (docs: dossier wording — setup tests fake only PATH (codex re-review))</sub>

---

# The Broker Lifecycle Dossier

`codex-plugin-cc` coordinates several independent OS processes — interactive
sessions, detached background workers, and lifecycle hooks — through shared JSON
files (`state.json`, `broker.json`) and one broker process, all keyed by the
working directory. Nothing serialises them: there is no cross-process lock,
compare-and-swap, or liveness check anywhere in the tree, so every mutation is
an unguarded read-modify-write or check-then-act done as if a single owner held
the files. The `tmp+rename` writes keep individual writes from tearing but never
make the compound operations atomic — which is why the same defect surfaces
repeatedly: concurrent processes clobber each other's records, spawn duplicate
brokers, and delete files or kill pids out from under a peer still using them.

*(That paragraph is the shared conclusion of three independent reviewers given
only the raw findings and the source, asked to state the core problem cold.)*

Documented 2026-07-05 against plugin v1.0.5 (upstream `80c31f9`) on macOS, with
multiple concurrent Claude Code sessions. Below: four bugs worked through in
detail (Part I), then an audit of the shared-state surfaces that found 19
confirmed races sharing this cause (Part II).

## TL;DR

| # | Bug | Severity | One line |
|---|-----|----------|----------|
| 1 | **SessionEnd tears down a broker in use** | High | Any session ending shuts down the cwd-shared broker with no ownership check — killing other sessions' running jobs mid-turn and leaving their records stuck at `"running"`. |
| 2 | **The test suite leaks a process pair per broker** | Medium | Every `npm test` run leaves ~25–30 orphaned `app-server-broker` + `codex app-server` processes. We found **184** accumulated, the oldest 30+ hours old. |
| 3 | **Tests read live workspace state** | Medium | The setup tests run in the real repo cwd and talk to the real `broker.json` — one live broker made five setup tests fail (one on `'shared' !== 'direct'`, four on auth assertions contaminated via `reuseExistingBroker`). |
| 4 | **Broker keeps answering after its child dies** | Medium | The broker's child app-server exits, but the broker keeps passing health probes locally, so the next turn fails with `connection closed` — the pass/fail/pass/fail of #402. |

This began as three bugs found by hand. A fourth turned up while fixing them
(the zombie broker, upstream #402). All four only bite when more than one
session shares a directory, so we then checked the shared-state surfaces
directly: eight read-only passes, each finding re-checked by a separate
verifier. That found 19 more concurrency races. Most share a couple of root
causes.

Two parts:

- **Part I** — the four bugs, each worked through with permalinks to the source.
- **Part II** — the audit, and what the 19 findings have in common.

Code references in Part I link to the marked source in this fork (the
`//@@` markers are the anchors). Related issues: #380, #402, #286, #416.


# Part I — the four bugs

*All four bugs are fixed in this branch. Each section describes the bug as it
was and the fix that landed; the snippets below are the pre-fix code (pinned to
commit `86ea7dd`) with the `//@@` markers around the relevant lines.*

---

## Bug 1 — SessionEnd tears down a broker other sessions are using

Every Claude Code session in the same directory shares one Codex broker
(`broker.json` is keyed by cwd). When a session ended, the SessionEnd hook
found the broker by cwd and shut it down unconditionally — killed its process
tree and deleted the record:

📍 [`plugins/codex/scripts/session-lifecycle-hook.mjs:101-114 @ 86ea7dd`](https://github.com/sublimator/codex-plugin-cc/blob/86ea7dd6cd57b847b55cda2efcecb0e23fcd31ca/plugins/codex/scripts/session-lifecycle-hook.mjs#L101-L114)
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

There was no check for other live sessions or their in-flight jobs. The
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

📍 [`plugins/codex/scripts/session-lifecycle-hook.mjs:42-75 @ 86ea7dd`](https://github.com/sublimator/codex-plugin-cc/blob/86ea7dd6cd57b847b55cda2efcecb0e23fcd31ca/plugins/codex/scripts/session-lifecycle-hook.mjs#L42-L75)
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

📍 [`tests/helpers.mjs:7-9 @ 86ea7dd`](https://github.com/sublimator/codex-plugin-cc/blob/86ea7dd6cd57b847b55cda2efcecb0e23fcd31ca/tests/helpers.mjs#L7-L9)
```javascript
   7 export function makeTempDir(prefix = "codex-plugin-test-") {
   8   return fs.mkdtempSync(path.join(os.tmpdir(), prefix));
   9 }
```

Tests then drive the real companion, which auto-starts a broker for the
workspace. This test checked that a broker was started (`loadBrokerSession(repo)`),
ran one more `task` command against it, and ended:

📍 [`tests/runtime.test.mjs:907-915 @ 86ea7dd`](https://github.com/sublimator/codex-plugin-cc/blob/86ea7dd6cd57b847b55cda2efcecb0e23fcd31ca/tests/runtime.test.mjs#L907-L915)
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

Nothing stopped that broker afterwards. The harness cleaned up other
resources — here it reaps a throwaway `sleep` process — but never did the same
for brokers:

📍 [`tests/runtime.test.mjs:1560-1570 @ 86ea7dd`](https://github.com/sublimator/codex-plugin-cc/blob/86ea7dd6cd57b847b55cda2efcecb0e23fcd31ca/tests/runtime.test.mjs#L1560-L1570)
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

The setup tests ran the companion **in the actual repo root**, with only PATH
faked:

📍 [`tests/runtime.test.mjs:36-45 @ 86ea7dd`](https://github.com/sublimator/codex-plugin-cc/blob/86ea7dd6cd57b847b55cda2efcecb0e23fcd31ca/tests/runtime.test.mjs#L36-L45)
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

📍 [`plugins/codex/scripts/lib/codex.mjs:908 @ 86ea7dd`](https://github.com/sublimator/codex-plugin-cc/blob/86ea7dd6cd57b847b55cda2efcecb0e23fcd31ca/plugins/codex/scripts/lib/codex.mjs#L908)
```javascript
 908   const endpoint = env?.[BROKER_ENDPOINT_ENV] ?? loadBrokerSession(cwd)?.endpoint ?? null;
```

That explains one failing test. The other four setup tests are contaminated
through a second door: setup's auth check *connects to the live broker* rather
than the faked `codex` on PATH, so their auth/ready assertions are judged
against the real runtime's state instead of the fixture's:

📍 [`plugins/codex/scripts/lib/codex.mjs:945-948 @ 86ea7dd`](https://github.com/sublimator/codex-plugin-cc/blob/86ea7dd6cd57b847b55cda2efcecb0e23fcd31ca/plugins/codex/scripts/lib/codex.mjs#L945-L948)
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

## Bug 4 — the broker keeps answering after its child dies (#402)

The broker multiplexes one child `codex app-server` to every session in the
cwd. That child exits after a turn, but the broker process stays up and its
socket keeps answering the `initialize` health-probe locally, without checking
the dead child. So the next session's reuse check passes, it sends a real turn,
and the turn fails with `codex app-server connection closed.` — no `rpcCode`,
no errno, which the client's retry classifier didn't recognise. That produces
the pass/fail/pass/fail alternation described in #402.

Shipped fix, two parts: the broker now watches its child's exit and shuts
itself down, so probes fail with `ECONNREFUSED` (which the retry path already
handles); and the classifier now treats the bare "connection closed" as a
broker failure worth a direct retry. Covered by a new regression test.

## How the four relate

The test-suite bugs (2, 3) mostly get in the way of diagnosis: leaked brokers
bury the one that matters, and a live repo-root broker fails unrelated tests.
The runtime bugs (1, 4) are the ones that hurt users: an unrelated session's
exit kills your long-running job, and surviving brokers die intermittently on
reuse. They share a cause — no coordination over a shared, cwd-keyed broker —
which Part II makes explicit.

---

# Part II — the pattern

All four Part I bugs are lifecycle bugs that only appear under concurrent
sessions. So rather than keep finding them one at a time, we audited the
shared-state surfaces directly: eight read-only passes — `state.json`
read-modify-write, `broker.json` lifecycle, the broker's serving model, client
reuse/retry, the env-file and cwd-hash keying, background workers, the review
gate, and process/PID handling. Each finding was then given to a separate
verifier told to refute it — to reject anything actually serialised by process
boundaries or the blocking `spawnSync`, and to default to "not real" when it
couldn't reconstruct the interleaving.

19 findings survived. The full list, with locations and the verifiers'
reasoning, is in
[`multisession-findings-raw.md`](multisession-findings-raw.md). The useful
number, though, isn't 19 — it's how few root causes they share.

## 19 findings, 6 clusters

| Cluster | Root cause | Findings | Simple fix reaches? |
|--------|-----------|:--------:|--------------------|
| **A. Unlocked `state.json`** | `load → mutate → save` is not atomic across processes, and `saveState` then *deletes* job files absent from a possibly-stale snapshot | **8** | Partly — see scorecard |
| **B. No broker ownership** | cold-start spawn is check-then-act with no lock; `broker.json` written non-atomically | **4** | Partly |
| **C. `BROKER_BUSY` handling** | for write tasks the client retries *direct*, running two agents on one tree (finding 5); for setup/auth, a busy broker is reported as the user being busy / logged-out (findings 16, 17) | **3** | Mixed |
| **D. PID trusted without identity** | stored pids are group-killed with no liveness/identity check (PID reuse) | **2** | Partly |
| **E. Cancel clobber** | cancel overwrites a job that finished mid-cancel | **1** | Needs CAS |
| **F. Jobless-broker ownership** | the teardown gate counts only *tracked jobs*, missing jobless broker clients | **1** | Design choice |

Clusters A and B are twelve of the nineteen, and most of them come down to
unsynchronized access to a shared file. Not all: two are torn `broker.json`
writes, already fixed here by an atomic rename, and one (finding 14) is a prune
policy that drops active jobs rather than a race. But the bulk is the lock.
Two separate reviews of the findings (one given no priming) agreed on this and
on the design conclusion: the one-broker-per-cwd model is the right shape — it
serialises app-server turns over a shared working tree — but sharing JSON files
that every process mutates without coordination is not. The fix is a lock
discipline (or SQLite if it grows), not per-session brokers.

## What the simple fixes reach

The race and the damage are separable, and the damage is the simpler thing to
fix. A briefly-stale status field is self-healing; a live worker's files
deleted off disk are not. So the safe subset shipped here targets the
destruction rather than the race:

- **`broker.json` atomic write** (Cluster B) — the `tmp+rename` its sibling
  `state.json` already had. A torn read can no longer be misread as "no broker"
  and spawn a duplicate.
- **The GC guard** (Cluster A) — `pruneJobs`/`saveState` no longer delete a
  *queued or running* job's files, so a concurrent worker's output survives a
  stale-snapshot write while it runs. (Two narrow gaps remain: a worker's
  output can still be deleted in the short window after it goes terminal but
  before the stale write lands, and SessionEnd now deletes its own jobs' files
  itself since the guard stops `saveState` from doing it. Neither is the
  durable fix.)
- The Part I fixes: the SessionEnd ownership gate, the #402 self-teardown and
  retry classifier, and the `pid > 1` group-kill guard (Cluster D, partial).

Where the simple fix stops: the guard saves the job's *files*, but the *record*
can still drop from `state.json`, because `saveState` writes the caller's whole
array and a stale caller never held the other session's job. If the record is
gone, the ownership gate can't see the job and the broker can be torn down
under it. A stale write can also go the other way — resurrecting a
just-completed job as `running` with a dead pid (finding 11), which the status
and cancel paths then trust. Closing these needs the lock (so cleanup reads
fresh state) or a merge that knows the caller's intent — you have to tell
"another writer added this" apart from "I'm removing my own." That's the part
the simple fix can't reach.

## What this PR does, and what it leaves

This PR fixes some of the issues, not all of them. It ships the changes that
are correct, self-contained, and low-risk: the four Part I bugs, plus the safe
subset of Clusters A/B/D — `broker.json` made atomic, the GC guard, and the
`pid > 1` group-kill guard. These remove the most destructive edges.

It does not attempt the underlying refactor. The durable fix is a design
change — a state-transaction lock and a broker-acquisition lease — which is
larger than a bug-fix PR should carry and is better done as its own reviewed
change. The scorecard above leaves the seam visible so this boundary is clear:
what's left below is mostly one design change plus a few smaller fixes, not a
long list of unrelated tickets.

## The refactor this needs (not in this PR)

1. **A single locked state transaction.** Wrap `load → mutate → save` in a
   cross-process lock (lock-dir or `open(…, "wx")` with stale-lock recovery),
   expose it as one `updateStateLocked(cwd, mutator)`, and route every writer
   through it. Rewrite `cleanupSessionJobs` to remove *this session's* jobs
   from the freshly-locked state, never to write back a stale survivor array.
   Closes Cluster A (8) and the seam above.
2. **The same lock around broker acquisition** — `load/probe/spawn/publish`
   under one lease so two sessions can't both cold-start a broker. Closes the
   rest of Cluster B (4).
3. **Treat `BROKER_BUSY` as serialisation, not transport failure** — for
   write tasks, wait/queue/fail loudly rather than direct-retry into a second
   writer on the same tree (finding 5). This is a protocol decision, not a
   lock. The related setup/auth false-negatives (16, 17) are a separate,
   smaller diagnostic fix.
4. Targeted hardening for the tail: PID identity before group-kill (D),
   compare-and-set on cancel (E), broker-side shutdown awareness (F) — each a
   different mechanism.

Items 1 and 2 are the same primitive (a lock) and cover about half the
findings. Items 3 and 4 are separate mechanisms — a protocol decision and some
targeted hardening. So the remaining work is one design change plus a few
independent fixes: fewer moving parts than nineteen tickets suggest, but more
than a single lever.

---

*Generated with [projected-source](https://github.com/sublimator/projected-source):
every snippet above is extracted from the marked source in this fork at render
time, with GitHub permalinks — the `//@@start`/`//@@end` markers in the source
are the anchors. Permalinks deliberately pin the markers commit (the dossier
commit lands after it, so the two shas differ by design). Rendered from
`dossier/broker-lifecycle.md.j2`.*