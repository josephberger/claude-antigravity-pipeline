# Claude + Antigravity Brief Pipeline

A blueprint for an unattended, two-agent development loop: **Claude Code reviews (and usually plans); Antigravity (AG, the `agy` CLI) builds**, with other builder CLIs (e.g. Grok Build) as optional extra slots. It was built and battle-tested on a Godot game project. This document describes it generically, so you can set it up for another project. Project-specific examples are marked *(example)*.

**If you're a Claude session reading this to set it up:** read it all once, then follow the build order in section 10. Adapt names, paths, the git host, the test command and the allow-lists to the new project. Don't copy the example values blindly.

**This is the blueprint, not an operator guide.** Each project keeps its own `PIPELINE.md` (how to pause, reset, read the viewer) next to its code. This file explains how to build one.

---

## 1. The idea

```
 user ──talks to──► Claude (interactive)
                     │ writes a brief: briefs/ag-NNN-<topic>.md, **Status:** ready, pushed to main
                     ▼
 launchd (every 5 min) ──► dispatch.py
                     │ 1. AG builds the brief headless in its OWN clone, on branch ag/<topic>, and pushes the branch
                     │ 2. Claude reviews headless in ANOTHER clone: tests, probes, screenshots
                     │    → briefs/reviews/<id>-review.md with a verdict (APPROVE / CHANGES / BLOCKED)
                     │    → on CHANGES, also a follow-up brief <id>-fixes.md (same branch), picked up next tick
                     │ 3. merge: manual by default; automatic when ~/agents/state/AUTO_MERGE exists
                     ▼
 main holds reviewed work only. Chained briefs (**Depends on:**) start when their dependency is done.
```

**Division of labour:**
- **Planner (usually interactive Claude):** turns the user's intent into briefs. Another planning agent can draft briefs too, as long as the designated reviewer still reviews the result.
- **Builder (AG, or another CLI in the builder slot):** heavy lifting: new systems, many-file features, big refactors.
- **Reviewer (headless Claude):** an independent reviewer that never edits the branch.
- **Docs owner (interactive Claude or a separate docs agent):** keeps the design docs / bible and the operator guide current after merges (section 4.5). Not the builder's or reviewer's job.
- **Interactive Claude** also makes small fixes directly and merges when AUTO_MERGE is off.

**Why two agents:** the builder and the reviewer have different blind spots, and the reviewer runs its own probes instead of trusting the builder's tests. Running unattended means work continues while the user is away. Each agent bills to its own quota.

---

## 2. Directory layout and remotes

```
~/agents/
  bin/        dispatch.py, logview.py, PIPELINE.md   ← INSTALLED copies (launchd runs these)
  ag/<project>/          AG's clone (plus sibling repos it needs, e.g. docs or assets)
  cc/<project>/          the reviewer's clone (plus siblings)
  state/      <brief-id>.json per brief, dispatch.lock, PAUSED, AUTO_MERGE, <BUILDER>_QUOTA_UNTIL
  logs/       dispatch.log, <id>-ag-<stamp>.jsonl, <id>-review-<stamp>.jsonl, <id>-merge-tests.log
  quota-check/           an empty dir, so `agy -p /usage` never runs inside a repo
  logview-venv/          Python venv for the log viewer (Flask)
```

- **Source of truth for the scripts:** they live in the project repo under `tools/pipeline/`. You **copy** them (and the project's `PIPELINE.md`) to `~/agents/bin/` after every edit. (The reason: macOS background services can't read `~/Documents` without a privacy prompt. Anything launchd runs must live outside it.)
- **Never work in the pipeline clones by hand.** They're disposable and owned by the dispatcher. The user's own checkout (e.g. `~/Documents/<project>`) is where interactive Claude works.

### 2.1 Remotes

