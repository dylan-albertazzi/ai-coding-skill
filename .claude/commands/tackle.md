---
description: "Tackle a Linear ticket end-to-end: worktree setup, plan, build, local review loop, preview, PR"
argument-hint: "Linear URL/key (PROJ-87), or free-text title"
---

# Tackle Task: $ARGUMENTS

You are orchestrating a full development cycle for ticket **$ARGUMENTS**.

**CRITICAL EXECUTION RULES:**

- **DO NOT STOP between phases.** Move from one phase to the next automatically.
- **DO NOT ask "should I continue?" or "want me to proceed?"** — just proceed.
- **You stop for the user in exactly TWO places, and nowhere else:** plan approval
  (Phase 3) and the final review (Phase 6). Each one exists for a different reason —
  shaping the work, and checking the result. If you want to check in anywhere else,
  the answer is to verify harder, not to ask.
- If something fails (type-check, lint, test, build), fix it yourself and retry. Do not ask the user what to do.
- Use subagents liberally to parallelize work and keep the main context clean.
- **Name agent types, never models.** Use `subagent_type: "tackle-planner"` and
  `subagent_type: "tackle-implementer"`; each definition pins its own model (Fable and
  Sonnet respectively) in `.claude/agents/`. Do NOT pass a `model` override — it takes
  precedence over the definition and silently defeats the point. If routing is wrong,
  fix the definition, not the call site.
- Mechanical steps — fetching the ticket, running setup, collecting output — can use
  `model: "haiku"` directly. Don't spend a reasoning model on a Linear fetch.
- Never delegate Phase 4e (design self-review) to a cheaper model. Judging a rendered
  screen is the hardest thing in this workflow, not the easiest.
- **QA is not a formality in this workflow — it is one of the main deliverables.**
  Give the same care to "what done looks like" and the QA plan that you give to the
  code. A fix that was never actually run against real data, or that broke
  something adjacent nobody checked, is not done — regardless of how clean the diff
  looks.
- **Write all messages to the user in ASD-STE100 Simplified Technical English.**
  Use short sentences. Put one idea in each sentence. Use plain, common words. Use
  active voice. Do not use jargon, idioms, or long noun strings. Follow this rule in
  every phase, not only in the final handoff.

## Phase 1: Fetch the Ticket

1. Parse `$ARGUMENTS` to determine the input format:
   - **Linear issue** (URL containing `linear.app`, or an issue key like `PROJ-87`): fetch it with the Linear connector's `get_issue`.
   - **Anything else** (free-text title): search Linear (`list_issues`). If multiple matches, list them and ask the user which one. If none, tell the user — and offer to create a Linear issue for it (title, description, Type + Area labels), but **only create it after the user explicitly confirms**.
2. Capture the issue's title, description, labels, state, and URL. If it has no Type or Area label, flag that to the user.
3. Save the ticket title, description/spec, acceptance criteria, and Linear URL. You'll need these for the plan, PR, and ticket close-out.

→ **Immediately proceed to Phase 2.**

## Phase 2: Worktree Setup & Sync

1. Check if you're already in a worktree (the Claude Code desktop app creates one automatically). If `pwd` is inside `.claude/worktrees/`, skip to step 2. Otherwise, use `EnterWorktree` to create one.
2. Run `bash .claude/skills/worktree-setup/setup.sh` — this auto-detects a free port offset, runs `pnpm install`, copies and patches env files, runs `pnpm build`, and generates `.claude/launch.json`.
3. Wait for the setup script to complete. Verify it reports all checks passing.

→ **Immediately proceed to Phase 3.**

## Phase 3: Plan

1. Enter plan mode with `EnterPlanMode`.
2. Spawn **`subagent_type: "tackle-planner"`** with the ticket. Its definition pins
   Fable and restricts it to read-only tools, and it already requires everything below
   — so you are checking its output, not re-specifying the job.
