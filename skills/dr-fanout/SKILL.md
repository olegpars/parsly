---
name: dr-fanout
description: Use when the user wants to fan out a deep research on a topic across browser-based models via Chrome MCP -- one prompt into 4 models OR N prompts x 4 models (ChatGPT + Gemini + Claude + Grok), clicked through in the browser. Triggers -- "/dr-fanout", "run deep research in three/four models", "fan out the research across models", "push this topic through ChatGPT/Gemini/Claude/Grok", "deep research via MCP / in the browser", "fan out deep research". Slash form is "/parsenal:dr-fanout" (bare "/dr-fanout" only applies to a manual single-skill install, see the PARSENAL README). Part of the PARSENAL skill system. User-invoked only -- do not auto-trigger. Not to be confused with `digger` (multi-hour autonomous knowledge-base digging) or a local deep-research harness (no browser). For a one-off lookup use ordinary web search, not this skill.
disable-model-invocation: true
---

# dr-fanout — fan out deep research across 4 browser models (Chrome MCP)

You click deep research through in **ChatGPT + Gemini + Claude + Grok** via Chrome MCP (`mcp__Claude_in_Chrome__*`). Input: **one prompt** (4 runs = triangulation) or **N prompts** (N×4). These are the user's browser subscriptions, NOT API tokens. If the user explicitly asks for "three models", the mode without Grok is valid too.

## Preparation
- `list_connected_browsers` → `tabs_context_mcp{createIfEmpty:true}`. One tab per model (already logged in in Chrome). Research runs **server-side** — navigating the tab to a new chat does NOT stop a running job, so several prompts in one model = the same tab via "new chat".
- **Gemini account guard — before touching the composer:** before typing anything, check by the visible name/email in Gemini's profile menu that the active Google account is the one you intend to use for deep research. A URL of the form `/u/N/app` does not confirm the account. Under a wrong or unidentified account do not enable Deep Research, do not paste the prompt, do not send it, do not create a plan and do not start the research — switch to the right account first. If the right account is not available in the browser or the profile identity is unclear — skip Gemini, continue with the other models and report: `Gemini skipped: wrong/unknown account`.
- Prepare prompts in advance, in English (wider source coverage). **Never paste sensitive data into external models.**

## Pasting a long prompt (critical)
Never type a multi-line prompt via `computer` `type` — `\n` = Enter = premature send. Focus the composer + `javascript_tool`:
`el.focus(); document.execCommand('insertText', false, PROMPT)` — line breaks do not submit. `el` = `div.ProseMirror[contenteditable]` / `div.ql-editor` / `[contenteditable=true]`.

## Flow per model
- **ChatGPT** — `+` → **Deep research** → paste → send → a plan with an auto-countdown appears → click **Start** (or it starts by itself in ~60 s).
- **Gemini** — first pass the **Gemini account guard** above and explicitly confirm the active profile is the one you intend to use; only then `+` → **Deep Research** → paste → send → it generates a plan (~15–25 s) → click **Start research**.
- **Claude** — `+` → **Research** (check: blue magnifier next to `+` / checkmark in the menu) → paste → **Enter** → **"Enable connectors"** dialog → **"Disable all tools" → Confirm** (privacy: do not grant Gmail/Calendar/Drive; web search stays on).
- **Grok** — `grok.com` (a subscription with deep research is required). **There is NO separate DeepSearch button** — the old DeepSearch was merged into model modes: the dropdown on the right of the composer (label = current mode, e.g. "Heavy") → pick **Heavy** ("Team of Experts" · Grok 4.5 — deeper and slower, consumes Heavy quota; **default for fan-out**) or **Expert** ("Thinks hard" · Grok 4.5 — faster and cheaper; only if the user explicitly asks to save Heavy quota). Composer = `div.tiptap.ProseMirror[contenteditable=true]`, paste with the same `insertText` (line breaks do NOT submit — verified). Send = the round arrow button on the right. The "+" menu holds only files/Skills/Connectors — do not look for the research mode there. The chat gets a URL `grok.com/c/<uuid>`; the next prompt goes through "New Chat" in the sidebar.

