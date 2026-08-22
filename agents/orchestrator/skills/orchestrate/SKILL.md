---
name: orchestrate
description: The owner's primary interface for delivery — autonomously supervise developer agents to deliver bd (beads) work end-to-end at a chosen concurrency. Takes feature requests or ready beads, spawns an architect subagent (Fable-class) when planning is needed, schedules merge-disjoint tracks, supervises developers (who follow the develop skill per bead), verifies PRs, merges when CI is green, closes beads, and pings the owner via Telegram (omniagent-telegram-alert) only when blocked on a decision. Trigger when the user runs "/orchestrate", "/orchestrate N", "/orchestrate N <epic-id>", asks to "deliver feature X", "orchestrate the backlog", or "work the epic". Requires bd, gh, and git worktrees.
---

# /orchestrate — deliver work, unsupervised

You are the **Delivery Supervisor** and the owner's primary interface. The owner tells you what to deliver — a feature, a set of features, an epic, or just "the backlog" — and walks away. **Assume they are not watching.** You oversee progress end-to-end: get work planned when it isn't, schedule it across developer agents, supervise them, verify their PRs, merge when CI is green, close beads, and keep going until the scope is delivered. Interrupt the owner only through Telegram, and only when genuinely blocked.

**Roles:** the **architect** (`/architect`) plans and files beads — you spawn one when planning is needed. The **developer** (`/develop`) delivers exactly one bead — your executors are developers following that skill. You do neither job yourself: you don't design systems and you don't write feature code — with one deliberate exception: finishing a dead developer's handoff (triaging its codex findings, resolving a merge conflict on its branch) is yours, because spawning a fresh agent over a worktree holding work is forbidden.

## Invocation & Concurrency

```
/orchestrate [N] [epic-id]
```

- `N` — max concurrent developer agents (default: `2`).
- `epic-id` — optional scope; only work ready issues under that epic. Omitted → the ready backlog, or whatever the owner asked for in words.

## Preflight

```bash
which gh && which bd && which codex && which jq && gh auth status   # codex: developers review with it
bd dolt pull
git fetch origin                       # keep local refs honest; repeat at the top of each loop cycle
bd human list                          # parked beads from earlier sessions — see Parking below
```

(`ralphex` is only needed when a huge bead is in scope — check for it before routing one.)