3. **Ask the user the planner's open questions before presenting anything.** This is
   the step that keeps them in the loop: a subagent cannot talk to them, so unresolved
   ambiguity comes back to you as data and _you_ ask. Use `AskUserQuestion`, or the
   **interview** skill when there is a lot to settle.

   An empty `openQuestions` on a non-trivial ticket is a smell — it usually means the
   planner guessed. Read the plan for buried assumptions and ask about those instead of
   taking the empty list at face value.

4. Check the plan contains a non-empty **Architecture & Principles Check** section,
   a non-empty **What done looks like** section, a non-empty **QA plan** section,
   and, for frontend tickets, a non-empty **Design intent** section —
   `tackle-planner`'s own spec defines what belongs in each. Missing or thin ⇒ send
   the planner back; do not write any of these sections yourself from guesswork.
5. Write the plan to `tasks/todo.md` with checkable items.
6. If requirements are ambiguous, use the **interview** skill to clarify with the user.
7. **Present the What done looks like, QA plan, and Design intent sections inside
   `ExitPlanMode` itself**, not only in `tasks/todo.md`. The user approves what they
   can see; a decision that lives in a file they didn't open has not been approved,
   and reversing it after implementation is exactly the rework this phase exists to
   prevent. The approval prompt must state:
   - **what done looks like**, in business terms, so the user can say "no, that's
     not actually the problem" before any code is written
   - **the QA plan**, so the user knows what will be proven true before Phase 6
   - for a frontend ticket, also state in a few lines each:
     - the components being reused (or the justification for a new one)
     - what is visible at rest vs. revealed on interaction
     - anything being deliberately left off the screen
8. **PAUSE HERE: Do not proceed until the user approves the plan.** If they change
   any design decision, update `tasks/todo.md` before implementing.

→ **After approval, immediately proceed to Phase 4.**

## Phase 4: Build & Self-Test

### 4a: Start Dev Servers

1. Read `.claude/launch.json` for this worktree's port assignments.
2. Start the relevant dev servers as background Bash tasks — api-server, plus
   whichever portal the ticket affects, each on its launch.json port. Never run a
   bare `pnpm dev`; it collides with other worktrees.
3. Confirm each is actually serving by loading a page — not by checking that the
   process exists. A dev server that died on a stale package `dist` makes every
   later QA step silently meaningless.
4. Keep them running; the browser gate in 4d needs them.

### 4b: Implement

1. Spawn **`subagent_type: "tackle-implementer"`**, one per coherent slice of work — its
   definition pins Sonnet and already owns CLAUDE.md conventions, design-system
   values, and restraint; you are dispatching and checking its report, not
   re-specifying its job. Hand each the approved plan including its Design intent
   section. If slices touch overlapping files, run them sequentially or pass
   `isolation: "worktree"`; two agents editing one file overwrite each other.
2. Mark tasks complete in `tasks/todo.md` as each slice reports back. If a report
   names a departure from the plan or a hex/pixel value outside the design skill,
   send that slice back rather than accepting it.

### 4c: Verify — Code Quality

```bash
scripts/qa/verify.sh
```

One command: type-check, lint and tests scoped to the packages your diff touches,
plus the browser route sweep when frontend files changed. Exits non-zero if any gate
fails. Do not run the individual `pnpm` commands instead — a single entry point is
what makes a skipped gate visible as a missing line rather than as nothing at all.

Fix and re-run until it exits 0. Keep its output; Phase 7 (PR) hands it over as evidence.

**If the diff touches anything under `apps/web/`**, also run
**web-design-guidelines** against the files you changed — not the whole repo.
Fix the accessibility and form findings. Deliberately not fixing one is fine —
say so and why in the report.

### 4d: Verify — Self-Test in the Browser

Use **agent-browser**, not the claude-in-chrome tools: it runs headless in an
isolated session instead of driving the user's real Chrome, which is also logged
into production.

