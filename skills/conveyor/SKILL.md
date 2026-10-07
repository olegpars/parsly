---
name: conveyor
description: >-
  Use when the user tells you to drive an issue map or epic to completion with subagents -- "drive the map <repo>#N", "drive the map to the end", "run the conveyor on #N", "conveyor #N", "finish the epic #N". One dispatcher session takes an issue with sub-issues and drives it to its Destination: the head session executes tickets itself or through subagents, and a model of a different family accepts each result. NOT for charting a new map and NOT for the manual one-ticket-per-session flow; deep topic research is digger, a one-shot expert council is dreamteam. Slash form is "/parsenal:conveyor" (bare "/conveyor" only applies to a manual single-skill install, see the PARSENAL README). Part of the PARSENAL skill system. User-invoked only -- do not auto-trigger.
disable-model-invocation: true
---

# Conveyor -- drive an issue map to Done

## Overview

A map-charting skill (wayfinder-style, for example Matt Pocock's wayfinder in [mattpocock/skills](https://github.com/mattpocock/skills)) charts the map and stops. Conveyor continues from there: one session takes an issue with sub-issues and drives it to its Destination. The head session executes tickets itself or through subagents; every result is accepted by a model of a different family; the head does the GitHub wiring itself.

Why this shape (hard-won lesson): an earlier scheme -- a cheap dispatcher model per wave, an envelope-and-receipt protocol, the dispatcher executing and accepting its own tickets -- missed every seeded defect in a test wave. A strong head executing the work plus a cross-family verifier caught more defects with fewer agents.

## Input

- Launch is a human action: the user runs `/parsenal:conveyor <repo>#N` in a fresh session, or clicks a handoff chip from a previous chain session (see HITL). The skill is user-invoked only (`disable-model-invocation: true`).
- Address the map as `<owner>/<repo>#N`. If no repo is given, use the repo of the current working directory; ask if that is ambiguous.
- Maps charted in the wayfinder format (`wayfinder:map` label) are first-class: read the Destination, blocked-by links, and ticket types from the map. The ticket type comes from its label `wayfinder:research|prototype|grilling|task`, not from the ticket body; write the type into the executor's prompt explicitly.
- Tickets are GitHub sub-issues only. A task list in the map body and a local-markdown tracker are not supported.
- Blocked-by: native GitHub issue dependencies; fallback -- a `Blocked by: #n` line in the ticket body.
- Labels `wip:*` and `flag:human-only` are a Conveyor extension to the wayfinder format: the format itself carries only `wayfinder:*` labels.
- For any other issue with sub-issues, fill only the missing essentials before starting: ask one question for the Destination and classify tickets as AFK or human-only. Ask nothing else.

## Session in a worktree

A chain session may live in a git worktree on its own branch; that is normal, not a failure.

- The session root is `git rev-parse --show-toplevel` at start. All repo paths for subagents and external hands are absolute paths under that root; do not write to the main checkout from a worktree session.
- Before the first commit, run `git fetch origin` and check that the branch was cut from `origin/main` and carries no commits missing from it: `git merge-base --is-ancestor HEAD origin/main` must exit 0. Exit code 1 means the branch was cut from a dirty local main and carries foreign commits: there are no own commits yet, so recreate the branch from `origin/main` and report it.
- Commit only your own files: `git commit -- <files>`. Commits stay on the session branch until the push at the finale (see Gates).

## The loop (the core)

0. **Start of run -- the "conveyor running" mark.** The first GitHub action of every chain session (first run, continuation, restart), before the starting plan: put the label `wip:conveyor` on the map (description "conveyor: run in progress"; create it if missing) and post the comment `▶ conveyor-start <RUN_ID> <ISO timestamp>`. If the label is already there (a continuing chain), do not duplicate it -- only post the comment with the new RUN_ID. The finale removes the label; while the chain is alive, it stays. Why: anyone browsing the map must see from the label alone whether a conveyor is running on it.
1. **Frontier:** open sub-issues with no open blocked-by links, not claimed, and without `flag:human-only`.
2. **Starting plan (only before the first wave):** send the user one message with the frontier, the AFK/HITL classification, and who executes each ticket (the head or a subagent). This is a briefing, not a gate: the command to drive the map has already been given, so do not wait for confirmation.
3. **Executing a wave.** Claim every ticket before work (see Claims).
   - One ticket, or tickets that edit the same files -- the head executes them itself, sequentially.
   - 2+ independent tickets with no file overlap -- one subagent per ticket, in parallel. The cap is 4 at once; the head adjusts it to the actual limits.
   - Subagent prompt: repo and ticket number; "the spec is the ticket body, read it with `gh issue view`"; the ticket type and its resolution standard; boundaries -- do not close the ticket, do not write `closes #N`, do not commit (parallel subagents share one git index; the head commits per ticket with `git commit -- <files>`), a permission or sandbox block means stop and report; the full report goes to a file in the scratchpad, and only a digest of at most 15 lines comes back. For gated GitHub actions, include the line "user approval: <quote>", otherwise an auto-mode classifier may block the dispatch.
   - Resolution standard by type: research -- facts with primary-source pointers, with summaries longer than ~10 lines saved as a repository file and linked from a comment; prototype -- a cheap artifact linked from the ticket (Conveyor treats prototype as AFK and executes it like any other ticket, by the rules above; in the wayfinder format prototype is HITL); task -- the result in a comment.
   - External hands write code only when the head decides so -- a lot of code, or to offload the session's limits. Then the spec goes as a file, and acceptance is done by a fresh-context subagent with the same prompt as in step 4, because the usual verifier family wrote the code.
   - Subagent discipline: a subagent verifies its own result synchronously; no background monitors and no "waiting". A subagent that exits early is restarted, not waited for. The head also waits for the verifier synchronously.
4. **Acceptance -- a model of a different family, on every ticket.** The executor never accepts its own work. The head writes the acceptance prompt to a file `<scratchpad>/accept-<N>.md`:
   - the ticket body and its DoD (no DoD -- check against the ticket body);
   - what was done: the SHA and the diff, the result file path, or the comment text;
   - check axes: defects and edge cases, each DoD item, the environment (OS, encodings, long-running processes), checks that cannot fail;
   - the verdict frame, verbatim from the first acceptance: "P1/P2 -- only what breaks a DoD item or the ticket spec on a plausible input; exotic inputs and anything outside the DoD are P3 and do not fail the verdict";
   - the answer format: "accepted / not accepted", then defects with file:line and severity P1 (DoD not met, data corruption) / P2 (must fix) / P3 (minor), then each DoD item "passed / failed / unverifiable (why)".

   Run it synchronously (a few minutes; allow a 10-minute timeout); acceptances of different tickets may run as parallel synchronous calls. The verifier gets a read-only sandbox and may run commands that do not write to disk; a test that needs writes (build, cache, database) is run by the executor, and the raw output goes into the acceptance prompt. Example with Codex CLI as the verifier (reasoning effort `high`; higher levels cost time without adding findings):

   ```bash
   codex exec --skip-git-repo-check -s read-only -c model_reasoning_effort=high \
     -C <repo root> -o <scratchpad>/accept-<N>.out.md - < <scratchpad>/accept-<N>.md
   ```

   - The full answer goes into the ticket as a comment `✔ acceptance` / `✖ acceptance` (a long one inside `<details>`).
   - The head triages the findings: a real defect -- rework by the same executor and a repeat acceptance. P3, pedantic, or outside the DoD -- one line in the comment "not taken for fixing: <why>".
   - **At most two full acceptances per ticket** (hard-won rule: without a limit, a verifier on an open-ended validator finds new rare cases every round). After the second ✖:
     - only P2/P3 remain -- close the ticket; P2 go into one follow-up ticket `[task] follow-ups #N` on the map (a sub-issue, no blocked-by, linking the ✖ comment), P3 into a "not taken for fixing" line;
     - a P1 remains -- P2/P3 go to the follow-up and lines as above; for the P1, a fresh executor and a third, last acceptance of those P1 only. The original P1 all "passed" -- close the ticket, anything new from the answer goes into the same follow-up; any original P1 failed -- label `blocked`, a diagnosis to the user, the conveyor moves on.
     - There is no fourth acceptance. "The remarks are narrow, I'll fix them myself and run it once more" is exactly the endless loop; the follow-up lives in a ticket, not in an extra round.
   - The verdict "unverifiable" is legal: a web fact in a research ticket that the verifier cannot open is marked so.
   - Verifier unavailable (not logged in, limit, outage) -- the same acceptance by a fresh-context subagent, with the note "verifier unavailable: <verbatim error>, accepted by a fresh-context subagent" in the comment. This is the only allowed exception to cross-family acceptance; once the verifier is back, re-verify those tickets in the finale.
   - Nobody accepts a grilling ticket: it is the user's decision.
5. **Decisions:** the head writes the conclusions from the wave itself. Grills in the queue -- hand them to a separate session and end the turn (see HITL); no grills -- the next wave in this same session.
6. **Wiring + map + self-check:** close accepted tickets, remove claims, and update blocked-by links. Maintain the map after every accepted ticket (without it, the map stops being a decision index): add a one-line gist with a link under `## Decisions so far`; if the result cleared uncertainty, create new tickets before wiring them in and clean up `Not yet specified`; if a decision invalidated other tickets, update or close them with a note; if a ticket proved beyond the Destination, move it to `Out of scope`. Then the mandatory self-check of the GitHub wiring: reread issue numbers, repos, and blocked-by links and confirm that everything landed in the TARGET repo (hard-won rule: in one real run, blocked-by links silently landed in unrelated public repos).
7. **Progress line in chat:** what closed, what became unblocked, what is in the question queue.
8. Repeat while the frontier is not empty. An empty frontier while tickets remain open means only human-only or blocked tickets remain -- move to the finale.

The journal lives in the ticket: `▶ dispatch` (names the executor, the head when it works itself), `✔/✖ acceptance`, `🔁 rework`, `⛔ blocked`; the head posts it. Red flags -- the old scheme creeping back: a separate cheap dispatcher per wave, an envelope with a receipt, a verifier of the same model as the executor, acceptance "from the digest" without running the verifier.

## Claims and restart

- A claim is an assignee plus the marker comment `conveyor-claim <run-id> <timestamp ISO>` in the ticket.
- A ticket with an assignee but no `conveyor-claim` comment is taken by someone else (for example a manual map session) -- treat it as claimed and skip it.
- At the start of a run, remove stale claims made with your own pattern (a `conveyor-claim` comment from a previous run-id). A repeated request to drive the same map in a fresh session is idempotent: state lives in GitHub, work is not duplicated.
- **Restart instead of compaction.** As the head's context approaches its auto-compact ceiling (~300k tokens in current Claude Code), or after a large phase of the run closes, do not compact. Bring claims and the journal in GitHub current (remove your stale claims, update `wip:*` labels, finish the journal), wrap up the session (see HITL), and hand off the continuation the same way as a grill (see HITL). Idempotency is guaranteed by the GitHub state, so no context reconstruction is needed. A spoken "continue in a new session" instead of a handoff does not count: the chain keeps the continuation point, not the user's memory. Global auto-compaction is insurance for a missed restart, not the normal way to continue a run.

## HITL -- grills in a fresh session

**Never run a grill in the run session.** When a wave closes and grills remain on the frontier, finish the wave (wiring of accepted tickets, map, journal, removed claims), hand the grills to a fresh session, and end the turn. The run session carries the whole run in its context; a grill needs only the map, the ticket body, and the reference notes. "The user is right here, it's just one question" is the turn-saving that costs the most: a grill is rarely alone, and the context grows until the end of the run.

- **Session handoff, exactly one at a time.** In Claude Code Desktop -- a `spawn_task` chip whose prompt starts with `conveyor-start <repo>#N` (or `conveyor-start <repo>#N grill #M`); elsewhere -- print a one-line command for a fresh session (`/parsenal:conveyor <repo>#N`, or `/parsenal:conveyor <repo>#N grill #M` for a grill session; the `grill #M` argument tells the new session to run the grill queue instead of a wave). A slash command inside a chip prompt does not run and the skill is user-invoked only, so the chip prompt tells the new session to read this skill's `SKILL.md` from the installed PARSENAL plugin and follow it. Two pending handoffs break the invariant "exactly one next step ahead". The prompt carries the map number, the grill ticket number, links to the reference notes, and an explicit ban on carrying over the run history. Do not batch questions into the handoff: grills depend on each other through blocked-by.
- **The grill session** runs the whole queue in rounds: all independent open questions of the frontier at once (independent by blocked-by), each with a recommendation; an answer that unlocks a new question goes into the next round. Decisions land in GitHub: a comment in the ticket, closure, a gist line under `## Decisions so far`. Issues are the only carrier of decisions; a cross-session message is a wake-up, not a transport.
- **The end of a grill is a continuation handoff.** When the queue is empty, wrap up the session and hand off the continuation. The user does not have to remember where to continue: the chain itself serves the next button or command. Answering "continue in a new session whenever you like" instead of a handoff breaks this rule.
- **Every chain session ends with a wrap-up** -- commit own work, record leftovers on the map. The handoff is the last action, after the wrap-up, and points at the settled state.
- Order within a round: first the grill that unblocks more tickets, on a tie the lower number; a grill without the needed facts waits for the next wave.
- The user is away (night, AFK) -- do not wait: keep running waves while unblocked non-grill tickets exist; when only grills remain, write the queue as a comment on the map (a question plus a recommendation for each) and leave the handoff; it starts the grill session when the user is back.
- The user did not pick up the handoff -- nothing is lost: the queue, claims, and decisions live in issues, and any new run on the map is idempotent.
- The queue lives on the map; no separate HITL file.

## Gates

- Do without asking: GitHub operations (comments, ticket closure, map-body edits), repository files, and local commits.
- **Any push needs one explicit yes from the user, at the finale**, never earlier.
- External, irreversible, or paid actions (publication, outbound messages, spending) are for the user only. Never perform them in the background: add a note to the queue and wait.
- Do not touch `flag:human-only` tickets or actions marked "you publish"; only highlight them in the finale.

## Models

| Role | Who |
|---|---|
| Head: plan, decisions, GitHub wiring, claims, journal | the strongest model in the session |
| Ticket executor | the head itself or a subagent; external hands only when the head decides (step 3) |
| Acceptance | a model of a different family (for example Codex CLI, reasoning effort high, read-only sandbox); unavailable -- a fresh-context subagent, noted in the comment |

- Do not use small models as executor or verifier: in a test wave, a small-model executor with a small-model verifier missed every seeded defect.
- Put the executor-to-ticket mapping in the starting plan; the user may revise it.

## Finale

When the Destination is reached and the map is heading to closure: post the summary comment `✔ conveyor-final <RUN_ID> <ISO timestamp>` on the map (what was done, what remains human-only, the question queue), remove the `wip:conveyor` label, give the same summary in chat, and propose the push (gate: one explicit yes). Then wrap up the session; there is no continuation handoff at the finale.

## v1 boundaries

- No headless/night runner and no automatic cron: Conveyor lives while the session window is open.
- Do not modify a map-charting skill or invoke one to chart a map. "Continue the map #N" belongs to the manual one-ticket-per-session flow, not Conveyor.
- Do not make product decisions inside tickets; add them to the queue as grilling questions.