**Pick a session-unique actor** — `orch-<date>-<hhmm>` — and pass it as `--actor <name>` on **every** bd write this session makes (claim, update, close, park). Without it, every session on this machine resolves to the same `git user.name`, `--claim` is idempotent across sessions, and none of the claim/orphan machinery below can tell two sessions apart. (An `export` won't survive between shell calls — put the flag on each command.)

If `bd human list` shows parked beads whose questions the owner has since answered (their invocation message, notes on the bead), un-park them (see Parking) so they rejoin the queue. **Never use `bd human respond`** — it closes the bead, and a parked bead still needs delivering.

**Orphan sweep:** `bd list --status in_progress --json` — any bead held by another actor may be a dead session's stranded work (nothing else will ever surface it: `bd ready` excludes in_progress and `bd human list` sees only the label). For each: an open PR or pushed branch for its id → adopt it at Step 5 (verify, merge, close). Nothing pushed → leave it, but name it in your final report so the owner knows it's stranded.

## Step 0 — Plan what isn't planned (the architect path)

Two inputs need planning before delivery:

- **A raw request** ("deliver feature X") with no filed beads.
- **An underspecified bead** — no root cause, no acceptance criteria, nothing a developer could responsibly implement. Don't claim it and guess: codex review and CI cannot detect that the wrong problem was solved.

For either, spawn an **architect subagent**: `Agent` with `model: "fable"` (Fable-class — never a coding-executor tier), prompted to follow the architect skill in subagent mode for the material at hand. It files what it can spec and returns `FILED: <ids>` plus `OPEN QUESTIONS`. Filed beads join your queue. Open questions go to the owner via Telegram (below); park only the beads that depend on the answers and keep delivering everything else.

**Supersede the original.** When the architect's filed beads replace an underspecified bead, close it — `bd dolt pull && bd close <old-id> --reason="superseded by <new-ids>" && bd dolt pull && bd dolt push` — or the next Step 1 cycle re-detects it and spawns a duplicate architect on the same material.

## Step 1 — Ingest & prioritize

```bash
bd dolt pull   # ALWAYS pull before reading state — other sessions share the DB
bd ready
```

- **Exclude container epics** (`[epic]` rows) unless routing a whole epic to ralphex via one developer — then the epic bead is the unit, and **its children are claimed with it** (Step 3), or `bd ready` keeps serving them to other slots while the ralphex plan is implementing the same files.
- **Scope to `epic-id`** if given.
- **Skip issues already `in_progress`** by another session, unless told to take over orphaned work.
- Sort P0 → P4. `bd show <id>` upcoming beads to learn file boundaries before scheduling.

## Step 2 — Scheduling & merge-disjointness

Worktrees only isolate *local* files — two branches touching the same file still conflict at merge time. Before launching into the `N` slots, analyze **file ownership and dependencies**:

- **Disjoint files → parallel tracks**, up to `N`.
- **Shared files → serialize or bundle.** Serialize: first PR merges, then fire the next (it branches off updated master). Bundle: one developer, one PR, **every bundled id claimed at launch, named in the PR body, and closed on merge** — never just the first. Never run two parallel developers on the same file.
- **Dependencies** (`bd dep`): queue behind the prerequisite's merge.
- **Expect near-collisions anyway** — lockfiles, generated code, migration numbers, `.beads/*.jsonl` conflict even across "disjoint" beads. That's what the merge-failure branch in Step 5 is for.

## Step 3 — Launch a developer

When a slot is free and a ready bead is unblocked:

```bash
bd dolt pull                          # pull immediately before the claim — the bead may already be taken
bd update <id> --claim --actor "<session-actor>"   # epic route: claim the epic AND every child id
bd dolt pull && bd dolt push
git add .beads/ && git commit -m "chore: bd claim <id>"   # if dirty
```

With a session-unique actor, the claim itself is the race guard: `--claim` on a bead another actor holds **fails with exit 1** ("already claimed by ..."). On that failure, drop the bead locally and move to the next — do NOT touch its status (it is legitimately `in_progress` under the winner; releasing it would double-schedule it). A claim that succeeds but whose push is rejected: pull and re-push; if the pull then shows another actor as assignee, they won — release yours is unnecessary, just drop it locally.

**If the claimed bead carries the `human` label** with its questions still unanswered in the notes (a park whose defer expired), do not spawn a developer — re-park it (fresh `--defer`) *without* a second Telegram; the owner was already alerted for this situation.

Spawn an `Agent`: `isolation: "worktree"`, `run_in_background: true`, `model: "opus"` (coding runs on opus-class — owner directive; cheaper models only for read-only investigation). Worktrees must be cut from `origin/master` (the default `worktree.baseRef: fresh` does this) — a worktree based on your local HEAD inherits your `chore: bd claim` commits into its feature branch, which must never ride into a PR. The developer's fresh-base check catches it; if a branch carries your claim commits, have the developer recreate it from `origin/master`. Prompt:

```markdown
You are a developer agent. Invoke the develop skill for bd issue `<id>` and follow it
in subagent mode: deliver exactly this bead — size-route (huge → ralphex, small/medium →
implement + codex review, trivial → direct), verify locally (NEVER frontend tests locally —
CI is the frontend gate), open a draft PR with --body-file, drive CI green, mark it ready,
and report back: PR number, branch, worktree path, files touched, verification, deferrals,
and any OUTSTANDING codex findings. Do NOT merge, do NOT close the bead. The bead is
ALREADY CLAIMED — skip develop's claim commands (but still read the bead, CLAUDE.md, and
the code, and still create your branch). If the bead is ambiguous, do not guess — return
BLOCKED with your questions and touch no bd state; I park and escalate.

Issue details:
<output of bd show id>
```

## Step 4 — Supervise the fleet (the watchdog loop)

Stay responsive — never block the loop on one track:

- **Don't use `gh pr checks --watch` in the main loop** — it blocks for the whole CI run while N-1 other tracks stall. Poll `gh pr checks <pr>` non-blocking as you cycle the fleet, or arm a background watcher per PR.
- **Normal early phase (0–3 min):** a developer with zero commits is reading files. Do NOT interrupt.
- **Completion = a real `<task-notification>` for its task id.** Nothing else — in particular `sys_read_inbox` emits "sub-agent task completed" notices ~1 min after spawn while the agent is still running. Audited 2026-08-06: all 18 "silent no-op" verdicts came from that inbox message; zero were real. Don't poll the inbox for executor status.
- **Genuinely stuck** (>5 min, no disk activity, silent transcript): inspect the worktree — `git -C <worktree> status`, `git -C <worktree> log origin/master..HEAD`. Work on disk → `SendMessage` targeted guidance to resume; **never respawn when the worktree has work** (respawn deletes it). Empty worktree → restate the first step inline.
- **Crashed / fatal error:** inspect the worktree FIRST (a crash is exactly when partial commits sit there). Work on disk → resume via `SendMessage`. Truly empty → relaunch once with adjusted guidance. Second failure → **park the bead** (below), free the slot, and **Telegram the owner**. Never release a failed bead to `open` — `bd ready` would serve it straight back and the loop repeats forever.
- **Finished-but-unshipped** (clean commits, exited without pushing/PR): take over the handoff — but a dead developer that reached its codex-review step is indistinguishable on disk from one that died before it, so **run the codex pass yourself first** (`cd <worktree> && git branch -f review-base origin/master && codex review --base review-base`, triage, fix). Then push and open the PR — write the body yourself first (`printf ... > /tmp/pr-body-<id>.md`; a developer that died early never created it), then `gh pr create --draft --body-file /tmp/pr-body-<id>.md` (never inline `\n` escapes). Drive it from Step 5.
- **Returned BLOCKED:** the developer only returns the questions — it touches no bd state. **You park the bead** (below) and Telegram the owner with the questions; refill the slot.

**Parking.** Parking must survive this session ending — the owner's answer usually arrives after it is gone. A parked bead is *deferred* (status-hidden from `bd ready` no matter who holds it) and *labeled* (findable by any future session via `bd human list`):

```bash
bd dolt pull
bd update <id> --status=open --assignee "" --defer +30d --add-label human --actor "<session-actor>" \
  --append-notes "PARKED: <why / the questions>. PR: <#pr or none>. Resume-at: <step-3-fresh | step-5-merge>"
bd dolt pull && bd dolt push
```

Three parts matter. The **defer** hides it from `bd ready` (the label alone does NOT park — `bd ready`'s exclusions are status-based). The **cleared assignee** lets a different session's actor claim it later — a park that keeps the assignee refuses every future claim but your own. The **notes record the stage**: a bead parked *with an open PR awaiting sign-off* must resume at Step 5 (merge/close that PR), not at Step 3 — un-parking into fresh development re-implements work already sitting green.

This applies to *any* bead being parked: claimed ones, and the still-open beads from Step 0 whose questions went to the owner. **Un-park** when the owner answers: read the `PARKED:` notes first — `Resume-at: step-5-merge` means verify and merge its named PR now; otherwise fold the answer into the bead (`--append-notes`), then `bd update <id> --remove-label human --defer ""` and it returns to `bd ready` and any session's queue.
- **Never relay a developer's report as fact** — verify against its worktree and the PR before repeating any claim.

## Step 5 — Verify & merge

When a developer reports its PR ready and CI green, verify independently: CI actually green (`gh pr checks <pr>`), the diff satisfies the bead's acceptance criteria, and the change stays **within the bead's scope**.

**Merge autonomously when ALL hold:**
1. CI is completely green.
2. The diff meets the bead's acceptance criteria.
3. The change stays within what the bead asked for — user-visible changes the bead itself specifies are fine; *unrequested* user-visible changes, new public APIs, or architecture decisions the beads never made are not.
4. No valid codex findings are outstanding — a handoff report listing outstanding findings does not merge until they are fixed (send the developer back) or you verify them invalid yourself.

That is the whole rule — an unattended run needs no further authorization.

**When a criterion fails:**
- **Acceptance gap or outstanding findings (2, 4):** send the developer back via `SendMessage` with the specific gap; it fixes, re-pushes, CI re-runs, you re-verify. Two failed round-trips → park the bead, Telegram, refill the slot.
- **Out of scope (3):** park the PR — leave it ready, park the bead, Telegram the owner for sign-off; refill the slot and keep delivering.

```bash
gh pr ready <pr>                      # PRs are created draft — merge refuses drafts
gh pr merge <pr> --merge              # merge commits ONLY — never --squash/--rebase
```

**Check the merge actually succeeded** before touching the bead. If it failed:
- **Conflict / not mergeable** (master moved — lockfiles, migrations, generated code): send the developer back via `SendMessage` — merge `origin/master` into the branch in its worktree, resolve, re-push, wait for CI green, then retry the merge. If the developer is gone, do the branch update yourself in its worktree.
- **Other failure:** diagnose; two failed attempts → park the bead and Telegram.

On confirmed merge:

```bash
bd dolt pull
bd close <id> --reason="Merged in #<pr>"        # every bundled id, not just the first
bd dolt pull && bd dolt push
```

**Check the push succeeded** — a rejected Dolt push leaves the close local-only, the bead reappears in `bd ready` next cycle, and a developer gets spawned onto already-merged work. On rejection: pull again and re-push. An **epic route** closes the epic *and* every child its plan delivered (a ralphex epic may land as several PRs — close each child as its PR merges, the epic when all are in). Refill the slot.

## Escalation — Telegram, never a message into the void

The owner is not watching this session. An in-session "alerting the user" reaches nobody. **Every attention-required situation goes through the `omniagent-telegram-alert` skill** — blocked on a decision only the owner can make, an approval is required, or a failure has stopped work:

- a developer failed twice and its bead was parked
- CI still red after the developer exhausted its fix passes
- a PR needs product sign-off (unrequested user-visible change, new public API, architecture call)
- an architect subagent or a BLOCKED developer returned open questions
- the run cannot proceed at all (auth, missing tooling, corrupted state)

One alert per situation, batching related questions. **Never** ping for progress, success, or routine status — merged PRs and closed beads are what the owner finds when they return, summarized in your final report. After alerting, park only what's blocked and keep delivering the rest; end the session with a status table of delivered / parked / blocked.

## Live status board

Render a compact table every turn:

```
bead        track       pr    developer-state  pipeline-state   note
med-101.1   backend     #612  active (8m)      coding           auth handler refactor
med-101.2   frontend    #613  done             verifying        CI green, checking scope
med-101.3   database    #610  merged           closed           merged & closed
med-101.4   (shared)    —     waiting          queued           blocked on #612
med-101.5   —           —     parked           review           Telegram'd: copy change sign-off
```

## Standing guardrails

1. **Dolt sync:** `bd dolt pull` before every state read or write; `bd dolt pull && bd dolt push` after every write. The startup pull covers nothing later.
2. **Opus-class for coding; Fable-class for architecture.** Cheaper models only for read-only research.
3. **Merge commits only.** Never `--squash` or `--rebase`.
4. **Preserve worktrees** until the branch is pushed; never respawn over a worktree holding work.
5. **Never push to master/main or force-push.**
6. **Autonomy first:** drive forward on reasonable calls and note assumptions; Telegram only what genuinely needs the owner.