1. **Run the route gate. This is a hard gate.**

   ```bash
   scripts/qa/sweep.sh --portal back-office
   ```

   With no arguments it sweeps **only the routes your diff can reach** and reads
   this worktree's launch.json port. That is the normal case and takes a few
   seconds. It must exit 0 before you continue.

   - **Do not use `--all`** for routine work. Reserve it for changes to a shared
     layout or something under `packages/ui-*`, where "what can this reach" is
     effectively everything.
   - If it reports a route is missing from `scripts/qa/routes.json`, add it —
     with a `waitFor` selector (or `waitForText` for card/kanban pages) and an
     `expect` block. Those assertions are the only thing that catches a crash
     swallowed by an error boundary: without them a broken page still answers 200.
   - Set `expect` values from what the page actually renders, not from what you
     assume it renders. Check the snapshot in `.qa/<portal>/<route>.snapshot.txt`.
   - If a failure is genuinely benign, add a justified pattern to
     `scripts/qa/noise-allowlist.txt`, or `noise-allowlist.local.txt` when it only
     fails because a local box has no production credentials.
   - Warnings (`proxy-bypass`, off-origin requests) don't fail the gate, but fix
     any your own diff introduced.

2. **Exercise the actual feature** — the gate proves pages render, not that your
   change works. Batch the flow into one invocation:

   ```bash
   agent-browser --session "tackle-$TICKET" --allowed-domains "localhost,127.0.0.1" \
     batch --bail "open <url>" "wait '<selector>'" "snapshot -i -c" \
     "click @e12" "wait 1000" "get text '.result'"
   ```

   Use `snapshot -i -c` for element refs, `find role|text|label` for stable
   targeting, and `console` / `errors` / `network requests` to check for failures.
   Never `get text body` on a page that might be erroring — it dumps the whole
   `__NEXT_DATA__` blob.

3. **If the change touches shared UI**, diff against `main`:
   `scripts/qa/diff-vs-baseline.sh --portal <p> --baseline-url <url> --branch-url <url>`.
   Every route it reports as changed needs a reason; one your diff never touched
   is a regression until proven otherwise.

4. **Hunt for bugs rather than confirming the happy path** — run
   `agent-browser skills get dogfood` against the screens you changed.

5. Fix what you find and re-run. Do not proceed with a red gate.

6. **Prove the fix with before/after evidence — do this yourself, do not hand it to
   the user as an untested checklist.** A checklist of steps you did not run is a
   claim, not proof. For a bug fix, find or make real data that reproduces the bug,
   then show the wrong result and the right result side by side:
   - Find (or seed) a real record that triggers the reported bug. Prefer real dev-DB
     data over invented data — a fabricated case can miss the actual shape of the bug.
   - Capture the "before" state: temporarily revert the fix (comment out the guard,
     checkout the pre-fix file, or run against `main`), reproduce the bug, and save
     a screenshot or the raw API/DB response showing the wrong result.
   - Restore the fix, repeat the same action, and save a screenshot or raw response
     showing the right result.
   - Use whichever evidence gives the highest confidence: UI screenshots for
     user-facing behavior, raw API/DB output for backend-only logic.
   - Confirm the working tree matches the committed fix afterward (`git diff` on the
     changed file should be empty) — the revert-and-restore step must not leave stray
     changes behind.
   - If a live before/after repro genuinely is not feasible (e.g. the bug only
     happens in a system you cannot access locally), say so explicitly and explain
     why, instead of silently falling back to a checklist.

7. **Run the plan's regression check.** The plan's QA plan names a specific behavior
   that shares code, data, or a query path with your change — actually exercise it
   against real data and capture the result the same way as the before/after
   evidence (screenshot or raw API/DB output). A regression check that was only
   described and never run is not a regression check. If it turns up a real
   regression, fix it before moving on — do not report a fix that broke something
   else as done.

8. **Write the self-test report** to `tasks/todo.md` under "## Self-Test Results":
   which routes the gate swept and its summary line, the before/after evidence you
   captured and where it is saved, the regression check you ran and its result, any
   allowlist entry you added and why, what `dogfood` found, and anything you
   consciously chose not to fix. Artifacts stay in `.qa/` for Phase 6 and Phase 7.