- **Pick one git host as the source of truth** (GitHub, GitLab, Gitea, whatever the project uses; GitHub-primary *(example)*). Every clone's `origin` points there: the AG clone, the reviewer clone, sibling docs clones, and the user's own checkout.
- **The dispatcher reads briefs from `origin/main` on that host** and pushes branches, reviews and merges there. Keep the remote name/URL in config or the clone itself; **don't hardcode a particular host** in the dispatcher or the prompts.
- **Mirrors and backups are optional.** If you keep one, sync it one way from the source of truth (a small launchd job running `git fetch` + `git push --mirror`, or the host's own push-mirror feature *(example)*). Agents never push to the mirror directly, so the two can't fork.
- **Moving hosts later:** repoint `origin` in every clone (including sibling docs repos; it's easy to miss one and get a silent fork), fast-forward, and check both ends show the same SHA before the next tick.

---

## 3. Briefs: the work orders

`briefs/ag-NNN-<topic>.md`. The dispatcher reads only these header lines (regex, from `origin/main`):

```markdown
# AG Brief 012: Grid Inventory

**Status:** ready            ← draft | ready | review | blocked | done (done may carry "(auto-merged DATE)")
**Depends on:** `ag-011`     ← optional; starts only once a brief whose id starts with this prefix is done
**Branch:** `ag/grid-inventory`
**Type:** …   **Read first:** …
```

**Who writes them:** interactive Claude by default, but any planning agent can draft a brief. New briefs start as `draft`; only the user (or the planner on the user's say-so) flips them to `ready`. The designated reviewer still reviews whatever the builder produces.

**Body:** goal, the user's decisions (quoted, with dates), parts or requirements, required tests (unit and integration, with specific scenarios and numbers), screenshots if visual, docs updates, out of scope, and a definition-of-done checklist. Good briefs:
- **Name exact files, functions and data fields.** The builder can't ask questions.
- **Require tests that would fail without the change**, and forbid loosening existing assertions. Any changed assertion must be listed in the handoff with its old value, new value and reason.
- **Say what's out of scope.**
- **Quote the user's decisions verbatim**, so neither agent re-decides them.
- **Keep them small.** One builder, one branch, one reviewable diff.

**Naming and chaining rules (learned the hard way):**
- **Fixes briefs:** `<id>-fixes.md`, then `<id>-fixes-2.md`, on the **same branch**, with **no** `Depends on:` line. A fixes brief that "depends on" its parent deadlocks: the parent can't be `done` until the fixes land. The dispatcher also ignores a dependency on a prefix of the brief's own id, as a backstop.
- **Merge now, clean up later:** when a review says CHANGES but nothing blocks a merge and the user chooses to merge anyway, merge the branch, mark the brief and its fixes `done`, and put the leftovers in a **new numbered brief** (new id, new branch). Don't open yet another `-fixes` round on a branch that's already merged; it thrashes the same branch and muddles which brief owns what.
- **Split briefs:** if a brief has to be split (say, waiting on assets), the remainder is also a new numbered brief, not a "fix".
- **Status lines:** AG sets `review` on its branch when finished, or `blocked` with questions. `main` owns briefs. On merge, conflicting `briefs/*` files keep main's copy.

### 3.1 Serial by preference

The dispatcher is already serial: one action per tick, and only one brief in flight at a time. On top of that, **recommend finishing one brief 100% (merged, docs updated) before marking the next one ready**, unless the user explicitly queues several. A queue of ready briefs works, but:
- quota burns faster and less predictably overnight;
- a CHANGES review produces a fixes brief that jumps ahead of the queue, so the order you expected isn't the order you get;
- a later brief may be built on a main that's missing an earlier brief's fix.

If the user does queue several, use `Depends on:` for real ordering, and hold anything that shouldn't burn a builder pass unattended as `draft`.

---

## 4. The dispatcher (`dispatch.py`, one tick)

launchd runs it every 300 s (`StartInterval`, `RunAtLoad`). Each tick:

1. **Exit if `state/PAUSED` exists.** Take a non-blocking `flock` on `state/dispatch.lock`; if a previous tick still holds it (a long AG run), exit.
2. **`git fetch --prune`** both clones, and list briefs from `origin/main` (status, branch, depends).
3. **Pick the next action:**
   - **Crash check:** any brief in `ag_running` or `review_running` while we hold the lock means a crash. Report it and stop; the user runs `reset`.
   - **Then:** `ag_done` → run the review. `ag_retry` with `retry_after` passed → resume AG.
   - **Otherwise:** the first brief with `Status: ready`, no state file, and all dependencies done → run AG.
4. **Run it**, recording every phase change in `state/<id>.json` with timestamps. A crash inside an action sets `phase: failed` with the reason, plus a notification. Never fail silently.

**Phases:** `ag_running → ag_done → review_running → reviewed (verdict) → merged`, plus `ag_retry`, `failed` and `merge_blocked`.

**CLI:** `dispatch.py` (one tick), `--dry-run`, `status`, `reset <id>` (forget state), `resume <id>` (salvage leftovers, then retry).

**Notifications:** macOS `osascript display notification` at each step, also logged as `NOTIFY …` lines in `dispatch.log`.

### 4.1 Running the builder

```python
model = pick_model(AG_MODELS)                       # first entry whose pool has quota (see below)
cmd = builder_cmd(model, prompt)                    # per-prefix command line (agy:…, grok:…)
proc = subprocess.Popen(cmd, cwd=AG_REPO, stdout=logfile, stderr=STDOUT)
while proc.poll() is None:
    for k in godot_watchdog(AG_REPO, 300): log(f"watchdog killed hung godot {k}")
    time.sleep(60)
```

For agy, `builder_cmd` is roughly `[AGY, "-p", prompt, "--model", m, "--output-format", "stream-json", "--print-timeout", "6h"]`.

**`sync_main` before attempt 1:** ff-only the clone to `origin/main`, refusing a dirty tree. **But first auto-clean known generated junk**: untracked engine import files (Godot *(example)*: `*.import`) and similar caches that a crashed import or engine run leaves behind. Delete only untracked files matching that explicit pattern, then do the dirty-tree check. Without this, one crashed run blocks every following brief on "dirty tree".

**Picking a builder (`AG_MODELS`):**
- `AG_MODELS` is an **ordered** list with a prefix per builder CLI: `agy:<model>`, `grok:<model>`, …  *(example: Gemini first, then a Claude model inside agy, then Grok)*. Set it in the plist's environment so reordering is a one-line change.
- **Quota per pool:** before each run, check each builder's quota the way that CLI allows (agy: `agy -p /usage` from the empty `quota-check/` dir; a CLI with no usage command: the `<BUILDER>_QUOTA_UNTIL` state file set after its last 429). Use the first entry whose pool has quota.
- **On a 429 / quota wall:** salvage, park that pool until its reset, and **fall over to the next entry about 2 min later without using an attempt.** Wait for the earliest reset only when every pool is out.
- **Override switches:** `IGNORE_<POOL>_QUOTA`-style env vars or state files let the user say "the usage reading is wrong, try anyway" without editing code.
- **Smoke-test the exact `builder_cmd()` output** in a scratch repo before a new builder (or a changed flag list) runs its first real brief. A hand-typed trial command proves nothing about the one the dispatcher builds.

**Builder prompt (template):**
  - "You are running UNATTENDED; nobody can answer questions."
  - "Read AGENTS.md completely, including Unattended Runs."
  - "Execute `{brief}` on `{branch}`."
  - "If a decision is the user's, don't guess: use the BLOCKED protocol."
  - "Push the branch, never main."
  - "End with exactly one line: `RESULT: DONE` or `RESULT: BLOCKED`."

- **Reading the result:** take the **last** `RESULT: (DONE|BLOCKED)` in the log, after stripping the prompt's own `"RESULT: DONE or RESULT: BLOCKED"`. AG's context-compaction summaries echo the prompt into the log, and a naive "BLOCKED anywhere in the text" check reported finished runs as BLOCKED.
- **If the run ended without a RESULT** (denied command, crash, timeout): **salvage** it, then retry.
  - **Salvage:** commit everything left on the branch as `WIP (pipeline salvage)` and push.
  - **Retry with a resume prompt:** the attempt number, the stop reason (including the denied command, parsed from the log), "see `git log main..HEAD`", and "clean up scratch files".
  - **Before resuming,** merge `origin/main` into the branch so the latest rules and tools are present. If that conflicts, copy just the test runner in and point AG at `git show origin/main:AGENTS.md`.
  - **Retry timing:** wait 10 min before attempt 2 and 45 min before attempt 3. At most 3 attempts, then `failed`.
- **Quota walls (429) don't use an attempt** (above). If attempt 1 died before doing anything, start fresh instead of resuming.
- **If the builder reports a result but never pushed,** salvage-push anyway.

### 4.2 The watchdog

**Problem:** the builder sometimes launches a process that never exits, most often a scratch test script that errors or never calls quit. Then the agent waits on it for hours. The agy CLI's own "waiting up to 6h0m0s for 1 background task(s)" message looks alarming but is only its upper bound.

**Solution:** it's **not a separate daemon.** It's a function inside the dispatcher process that's supervising AG, called once a minute while AG runs:

```python
def godot_watchdog(repo, max_age_s):
    for pid, etime, comm in ps("-eo pid=,etime=,comm="):
        if basename(comm).lower() != "godot": continue      # the executable itself, never a shell that mentions it
        if age(etime) < max_age_s: continue                 # etime "[[dd-]hh:]mm:ss"
        cwd = lsof("-a", "-p", pid, "-d", "cwd", "-Fn")
        if f"n{repo}" in cwd: os.kill(pid, SIGKILL)          # only processes running inside AG's clone
```

- **It only targets the long-running tool inside the builder's clone.** It never touches the user's own instance (a different cwd), the reviewer's clone, or anything when no AG run is active.
- **The threshold (300 s)** sits well above the slowest legitimate run. The real test runner has its own timeouts, so it never needs the watchdog.
- **Pair it with a rule in AGENTS.md:** probe scripts must call quit *and* run with a frame or time cap (Godot *(example)*: `--quit-after 1200`). The watchdog is the backstop, not the plan.
- **Generalise for another stack:** replace `"godot"` with whatever hangs (a dev server, a test runner in watch mode, a headless browser).

### 4.3 Running the review

```python
cmd = [CLAUDE, "-p", prompt, "--model", CC_MODEL, "--output-format", "stream-json", "--verbose",
       "--add-dir", str(CC_REPO.parent), "--allowedTools", *CC_ALLOWED_TOOLS]
subprocess.run(cmd, cwd=CC_REPO, stdout=logfile, timeout=2*3600)
```

- **Before the review,** `sync_main` the reviewer clone. Because it's disposable, the dispatcher runs `reset --hard` and `clean -fd` first, wiping any reviewer scratch.
- **Review prompt (template):**
  - Review per CLAUDE.md: the handoff (`git log main..origin/{branch}`), the diff, AGENTS.md, the brief's criteria, and the design docs.
  - Check out `review/{id}` from the branch and run every verification command.
  - **Run every command in the foreground. Never start background tasks.** A headless session ends with the turn, so a backgrounded test run gets killed and no review is written.
  - **Run your own probes where the tests look weak.**
  - **Open every screenshot PNG and judge it visually**, and take your own captures.
  - **Never edit the branch.** Probes go under `/tmp`.
  - Write `briefs/reviews/{id}-review.md` on main. The first line is exactly `**Verdict:** APPROVE|CHANGES|BLOCKED`. Then test results, what was verified, issues by severity with `file:line`, whether anything actually blocks a merge, and anything the user must decide (quote BLOCKED questions verbatim with a recommendation).
  - On CHANGES, write `{id}-fixes.md` (ready, same branch, no Depends line), unless the id already ends in `-fixes`; then list the fixes for the user instead.
  - Commit only those files and push. Never change a brief's Status line.
  - **No `Co-authored-by:` or other AI-credit trailers** in commit messages.
- **Commit attribution:** keep it neutral. Headless CLIs may append a `Co-authored-by:` trailer that credits one vendor even though several agents touched the work. Either turn it off in the CLI's settings where it allows that, or tell every agent to omit AI co-author lines (and have the dispatcher strip them from its own commits). The handoff body already says who did what.
- **Reviewer model:** the review is the cheapest place to save plan usage. Prefer a **mid-size model via `CC_MODEL`** *(example: Sonnet)*; it still runs its own probes and reviews cost far less than the largest model. Switch up for a risky brief if you like. Keeping it in one variable makes that a one-line change.
- **`CC_ALLOWED_TOOLS`:** Read, Grep, Glob, Edit, Write, plus `Bash(git:*)`, `Bash(<engine>:*)`, `Bash(<test command>:*)` and read-only shell commands (ls, cat, head, tail, grep, find, wc, diff, sort, uniq, mkdir, echo).
- **After the run:**
  - If the reviewer committed but didn't push (the network blipped), `pull --rebase` and push for it.
  - Read the verdict from `origin/main:briefs/reviews/<id>-review.md`. No file means `failed`.
- **Network:** the reviewer's own `git fetch` often failed when the git host was unreachable from the headless session. Reviews then state which local ref they reviewed, and the dispatcher's push-after fixes up the rest.

### 4.4 Auto-merge (opt-in: `touch ~/agents/state/AUTO_MERGE`)

On APPROVE, in the reviewer's clone:
1. **Merge:** `reset --hard origin/main`, then `merge --no-ff origin/<branch>`.
2. **Conflicts:** if only `briefs/*` files conflict, take main's copy (AG commits its own Status edits, which collide add/add). Any other conflict means `merge --abort`, `merge_blocked`, a notification, and a stopped chain.
3. **Re-run the full test suite on the merged result.** On failure, reset, `merge_blocked`, and a notification, with the log in `<id>-merge-tests.log`.
4. **Mark briefs done:** every brief on that branch (the original plus each fixes round) gets `Status: done (auto-merged DATE)`. Commit, push, and set each state to `merged`. Dependent briefs start next tick.

**Default is manual.** The interactive Claude merges `--no-ff` in the user's checkout, runs the tests, sets Status done, pushes, and hands off to the docs owner. After an auto-merge chain, someone (interactive Claude or the docs agent) should pull, read the reviews, update the docs, report what landed, and ask whether to keep AUTO_MERGE on.

### 4.5 Docs after a merge

The design docs (a design bible, a decisions log *(example)*) and the operator guide need updating after every merge. Give that duty to **one owner**:
- **Interactive Claude**, or
- **a separate docs agent** that watches for merged reviews and writes the doc update.

Write the duty into that owner's instructions (`CLAUDE.md`, or the docs agent's own instructions): read the merged review and handoff, update the relevant chapters with the right status marks, commit, push. **Don't require the builder or the reviewer to edit the docs.** Builders that do it anyway create duplicate or conflicting entries, and the reviewer must stay read-only. If a builder did already write a docs bit, the owner checks it rather than writing it twice.

---

## 5. Guardrails for the unattended builder

Defense in depth. Each layer catches what the one before it misses.

1. **agy permissions** (`~/.gemini/antigravity-cli/settings.json`):
   - **`permissions.allow`:** `read_file` / `write_file` scoped to `~/agents/ag/**`, plus an explicit list of allowed commands.
   - **`permissions.deny`:** `sudo`, `rm -rf` / `rm -fr`, force-pushes (including `--force-with-lease`), a regex denying `git push … main`, `git reset --hard origin/main`, and writes to read-only sibling repos.
   - **`trustedWorkspaces`:** the clone root.
   - **A denied command ends the run immediately**, so make the allow-list generous for safe commands.
2. **A git pre-push hook** in AG's clone rejects `refs/heads/main`. Read-only sibling repos get a hook that rejects every push.
3. **AGENTS.md → "Unattended Runs"**, which AG reads every run:
   - **BLOCKED protocol:** commit work in progress, put a `BLOCKED:` section (questions, options, recommendation) in the final commit body, set Status `blocked`, push, stop. "Partial work plus a clear question is a good outcome."
   - **The exact allowed-command list.**
   - **Shell rules:**
     - Never write files through the shell (no heredocs, `>`/`>>`, or file-writing python); use file tools.
     - Multi-line commit messages go to `.git/HANDOFF_MSG` and use `git commit -F`, because `-m` with newlines or `#` gets denied.
     - **Quote every wildcard.** Unquoted `*` got commands denied.
   - **Commit after every working step**, since interrupted runs are salvaged.
   - **Always use the project's test runner** (it has timeouts). Probe scripts must self-terminate.
   - **Fixes briefs continue the branch** (`git switch <branch> && git pull --ff-only`).
   - **The final commit body is a handoff:** what changed and why, new or changed tests (old → new values), deviations from the brief, docs notes for the docs owner, and anything blocked. No AI co-author trailers.
4. **The watchdog** (4.2) and **retry with backoff** (4.1).
5. **The reviewer never edits the branch**, and only the merge step writes code to main.

**Model notes *(example)*:**
- One AG model was dropped because it kept writing files through the shell and hit the deny rules.
- Claude models offered *inside* agy bill to the Antigravity quota, which makes them a good fallback when the Gemini quota is low.
- Expose the models as `AG_MODELS` / `CC_MODEL` environment variables in the plist.

---

## 5b. Adding another builder CLI (example: Grok Build)

The builder slot isn't tied to agy. Any CLI that has a headless prompt mode, a model flag, streamed JSON output and permission or deny rules can join the `AG_MODELS` list with a prefix (`grok:<model>`):
- **`builder_cmd(model, prompt)`** returns the command line per builder. Grok's is `grok -p <prompt> --model <m> --output-format streaming-json --always-approve --sandbox <profile> --cwd <AG clone> --disallowed-tools <image/scheduler/ask tools> --deny 'Bash(sudo*)' …` *(example; check the CLI's current flags)*.
- **Reading the result:** Grok streams its reply as `{"type":"text","data":"…"}` word pieces, so join the deltas before searching for `RESULT:`.
- **Quota:** with no account-level usage command, detect a 429 or rate limit only in non-content lines (never file contents or agent text). Then park the builder in a `<BUILDER>_QUOTA_UNTIL` state file until its reset, or an hour, so model picking skips it.
- **Sandboxing:** prefer the CLI's own kernel sandbox where it has one. Grok's custom profile (`~/.grok/sandbox.toml`, which the sandbox itself write-protects) extends `workspace`, adds read-write for the sibling docs clone and the engine's user-data and cache folders, and denies the user's own checkouts entirely. Test it once with a run that tries a forbidden write and read, and check that both fail.
- **Deny semantics differ:** agy ends the run on a denied call; Grok reports the denial to the model and continues. The salvage-and-retry path covers both.
- **Smoke test:** run the exact `builder_cmd()` once in a scratch repo before its first real brief (4.1).
- **Viewer:** add a condenser that shows tool calls, failures and the end event, and hides word-by-word deltas.
- **Watchdog:** keep a wall-clock cap in the supervising loop (6 h) for builders that have no timeout flag of their own.

## 6. The log viewer (`logview.py`, Flask, port 5050)

A read-only, browser-based `tail -f`, run as its own launchd service (`KeepAlive`). Open it from any device on the LAN.

**Routes:**
- **`/`:** the page. A dropdown lists every file in `~/agents/logs/`, newest first, defaulting to `dispatch.log`. A help link is in the header.
- **`/stream?file=…`:** Server-Sent Events that follow the file.
  - `.jsonl` agent logs are **condensed to one readable line per step**: tool name and key argument, agent text, errors.
  - There's a separate condenser for Claude's stream-json versus agy's `step_update` events (and one per extra builder).
- **`/status`:** a human-readable chip per brief, from `state/*.json` plus switches:
  - "AG working (attempt 2, 12m)"
  - "AG stopped early, resuming after 23:40"
  - "reviewed: APPROVE"
  - "merged into main"
  - "AUTO-MERGE ON", "PIPELINE PAUSED"
- **`/quota`:** the Antigravity quota from `agy -p /usage`. That's a local slash command, not a model call. A background thread refreshes it every 5 min, running from the empty `quota-check/` dir. Chips go orange below 50% and red below 20%, and hovering shows reset times. Show parked builders (`<BUILDER>_QUOTA_UNTIL`) too.
- **`/help`:** serves the installed `PIPELINE.md`, the operator guide for the user.

**Notes:**
- Port 5000 is taken by macOS AirPlay Receiver.
- **Binding to `0.0.0.0` means no authentication**: anyone on the network can read the logs, which include prompts, diffs and file contents. Bind to `127.0.0.1` unless you trust the LAN, or put it behind something that authenticates.
- Python venv: `uv venv ~/agents/logview-venv --python 3.11 && uv pip install --python ~/agents/logview-venv/bin/python flask`.

---

## 7. launchd

Dispatcher plist (`~/Library/LaunchAgents/com.<user>.<project>-dispatch.plist`):
- **`ProgramArguments`:** `/usr/bin/python3 ~/agents/bin/dispatch.py`
- **Timing:** `StartInterval` 300, `RunAtLoad` true
- **`EnvironmentVariables`:** a full `PATH` including `~/.local/bin`, where `agy`, `claude` (and any extra builder CLI) live (launchd's default PATH is minimal), plus optional `AG_MODELS` / `CC_MODEL` / `IGNORE_<POOL>_QUOTA`
- **Output:** `StandardOutPath` / `StandardErrorPath` → `~/agents/logs/launchd.{out,err}`

Viewer plist: the same shape, running `~/agents/logview-venv/bin/python ~/agents/bin/logview.py` with `PORT=5050` and `KeepAlive` true.

Optional mirror-sync plist: the same shape, running a small fetch-and-push script on an interval, if you keep a mirror (2.1).

```sh
launchctl bootout   gui/$(id -u)/<label> 2>/dev/null
launchctl bootstrap gui/$(id -u) ~/Library/LaunchAgents/<label>.plist
launchctl kickstart -k gui/$(id -u)/<label>        # restart
```

---

## 8. Operator controls (put these in the project's PIPELINE.md)

| What | How |
|---|---|
| Pause (the current run finishes; nothing new starts) | `touch ~/agents/state/PAUSED` / `rm` to resume |
| Auto-merge on / off | `touch` / `rm ~/agents/state/AUTO_MERGE` |
| Status of every brief | `python3 ~/agents/bin/dispatch.py status` |
| What the next tick would do | `dispatch.py --dry-run` |
| Retry from scratch | `dispatch.py reset <id>` |
| Salvage leftovers and retry | `dispatch.py resume <id>` |
| Stop AG mid-run | pause, then `pkill -f "agy -p"` (treated as an early stop: salvaged) |
| Hold a brief so it won't start | set its Status back to `draft` on main |
| Change builder order | edit `AG_MODELS` in the plist, then bootout/bootstrap |
| Stop the pipeline entirely | `launchctl bootout gui/$(id -u)/<dispatch label>` |
| AG quota | `cd ~/agents/quota-check && agy -p /usage` |

**Interactive Claude's standing duties** (put these in the project's CLAUDE.md):
- Re-copy `dispatch.py` and `PIPELINE.md` to `~/agents/bin/` after editing them.
- Pause before direct work that touches files an upcoming brief will change, to avoid merge collisions.
- Never work in the pipeline clones.
- Keep PIPELINE.md current.
- If it's the docs owner: update the design docs after each merge (4.5).

---

## 9. Lessons learned (bugs we actually hit)

| Symptom | Cause | Fix |
|---|---|---|
| A finished run reported BLOCKED | The result parser matched the prompt's own "RESULT: DONE or RESULT: BLOCKED", echoed by AG's compaction summary | Take the last RESULT line after stripping the instruction text |
| The fixes brief never started; the dispatcher sat idle | The reviewer gave it `Depends on: <parent>`; the parent can't be done before its fixes | No Depends line on fixes briefs, plus the dispatcher ignores self-prefix dependencies |
| A third fixes round left the original brief "ready" | Auto-merge only marked `<id>` and `<id minus -fixes>` done | Mark every brief whose Branch matches |
| An agent waited "up to 6h" | A scratch script never exited | The watchdog, plus a must-quit-and-cap rule for probes |
| Runs ended instantly | A denied command: a heredoc, a `>` redirect, multi-line `-m`, an unquoted glob | The shell rules in AGENTS.md, repeated in the resume prompt; salvage and retry |
| The review was committed but missing on main | The git host was unreachable from the headless reviewer | The dispatcher does `pull --rebase` and push after every review |
| Auto-merge stopped on conflicts | AG commits its own brief Status edits | Briefs-only conflicts take main's copy |
| The service couldn't read files | macOS privacy protection on `~/Documents` | Install everything the services need into `~/agents/` |
| Flaky integration tests when the machine was busy | Frame-counting tests at variable frame rates | Fixed-timestep flag *(example: Godot `--fixed-fps 60`)* in the runner |
| The builder "passed" with weak tests | Tests didn't exercise the real path | Briefs demand real-input integration tests; the reviewer probes independently |
| All retries burned at once | A quota wall right after the first failure | Backoff of 10 min, then 45 min; at most 3 attempts |
| A new builder's first real run failed in 0 s | Its disabled-tools list removed one tool that others depend on, so the CLI refused to start a session. The trial runs had used a hand-typed command, not the pipeline's | Smoke-test the **exact** command line the dispatcher builds (call `builder_cmd()`) in a scratch repo before the first real brief. Treat a builder that can't start like a quota wall: park it and fall over to the next one |
| A review 'FAILED' with no review file, after a clean run | The headless reviewer started the test suite as a background task and ended its turn to wait; headless sessions end with the turn, so the task was killed and nothing was written | Tell the reviewer (and any headless agent) to run every command in the foreground and never use background tasks |
| A 5-hour quota wall used up a retry in 0 s | A 429 was treated like any early stop | Detect `RESOURCE_EXHAUSTED` / "quota reached", parse agy's "Resets in 1h2m3s", and resume after it plus 2 min **without using an attempt** (`quota_retries` in the state). Then go further: keep an ordered `AG_MODELS` list with one model per pool. Before each run, check quota and use the first model whose pool has some; on a 429, fall over to the next pool about 2 min later, and wait for the earliest reset only when every pool is out |
| Commits credited a single AI vendor | The headless CLI appended a `Co-authored-by:` trailer by default, though several agents did the work | Keep attribution neutral: disable the trailer where the CLI allows, tell agents to omit AI co-author lines, strip them in the dispatcher's own commits |
| Overnight order wasn't what the user expected, and quota ran out mid-queue | Several briefs were marked ready at once; CHANGES produced fixes briefs that jumped the queue | Prefer one brief 100% done before readying the next; use `Depends on:` when queueing; hold anything that shouldn't run unattended as `draft` |
| A merged branch kept getting more fixes rounds | The user merged a CHANGES review (nothing blocking), but leftovers went into another `-fixes` on the same branch | Merge, mark the brief and its fixes done, and put leftovers in a **new numbered brief** on a new branch |
| Every brief after a crash refused to start: "dirty tree" | A crashed engine/import run left untracked generated files (Godot `*.import` *(example)*) in the builder clone | `sync_main` deletes untracked files matching an explicit generated-junk pattern before the dirty-tree check |
| Duplicate or conflicting design-doc entries | Builders (and sometimes the docs owner) both wrote the doc update for the same merge | One docs owner (interactive Claude or a docs agent) updates docs after merges; builders put doc notes in the handoff instead; the owner checks for an existing entry first |
| A docs repo silently forked after a host move | One sibling clone still pointed at the old remote | Repoint `origin` in every clone, including siblings, and compare SHAs on both ends |

---

## 10. Build order for a new project

1. **Prerequisites:** `agy` (logged in) and `claude` CLIs in `~/.local/bin` (plus any extra builder CLI); a single test command that returns non-zero on failure and has timeouts.
2. **Git host and defaults:** pick the source-of-truth host and point every `origin` at it (2.1); decide whether you want a mirror and how it syncs. Decide whether AUTO_MERGE starts **off** (recommended) or on, and write that down in PIPELINE.md.
3. **Project docs:**
   - **AGENTS.md:** architecture rules, the test command, and the Unattended Runs section (section 5, item 3).
   - **CLAUDE.md:** roles, who drafts briefs, brief format, review steps, pipeline duties, and who owns docs after merges (4.5), or put that last one in the docs agent's instructions.
   - **PIPELINE.md:** the operator guide for this project.
   - **`briefs/` and `briefs/reviews/`.**
4. **Clones:** clone into `~/agents/ag/<project>` and `~/agents/cc/<project>`, plus siblings. Add the pre-push hooks.
5. **Builder permissions:** write agy's `settings.json` (allow, deny, trusted workspaces), and the sandbox profile for any extra builder. Test a harmless headless run by hand and read its log.
6. **The dispatcher:** write `tools/pipeline/dispatch.py` (sections 4–4.5), with prompts adapted to the project's doc names, `AG_MODELS` / `builder_cmd()` for each builder, quota checks per pool, and the `sync_main` junk-clean pattern for this stack. Swap the watchdog's process name for whatever hangs here.
7. **Smoke test:** run each builder's exact `builder_cmd()` output in a scratch repo.
8. **First tick:** copy `dispatch.py` and `PIPELINE.md` to `~/agents/bin/`, run `--dry-run`, then one manual tick with a tiny brief. Read every log. From now on, **re-copy to `~/agents/bin/` after every edit** to either file.
9. **Services:** install the dispatcher's launchd plist, then the log viewer and its plist (bind it to `127.0.0.1` unless you accept that `0.0.0.0` is unauthenticated on the LAN), then the mirror sync if you have one.
10. **Defaults:** if AUTO_MERGE started off, turn it on once a few manual merges have gone cleanly. Keep to one ready brief at a time until the loop has proven itself.