**Do not confuse:** X (Twitter) search is a separate pass through the headless Grok CLI, not through the browser Grok above — here Grok takes part only as one of the 4 deep-research fan-out models.

## Gotchas (battle-tested)
- **Verify the launch via the DOM, not screenshots** — screenshots time out while the renderer is busy. Markers of a running job: "Searching for sources" / "Researching" / "and counting" / the plan button appearing.
- **Menus re-render** (short → long, ~1–2 s) — wait and click by `find` / `read_page` refs, not by fixed coordinates. The first click on `+` after navigation often yields a tooltip, not the menu — click again.
- **Claude**: the weekly limit is real and does NOT tolerate >1 concurrent deep research. A failed / "Stopped" / "Something went wrong" run **is fixed by a NEW chat, not Retry**. Launch the next prompt only after the previous one has actually started.
- **Grok**: running-job markers in the DOM — "Worked for Ns" / "Ran N searches" / "Searching for…"; a finished answer ends with an "N sources" block. Whether the job keeps running server-side when the tab leaves the chat is NOT verified (for the other three it is verified that it does); until verified, launch the next Grok prompt after the previous one finishes, or in a separate tab.
- Each run takes ~5–30 min; ChatGPT is usually the fastest. A full Grok DR run (Expert on a large prompt) was not exercised live as of 2026-07-27 — a short Expert query with web search was verified (the channel works end-to-end).

## Collecting reports — save as FILES, not links (critical)
A chat link is useless to another session (no access to the authenticated web). **Always save the report TEXT to a file** in the project repo, e.g. `operations/<topic>/reports/<prompt>-<model>-report.md`. The file name must never be `claude.md` or `agents.md`: Windows is case-insensitive and such a file gets auto-picked-up by agents as CLAUDE.md/AGENTS.md instructions. Extraction per platform (battle-tested):
- **Claude** ✅ — the report = an artifact. Click the artifact card (clicking by coordinate is more reliable than by ref) → the text appears in the DOM → take the largest block (`div.flex-1.min-h-0.overflow-y-auto`) → save via **blob download** (`new Blob([text])` + `a.download='X-claude-report.md'` + click, then move the file out of Downloads). The text does NOT pass through the context, verified.
- **ChatGPT** ❌ — the report lives in a cross-origin sandbox iframe (`*.web-sandbox.oaiusercontent.com`): DOM / clipboard / iframe src are closed. Only the report's own **Download** button (PDF) OR a human copy-pastes.
- **Gemini** ❌ — the report lives in an immersive viewer, no text in the DOM (CSS only). Only **Export to Docs** (then read via Drive MCP) OR a human copy-pastes.
- **Grok** ✅ — the answer sits **directly in the DOM** (`<main>`): `get_page_text` grabs it whole, blob download works the same as for Claude. No sandbox iframes. Verified 2026-07-27.

**Bottom line:** the blob method is reliable for Claude and Grok. **Do NOT leave bare URLs in a handoff** — another session cannot open them. The deliverable is a prompt × model table with statuses; mark dead chats as "ignore".

**Confirmed 2026-06-17:** download/copy in ChatGPT and Gemini require a **trusted user gesture** — synthetic clicks from Claude_in_Chrome do NOT trigger them (the Download menu in the sandbox iframe does not open; "Copy response" does not write to the clipboard without user activation; Gemini text is not in the DOM). I.e. automatic extraction of finished reports from those two is **impossible** — manual only. An alternative to try: **Codex browser plugin + the `$chrome` skill for Codex** (its own browser automation, downloads where Claude_in_Chrome cannot).

## What NOT to do
Do not paste sensitive data into external models. **Do not launch or even prepare Gemini Deep Research under a wrong or unidentified Google account — if no confirmed right account is available, Gemini is skipped.** Do not leave Claude connectors enabled. Do not type the prompt raw via `type`. Do not hit Retry on a failed Claude run (new chat only). Do not look for a "DeepSearch" button in Grok (there is none — the mode = the model dropdown) and do not downgrade to Expert on your own (default is Heavy; Expert only when the user explicitly asks to save quota). Do not confuse with a local deep-research harness or with `digger`.