9. **Show the before/after evidence and the regression-check result to the user in
   chat, in this phase — do not wait for Phase 6.** Saving a screenshot to `.qa/`
   and never displaying it is the same failure as not taking it: the user cannot
   check a claim they cannot see. Use the Read tool on each screenshot so it renders
   inline, or paste the raw before/after API or DB output directly into your
   message. Do this as soon as the evidence exists, in the same turn you produced it.

### 4e: Verify — Design Self-Review (frontend changes only)

**Skip this only if the diff touches no files under `apps/web/`.**

Everything above proves the page works. This step is about whether it is any
good, and it exists because the alternative is the user reviewing it — which is
where their time actually goes.

1. **Look at the screenshots.** The sweep already wrote one per affected route to
   `.qa/<portal>/<route>.png`. Read them with the Read tool. You are reviewing the
   rendered result, not the source — reading code cannot tell you that something
   looks cramped, noisy, or off-brand.
2. **Load `project-frontend-design`** and check the render against it: palette,
   type scale, spacing scale, component patterns.
   2b. **Run the Impeccable pass — scaled to the diff.** Always run the
   deterministic detector on the changed frontend files:

   ```bash
   node .claude/skills/impeccable/scripts/detect.mjs --json <changed apps/web files/dirs>
   ```

   Exit 0 = clean, 2 = findings — triage each finding (fix or justify as a
   false positive in `.qa/design-review.md`). The skill is gitignored (local
   install, see .gitignore); if `.claude/skills/impeccable/` is absent on this
   machine, note that in the design review and rely on the checklist below.

   Additionally, run the full `/impeccable critique` (dual-agent) only when the
   frontend change is **significant**: a new page/route, a new shared component,
   or ≥150 changed lines under `apps/web/**` (`git diff --stat main -- apps/web`).
   For smaller diffs the detector plus this checklist is the whole pass — the
   full critique costs a large multiple of the rest of 4e and duplicates it on
   small changes. Impeccable's suggestions are input, not authority: DESIGN.md
   and existing product patterns win on conflict.

3. **Then work the restraint checklist.** These are the corrections the user has
   had to make repeatedly, so treat each as blocking:

   - **Is any text on screen that doesn't need to be there?** Redundant labels,
     helper text restating the obvious, headings for a single obvious control,
     a column whose value is the same for every row. Cut it.
   - **Does information appear only when it's needed?** Detail that belongs behind
     hover, focus, selection, expansion, or a detail view should not be sitting in
     the default state. Density is not thoroughness.
   - **Could an icon carry this instead of a word?** Prefer an icon with an
     accessible name over a text label where the meaning is unambiguous.
   - **Is the correct control used?** A toggle/switch where state is binary and
     immediate; not a checkbox-plus-Save, not a bespoke button pair. Check whether
     a shared component already does this before shipping a new one.
   - **Is there one clear primary action**, and does visual weight match
     importance? If everything has the same emphasis, nothing does.
   - **Spacing from the scale**, consistent with sibling screens — not eyeballed.
   - **Every state designed**: empty, loading, error, and the long-text/overflow
     case. An empty table with no message is an unfinished screen.

4. **Fix what you find, then re-screenshot and look again.** Iterate until you
   would send it to the user without an apology.
5. **Write `.qa/design-review.md`** — `verify.sh` fails if a rendering file changed
   and this file is missing, too thin, or older than the code it reviewed. It was
   prose before, which meant it could be skipped while the run still reported
   success; the only way to find out was the user noticing the screen looked wrong.

   State: which screenshots you read, what you changed as a result, and any
   judgement call you made that the user may disagree with — with the reason.
   Naming a judgement call is fine; leaving it silent is not. If you changed
   nothing, say what you checked and why it already passed.

→ **Immediately proceed to Phase 5.**

## Phase 5: Local Automated Review Loop

