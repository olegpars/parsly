# Parsly

Public Claude Code skill system by [@olegpars](https://github.com/olegpars). One plugin, seven skills - plus a separate plugin of lightweight subagents, [Dirty Little Helpers](#dirty-little-helpers).

| Skill | What it does |
|---|---|
| `dreamteam` | One-shot multi-agent expert council for hard decisions: role personas, a verifier, an adversarial critic, a gap-fill round, synthesis into a single document. |
| `digger` | Autonomous multi-hour deep-research pipeline: scout council -> wave loop -> consolidation -> static knowledge-base site, with A/B/C/D confidence tiers on every claim. |
| `meaning` | Work out WHAT to say before choosing the form: extract meaning from material, generate meaning directions, audit a text/concept against its meanings, compare concepts, translate a meaning core into a format. |
| `worldbuilder` | Serial fictional worlds for short-form video: a world bible built file-by-file with HITL approval, canon evolution with conflict/retcon resolution, episode drafting, and read-only continuity checks. |
| `ogre` | Two-headed orchestration: Opus runs the pipeline and writes self-contained specs into GitHub issue bodies, a second model of a different architecture answers every non-trivial fork blind, and hands execute. |
| `conveyor` | Issue-map auto-execution: the head session executes tickets itself or through subagents, a model of a different family accepts each result, grills run in a fresh session -- frontier, claims, GitHub bookkeeping, one explicit "yes" before any push. |
| `dr-fanout` | Fan out deep research to four browser models via Chrome MCP: one prompt (4 runs for triangulation) or N prompts x 4 (ChatGPT + Gemini + Claude + Grok), with account guards, safe long-prompt insertion, and file-based report capture. |

## Install

```text
/plugin marketplace add olegpars/parsly
/plugin install parsly@parsly
```

Formerly PARSENAL (and before that Parsival); the old GitHub URLs redirect here. If you installed `parsenal@parsenal`, switch over once:

```text
/plugin uninstall parsenal@parsenal
/plugin marketplace remove parsenal
/plugin marketplace add olegpars/parsly
/plugin install parsly@parsly
```

All seven skills become available and trigger on their own phrases. Installed this way, Claude Code namespaces slash commands by plugin: use `/parsly:meaning`, `/parsly:worldbuilder`, `/parsly:ogre`, `/parsly:conveyor` and `/parsly:dr-fanout` (not the bare `/meaning` / `/worldbuilder` / `/ogre` / `/conveyor` / `/dr-fanout` forms).

Manual install of a single skill (no marketplace): git clone https://github.com/olegpars/parsly and copy the needed folder from `skills/` into `~/.claude/skills/` (Windows: `$env:USERPROFILE\.claude\skills\`). Only in this manual single-skill install do the bare `/meaning` / `/worldbuilder` / `/ogre` / `/conveyor` / `/dr-fanout` triggers apply.

## dreamteam

`dreamteam` is a one-shot multi-agent expert council skill. It turns one concrete question into a council run: a council of role personas (6-8 by default, 3-12 via modifiers) research in parallel, a verifier checks load-bearing claims, an adversarial critic attacks weak points, a gap-fill round closes important holes, and a synthesizer writes one decision document.

Dreamteam is built for focused decisions and fast understanding, not for ordinary brainstorming or exhaustive multi-hour research.

### What It Does

- Creates a self-contained brief from the current conversation and selected artifacts.
- Assigns expert personas with sharp, checkable questions.
- Runs council members in parallel with artifact reading and web research.
- Adds verification, adversarial critique, and targeted gap-fill.
- Produces a single markdown document under a relative output path such as `./councils/<slug>/dreamteam-<date>.md`.

### Modes

- `decision`: for "what should we do", option selection, strategy, risk, cost, and tradeoff analysis.
- `understanding`: for a fast map of a topic, facts, schools of thought, disputes, and practical implications.

### Modifiers

- `quick` or `smoke`: smaller council for a cheap test.
- `bigger`: more roles.
- `with preview`: return the proposed role list before the full run.
- `with verification`: enable verifier in understanding mode.
- `OUTPUT_LANG='ru'`: final document and agent outputs in Russian. Default is English.

### Quickstart

```text
Convene a council: should we self-host the vector database or use a managed service?
```

### Requirements

- A host that supports Workflow scripts with `agent`, `parallel`, `phase`, `log`, and nested `workflow` for multidreamteam.
- Node.js for `engine/extract-doc.mjs`.
- Web tools available to workflow agents when current external facts matter.
- Write access to the chosen output directory.

### Files

- `skills/dreamteam/SKILL.md`: orchestration instructions.
- `skills/dreamteam/engine/dreamteam-template.js`: single council workflow template.
- `skills/dreamteam/engine/multidreamteam-template.js`: batch master workflow.
- `skills/dreamteam/engine/extract-doc.mjs`: fallback document extractor.
- `skills/dreamteam/references/unattended.md`: generic permission guidance for unattended runs.

## digger

`digger` is an autonomous multi-hour deep-research pipeline for Claude Code. It turns an explicit "dig deep into this topic" request into a reusable knowledge base: scout council -> wave loop -> consolidation -> static site.

Every claim is tagged with an A/B/C/D confidence tier, from primary-source evidence to weak folklore. Raw entries, consolidated digests, and the website remain separate so the result is both auditable and readable.

### Requirements

- Claude Code.
- A plan suitable for long multi-agent runs. Claude Max is recommended for full digs because waves can run for hours and touch many sources.
- A project directory where dig output can be written. By default, digger uses `./digs/<slug>/`.

### Quickstart

Ask for a focused dig:

```text
Dig deep into WebGPU debugging tools and build a knowledge base.
```

`digger` will:

1. Confirm the topic and mode.
2. Save output under `./digs/<slug>/` unless you choose another base.
3. Run scout to build a taxonomy and seed frontier.
4. Show the taxonomy before expensive waves begin.
5. Run research waves, process results on disk, and update the frontier.
6. Consolidate entries into subtopic and cross-cutting digests.
7. Build `dist/index.html`.

For long `full` runs, arm the supervised allowlist described in `skills/digger/references/unattended.md` before stepping away.

### Result Structure

```text
digs/<slug>/
|-- _meta/                 engine copies, manifest, taxonomy, frontier, state
|-- entries/               raw source notes, one file per researched source
|-- subtopics/             consolidated digest per subtopic
|-- cross-cutting/         consolidated cross-topic digests
|-- OVERVIEW.md            executive overview and comparison tables
|-- frontier.md            pending research queue
|-- sources.md             deduplicated source registry
|-- CHANGELOG.md           wave history
`-- dist/index.html        static knowledge-base site
```

### What Is Included

- `scout.js`: independent taxonomy proposals plus synthesis.
- `wave.js`: parallel research agents with source extraction and `new_frontier`.
- `process-wave.js`: deduplication, frontier update, source registry, and wave telemetry.
- `consolidation.js`: idempotent digest generation.
- `build-site.mjs`: single-file static site builder.
- `scripts/overnight.*` and `references/overnight-runner.md`: portable headless overnight runner for long `full` digs.

The public skill uses a supervised allowlist plus heartbeat for long runs. It does not include private runner infrastructure.

## meaning

`meaning` works out WHAT to say before choosing the form. A post, a script, an artwork concept, and an open-call application are treated as different renders of one underlying meaning construction, not separate tasks.

It is a conversational skill: no engine scripts, just SKILL.md plus reference files that Claude reads on demand.

### Modes

- `RESEARCH`: find a territory for an external brief (open call, venue, commission) before generating directions.
- `EXTRACT`: pull the meaning already present in a material without inventing anything.
- `GENERATE`: propose 3-5 meaning directions that differ by thesis, not phrasing.
- `AUDIT`: diagnose a finished text/concept against its meanings before proposing any edits.
- `SELECT`: compare 2+ directions with critique-before-scoring and an anti-homogenization check.
- `TRANSLATE`: render an approved Meaning Map into a target format via its adapter.

Every mode centers on a **Meaning Map** -- a structured card (subject, observation, tension, claim, stakes, audience_shift, embodiment, counter_reading, grounding, authorship) that must be shown and approved before any final text is written.

### Adapters

`skills/meaning/adapters/` covers paragraph/post, video script (reel/short), artwork concept/artist statement, and open-call application. Each adapter maps the generic Meaning Map fields onto format-specific checks.

### Quickstart

```text
/parsly:meaning -- audit this draft against its meaning map
```

### Requirements

- Claude Code only: a conversational skill -- no engine scripts, no extra runtime. SKILL.md plus reference files read on demand.

### Files

- `skills/meaning/SKILL.md`: modes, the Meaning Map, the constitution, and hard rules.
- `skills/meaning/failure-modes.md`: a 12-mode catalog of meaning failures (pseudo-depth, hermetic drift, slop, and others), each with a procedural test.
- `skills/meaning/research-protocol.md`: the RESEARCH-mode protocol for external briefs.
- `skills/meaning/adapters/`: format-specific renders.
- `skills/meaning/memory/`: empty templates for a user's own recurring questions, rejected cliches, and approved meaning cards -- the skill can grow project memory over time, but ships with no prefilled entries.

## worldbuilder

`worldbuilder` is a conversational engine for developing serial fictional worlds for short-form video content (Reels/Shorts/TikTok with recurring creatures, locations, objects), built around a world bible format. It is generic -- not tied to any single world.

A world is a folder `worlds/<slug>/` in your working repo, created by copying the bible template bundled inside this skill (`skills/worldbuilder/template/`, kept file-by-file: creative constitution, world overview, creature/location/object records, episode architecture, a seriality checklist, an append-only continuity log).

### Modes

- `CREATE`: build a new world's bible file-by-file, starting with a mandatory call to the `meaning` skill for the meaning core, each file shown and approved before it is written.
- `EVOLVE`: check a proposed canon change against the existing bible, propose a conflict-free resolution or an explicitly labeled retcon, then write it plus a new continuity-log entry after approval.
- `STORY`: draft an episode from the world's bible and continuity log, using generators such as Sanderson's Third Law and an engagement loop; canon consequences require a separate explicit EVOLVE.
- `CHECK`: read-only compatibility check of an idea/text against the bible's immutable rules, iceberg markers, and continuity state.

The HITL boundary is absolute: nothing is written to a world's canon (bible file, record card, continuity log) before the user has explicitly approved that specific content.

### Quickstart

```text
/parsly:worldbuilder -- create a new serial world for my short-form video series
```

### Requirements

- Claude Code only: a conversational engine, no scripts to run.
- Write access to your working repo: a world lives in `worlds/<slug>/`, copied from the bundled template.

### Files

- `skills/worldbuilder/SKILL.md`: modes, the HITL constitution, and hard rules.
- `skills/worldbuilder/template/`: the bundled world bible template CREATE copies from -- `README.md`, `00-creative-constitution.md`, `01-world-overview.md`, `02-records/` (creature/location/object cards), `03-episode-architecture.md`, `04-continuity-log.md`, `checklist-creature-series.md`.

## ogre

`ogre` is an orchestration mode: one head runs the pipeline, a second head of a **different architecture** answers every non-trivial fork blind, before it can be anchored by the first head's opinion. A single model is wrong exactly where it cannot see its own blind spots; two models of different architectures are wrong differently.

| Role | Who | What it does |
|---|---|---|
| Head 1 | Opus (the current session) | Runs the pipeline, writes self-contained specs into GitHub issue bodies, synthesizes, makes the final call. Has hands, inside explicit limits. |
| Head 2 | GPT-5.6-sol via Codex, read-only | Answers the fork **independently and blind** -- not a review of Opus's answer, its own answer. |
| Hands | Codex `--write`, headless Grok, Sonnet | Execution without judgement: coding and line-by-line review, scouting, `gh` operations, verification, acceptance. |

The blind protocol is the point: Opus states its own answer **first**, then sends the question with no hint of that answer, then synthesizes on the merits. A disagreement is always surfaced to the user as a line, never quietly resolved. Money, scope and irreversible actions form a **separate gate** that is not tied to whether the heads agree -- two models agreeing is an argument about correctness, not an authorization.

### What Is Actually Enforced

- **Issue-first.** One-line exception only, and only inside a file the user named -- the moment the diff grows or behaviour changes at a system boundary, an issue exists first.
- **Falsifiable DoD with a negative control.** A check that would also pass on the code *before* the change proves nothing and counts as "unverifiable".
- **A DoD item is a command and its expected output verbatim**, not a description in words; a place to edit is addressed by an anchor (heading, function name, quoted line), never by a line number -- line numbers drift after the first edit.
- **A delegation threshold by class of operation, not by file count.** Batch work, measuring across a set, an exhaustive search and producing the facts a decision rests on go to a hand immediately -- even when doing it yourself would be faster.
- **Opus never accepts its own work.** A cross-model verification matrix: whoever wrote it does not sign it off.
- **Observation is not a verdict.** "Works", "checked it", "nothing broke" are banned in reports until a verifier line of the form `<model> ran <command> -> <fact>` exists.
- **A grounding gate for synthesis tasks.** A guide/summary/digest is cross-checked verbatim against the deepest source, not against a derived corpus.
- **A red-flag table** -- 16 specific rationalizations that surface under deadline, sunk cost and a filling context window, each paired with what is actually happening.

### Quickstart

```text
/parsly:ogre -- two heads on this: migrate the billing cron to the new scheduler
```

### Requirements

- **Required for the second head and the coding hand:** the Codex plugin -- `/plugin marketplace add openai/codex-plugin-cc` -> `/plugin install codex@openai-codex` -> `/reload-plugins` -> `/codex:setup`. Without it the mode runs single-headed and marks every fork that went through without a second opinion.
- **Optional:** headless Grok (`grok.exe`, SuperGrok subscription) as a scout / second driver; an external terminal orchestrator for visible worker streams.
- A GitHub repo with `gh` available: issue bodies are where specs live, and the issue carries the whole acceptance machinery -- DoD, verifier, journal, closing comment.

### Files

- `skills/ogre/SKILL.md`: the mode -- limits on the head's hands, the second-opinion protocol, the delegation threshold, the starting gate, the pipeline, acceptance, the money/scope gate, the journal, the red-flag table.
- `skills/ogre/references/channels.md`: commands, flags and field-tested constraints for every channel (Codex as the second head and as a hand, Grok, the Sonnet hand). Read before the first call to a channel in a session.
- `skills/ogre/references/spec-and-dispatch.md`: the issue body template, executor and verifier envelopes, per-class DoD requirements, the failure ladder, the journal legend, agent survivability.
- `skills/ogre/references/gates.md`: what counts as money and as scope, what exactly stops, what to do when the user is unavailable, how to phrase the question.

### Lineage

Forked from a private predecessor mode built on the methodology of Sergey (serejaris)'s streams (agent map, cross-model verification, Grok as scout). Falsifiable DoD, the ban on "probably", stop-on-disagreement and the "noticed, didn't touch" rule are adapted from the Rigor Pack by Iwo Szapar. What ogre adds on top: the second head with its blind protocol, hands returned to Opus with explicit limits, and the separate money/scope gate.

## conveyor

`conveyor` drives an issue map to completion. A map-charting skill (for example the wayfinder skill in [mattpocock/skills](https://github.com/mattpocock/skills)) builds a GitHub issue with sub-issues and stops; conveyor is the continuation: one session takes that issue and drives it to its Destination. The head session executes tickets itself or through subagents, a model of a different family accepts every result, and the head keeps the GitHub bookkeeping.

### How It Runs

- **Start mark:** the map gets a `wip:conveyor` label and a `conveyor-start` comment, so the label alone tells whether a run is in progress.
- **Frontier:** open sub-issues that are not blocked, not claimed, and not marked human-only. Ticket types come from `wayfinder:*` labels; blocked-by comes from native GitHub issue dependencies, with a `Blocked by: #n` line as the fallback.
- **Execution:** one ticket, or tickets touching the same files, the head does itself; 2+ independent tickets go to parallel subagents (up to 4 at once). Subagents never close tickets and never commit -- the head commits per ticket and closes what was accepted.
- **Acceptance:** every ticket is accepted by a model of a different family (for example Codex CLI in a read-only sandbox), never by its executor. Findings are graded P1/P2/P3 against the ticket's DoD; at most two full acceptances per ticket: remaining P2 go to a follow-up ticket, P3 are noted, and a remaining P1 gets a fresh executor and one last acceptance of those P1 only; if they still fail, the ticket is marked `blocked` -- no endless loop.
- **Bookkeeping:** close accepted tickets, release claims, update blocked-by links, add a gist to the map's `Decisions so far` -- then re-check that everything landed in the target repo.
- **Restart over compaction:** run state lives in GitHub (claims, labels, journal), so a fresh session resumes idempotently instead of compacting a bloated context.

### HITL

Grills never run in the run session. When only grilling tickets are left, the run hands off to a fresh grill session (a `spawn_task` chip in Claude Code Desktop, or a one-line command elsewhere): it goes through the questions in rounds, each with a recommendation, records the decisions in the issues, and hands off back to the run. With the user away, the queue is written to the map while autonomous waves continue. Any push waits for one explicit "yes" at the end of the run; anything external, irreversible, or costing money is never executed in the background.

### Quickstart

```text
/parsly:conveyor -- drive the map #12 to the end
```

### Быстрый старт (RU)

1. Поставьте плагин: `/plugin marketplace add olegpars/parsly`, затем `/plugin install parsly@parsly`.
2. Нужна карта: GitHub issue с sub-issues, в теле - раздел `## Destination` (чем заканчивается работа). Удобно строить её скиллом wayfinder из [mattpocock/skills](https://github.com/mattpocock/skills); подойдёт и любой issue с sub-issues - conveyor сам спросит Destination.
3. В свежей сессии Claude Code: `/parsly:conveyor owner/repo#12` (или «drive the map owner/repo#12 to the end»).
4. Дальше конвейер сам: план волны, исполнение, приёмка другой моделью, закрытие тикетов. Вопросы к вам - отдельной сессией-грилем; push - только после вашего «да» в конце прогона.

### Requirements

- A GitHub repo with `gh` available: the map, claims, journal, and acceptance all live in issues.
- A host that can run subagents (Claude Code).
- A verifier of a different model family with a read-only mode (for example Codex CLI); without one, a fresh-context subagent accepts, and the comment says so.

### Files

- `skills/conveyor/SKILL.md`: the loop, acceptance, worktree rules, claims and restart, grills in a fresh session, gates, models, and v1 boundaries.

## Dirty Little Helpers

A separate plugin in the same marketplace: four lightweight subagents the main agent hands rough work to. Each one starts with a small tool set, so a call costs several times less than the built-in general-purpose subagent. Works in **Claude Code**, **Cursor** and **Codex**.

| Agent | What it does | Read-only |
|---|---|---|
| `scout` | finds and reads files, summarizes briefly, changes nothing | yes |
| `web-scout` | searches the web on one question, returns a brief with links | writes only to the file you name |
| `critic` | checks a finished text, plan or code, edits nothing | yes |
| `worker` | does one task with edits in the files you name | no |

### Why

When an agent hands a task to a subagent without saying which one, the general-purpose subagent starts. It brings everything: the skill list, all tools and connectors. In our measurement in Claude Code its first call cost about 60K tokens; a lightweight subagent with a narrow tool set cost 9-16K. Scouting and checking also run in a separate context window, so only a short summary returns to the main conversation. The four descriptions add about 195 tokens to every Claude Code session; the full agent text loads only when the agent is called.

The numbers are our own measurement of one first call on Opus in Claude Code, not Anthropic data. Cursor and Codex were not measured.

### Install

Claude Code, as a plugin (only the four agents, without the Parsly skills):

```text
/plugin marketplace add olegpars/parsly
/plugin install dirty-little-helpers@parsly
```

Plugin agents carry a prefix: `dirty-little-helpers:scout`, `dirty-little-helpers:critic` and so on.

Claude Code, by copying files: put the four `.md` files from `plugins/dirty-little-helpers/agents/` into `~/.claude/agents/` (all projects) or `.claude/agents/` inside a project, then restart Claude Code. The agents are called `scout`, `web-scout`, `critic`, `worker`.

Cursor reads the Claude Code agents folder, so the copied files work there too; otherwise put the same `.md` files into `~/.cursor/agents/`. Cursor ignores `tools` - the only restriction it honors is `readonly: true` (set on `scout` and `critic`). If an agent does not start, change `model: sonnet` to `model: inherit`.

Codex: put the `.toml` files from `plugins/dirty-little-helpers/codex/agents/` into `~/.codex/agents/` (or `.codex/agents/` in a project). In Codex the hyphenated agent is `web_scout`. Skills, plugins, apps and memories are switched off in these files; Codex cannot disable single tools, so read-only for `scout` and `critic` rests on their instructions. We tested Codex CLI 0.160: it finds all four files, but in `codex exec` it could not start an agent by name and passed the instructions to a helper by hand.

### Usage

Call them by name: "have scout find every .md file in this folder and say in a line what each is about", "have critic check this plan step by step". A subagent does not see your conversation - give it the whole task: what to do, where the files are, what to return.

### Быстрый старт (RU)

Скачивать и раскладывать файлы не нужно: агент поставит помощников сам. Скопируйте сообщение ниже (кнопка копирования - в правом верхнем углу блока) и отправьте его в Claude Code, Codex или Cursor как обычную просьбу.

```text
Поставь мне Dirty Little Helpers - четырёх помощников (scout, web-scout, critic, worker) из https://github.com/olegpars/parsly. Сделай всё сам, мне ничего не надо скачивать и искать.

1. Пойми, в какой ты программе (Claude Code, Codex или Cursor) и какая у меня система (Windows, Mac или Linux).
2. Скачай четыре файла, ветка main:
   - Claude Code и Cursor: scout.md, web-scout.md, critic.md, worker.md из plugins/dirty-little-helpers/agents/
   - Codex: scout.toml, web_scout.toml, critic.toml, worker.toml из plugins/dirty-little-helpers/codex/agents/
   Прямая ссылка: https://raw.githubusercontent.com/olegpars/parsly/main/plugins/dirty-little-helpers/agents/scout.md - остальные так же. Не открывается - возьми через git clone или страницу файла на github.com.
3. Положи их в папку агентов в моей домашней папке (нет папки - создай):
   - Claude Code: ~/.claude/agents (если задана CLAUDE_CONFIG_DIR - её папка agents)
   - Codex: ~/.codex/agents (если задана CODEX_HOME - её папка agents)
   - Cursor: ~/.cursor/agents, и в каждом файле замени строку model: sonnet на model: inherit
   Файлы - прямо в эту папку, без вложенных папок. Файл с таким именем уже есть - не трогай его и спроси меня.
4. Проверь, что все четыре файла на месте и не пустые.
5. Ответь коротко и без технических слов: что поставил, что программу надо перезапустить и как позвать помощника, например: «пусть critic проверит этот план».
```

Агент может спросить разрешение скачать и сохранить файлы - разрешите. Потом перезапустите программу и пишите: «пусть scout найдёт…», «пусть critic проверит этот план». Агенты отвечают на языке поручения.

Вручную в Claude Code - две команды в чате: `/plugin marketplace add olegpars/parsly`, затем `/plugin install dirty-little-helpers@parsly`.

Если что-то не получилось - бот [@olegpars42_bot](https://t.me/olegpars42_bot?start=agents_readme) подскажет.

### Files

- `plugins/dirty-little-helpers/agents/`: `scout.md`, `web-scout.md`, `critic.md`, `worker.md` - Claude Code and Cursor.
- `plugins/dirty-little-helpers/codex/agents/`: `scout.toml`, `web_scout.toml`, `critic.toml`, `worker.toml` - Codex.

## License

MIT