Share a summary of what you built — the branch, the files touched, anything you're
unsure about — then proceed without waiting for a go-ahead.

Not a trust gate — an economics one. **This loop is the most time-intensive step in
the workflow**, so anything the user already knows needs changing should land
_before_ it starts if they choose to say something — but you do not block on that.
There's no PR yet at this point — review is local, diffing against `main`.

Run the **code-review** skill's Phase 2 (Commit & Local Automated
Review Loop). This commits the implementation, runs parallel `codex exec` sessions
locally (`scripts/review/codex-review.sh`), triages valid vs invalid findings,
implements fixes, and re-reviews until a round comes back clean — capped at 3 rounds.
If a codex finding is wrong, note why and leave it; do not implement something you
believe is incorrect just because codex flagged it.

Tell the user: **"Local review loop complete. No more actionable findings. Moving to preview & manual QA."**

→ **Immediately proceed to Phase 6.**

## Phase 6: Preview & Manual QA Handoff

**PAUSE HERE — this is the handoff, and the last of the three stops.**

Open with an **evidence block**, before any prose. The user is checking work, not
reading a report, so lead with what is machine-verified and what is not:

```
verify.sh        4 gates passed (type-check 6s · lint 3s · tests n/a · sweep 22/22 in 22s)
design 4e        <what you changed after looking at the screenshots — .qa/design-review.md>
before/after     <the real bug repro you ran — where the evidence is saved>
exercised        <the actual flows you drove, and what you observed>
not covered      <what you could not verify, and why>
judgement calls  <anything you decided that they may disagree with>
```

If a line would be empty, say so explicitly rather than omitting it — an absent line
reads as "done" when it means "skipped".

1. Ensure the dev servers are still running — verify by loading a page, not by
   checking that the process exists.
2. **Reuse the artifacts from 4d/4e.** Share the existing `.qa/<portal>/*.png`
   screenshots, the before/after evidence from step 4d.6, and the sweep summary
   line. Only re-run the sweep if code changed after 4e; otherwise you are paying
   for a second run to learn what you already know.
3. Provide a **manual QA checklist** tailored to the specific changes:
   - **Pages/flows to test**: List specific URLs and user actions
   - **Happy path**: Step-by-step what to do and what to expect
   - **Edge cases**: Empty states, error states, permission boundaries
   - **Data conditions**: Specific client configs, employee states, or data scenarios to check
   - **Responsive/cross-browser**: Note any concerns
   - **Acceptance criteria check**: Map each acceptance criterion from the Linear issue to what to verify

   Assume 4e already caught the spacing, palette, restraint and wrong-component
   classes of problem — if you are surfacing those here for the first time, 4e was
   skipped. Assume 4d.6 already proved the core bug/feature works with real
   before/after evidence — if the "happy path" step in this checklist is the first
   time anyone has run it, 4d.6 was skipped. Weight this checklist toward what
   neither the sweep nor 4d/4e can check — real provider credentials, multi-step
   flows with side effects, anything requiring judgment about whether the output is
   _correct_ rather than merely present. Don't spend the user's attention
   re-verifying that pages render or that the reported bug is fixed; that's covered.

4. Say: **"Here's your QA checklist — let me know what you find."**

→ **After the user confirms QA looks good, immediately proceed to Phase 7.**

## Phase 7: PR

The code is already committed, locally reviewed (Phase 5), and manually QA'd
(Phase 6) — this phase opens a PR that's ready to merge on arrival, not a draft that
collects fix commits afterward.

1. Push the branch to remote.
2. Create a PR with `gh pr create`:
   - Title format: `[PROJ-XX] <description>` (e.g., `[PROJ-87] Auto-resolve action items`) — use the Linear issue key
   - **Body must be extremely brief — under 4 sentences total.** Three `##` headers,
     nothing else: **Problem** (what was broken), **Solution** (what the change
     does), **How to test**. Use markdown formatting — bullet lists over prose
     where they fit. Skip architecture rationale, trade-off discussion, and
     review/QA narration in the body — the Linear issue link covers deeper
     context, and QA evidence goes in the PR comment (step 3), not the description.
3. **Attach the QA evidence to the PR** rather than only claiming it. Post a comment with the sweep numbers and the screenshots of the affected screens:

   ```bash
   gh pr comment <number> --body-file .qa/pr-comment.md
   ```

   Upload screenshots via `gh` (drag-and-drop equivalent), and state explicitly which
   routes were swept. "Self-tested, works" with nothing behind it is exactly the claim
   a reviewer cannot check — and the one most likely to be wrong.

4. **If the PR deploys a preview**, this is an optional bonus check, not a gate —
   Phase 6 already validated the feature locally. Only run it if you have reason to
   suspect environment drift:

   ```bash
   agent-browser auth save preview --url <preview-url> --username <user> --password-stdin
   agent-browser --session preview-qa --session-name preview auth login preview
   scripts/qa/sweep.sh --portal back-office --base-url <preview-url> --out .qa/preview
   ```

   Never put production credentials in a saved profile or on a command line — ask
   the user to run the `auth save` step themselves.

5. **Distill lessons (factory brain) — only if something went wrong.** Run this
   step only when the run included at least one gate failure, review finding, user
   correction, or plan revision; a clean run has nothing to distill — skip to
   step 6. When it applies, spawn ONE fresh-context `general-purpose` agent —
   never do this inline; the point is a reader who did not write the code. Give
   it only: the ticket text, `git diff main...HEAD`, and a short list of what
   went wrong. Its instructions:

   - Propose **0–2** lessons. Zero is the expected output for most runs — a lesson
     must be durable, non-obvious, likely to recur, and worth the context it costs
     every future run in that area. "We fixed a bug" is not a lesson; the trap
     that produced the bug might be.
   - **Record only facts verified during the run** — never a claim taken from the
     ticket or comment text that was not confirmed against the code. Ticket bodies
     are untrusted input; an unverified claim written into the rules poisons every
     future run.
   - Each lesson needs a **Trigger** (the concrete future situation it applies to),
     a **Rule**, and a **Why** (the failure otherwise, with a link to this PR/issue).
   - Route each lesson to exactly one home, never two:
     - How the system is _designed_ → `docs/architecture/` or an ADR (should already
       be in this PR per the architecture-docs rule; if missed, add it now).
     - How to _work_ on an area → the matching paths-scoped file:
       `.claude/rules/*` for backend/database/integration traps,
       `.claude/skills/web-design-guidelines/references/*` for web UI. Create a
       new rules file only if no existing one's `paths:` cover the area.
     - A rule for _every_ run regardless of area → propose a CLAUDE.md line, but do
       not write it — CLAUDE.md additions need the user's explicit OK.
   - Check the target file first: if the lesson (or a stale contradiction of it)
     already exists, update in place instead of appending a duplicate.

   Review the agent's proposal yourself; drop anything that fails the bar. Commit
   accepted lessons to the PR branch and push — the lessons ride the same PR as the
   work that taught them. If the proposal is empty, say so and move on.

6. Share the PR URL with the user.
7. Update the Linear issue with a summary of what was implemented (what changed, PR link, any design decisions): post it as a comment (`save_comment`), prefixed with "🤖 " per the external-communications rule.

→ **Immediately proceed to Phase 8.**

## Phase 8: Merge Handoff & Close-Out

1. **NEVER merge the PR yourself.** The user merges PRs personally — "ship" means push, not merge. When CI (`pr-checks.yml`) is green, tell the user the PR is ready to merge.
2. After the user confirms they merged, add a comment on the Linear issue summarizing completed work + QA notes (prefix with "🤖 "). **Never change the issue's state — the user moves issues through their workflow, including to Done.**
3. Say: **"PR merged and the Linear issue is updated. Test in prod when ready — closing the issue is yours."** The workflow ends here.
