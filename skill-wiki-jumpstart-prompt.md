# Skill Wiki — Jumpstart Prompt

> Paste this whole file into your coding agent (Claude Code, Codex, Cursor, Copilot, Gemini CLI,
> or similar), in an empty folder that is (or will be) a GitHub repo.
> Agent: this file is your brief. Read all of it before you do anything.

---

## 0. Your job

You are setting up a **skill wiki** for this company: one git repo that holds what the company
knows about its own systems, written as small notes, and **built** into skill packs for each AI
agent that needs them.

Work in this order:

1. **Ask the questions in §7** — in batches, with a default for each. Don't block on optional ones.
2. **Do the first steps in §8.** The goal for session one is small and real: one agent, five
   useful notes, installed and answering real questions. Not a framework.
3. **Stop and show the user** what works, what's next, and what you assumed.

Keep everything as simple as it can be. Use Python stdlib + PyYAML for the build. No new
dependencies, services or abstractions until a real need shows up.

---

## 1. Purpose

People (and agents) keep asking the same questions: *Where does this table come from? Which job
fills it? How do I query this correctly? What does this field mean? Why did last night's run
fail?* The answers live in people's heads, old docs, and the code itself.

The skill wiki fixes that by:

- **Writing the answers down once**, as notes, in git, reviewed like code.
- **Shipping them to AI agents as skills**, so the agents the company uses answer from the
  company's real facts instead of guessing.
- **Refreshing the facts that change** (table lists, job schedules, repo index) automatically from
  the live systems, read-only, through a PR a human approves.
- **Optionally serving customers** — the same repo can build a separate, safe skill set for a
  client-facing agent, without leaking internal details. Whether this is needed is a question
  (§7), not an assumption.

**The wiki is agent-neutral.** The notes and `agents.yaml` don't care which AI vendor reads them.
Only the last build step — the **adapter** — knows about a specific agent.

---

## 2. The model in one picture

```text
notes/            humans write these — small, one subject each, tagged with who may read them
generated/        scripts write these from the live systems — never edited by hand
   ↓  resolve     agents.yaml: for each agent, walk its entry notes, keep only what it may read
   ↓  bundle      pack the allowed notes into a few skills per agent (neutral format, see below)
   ↓  adapt       wrap the skills for the agent that will read them
build output      plugins/ (committed, git ships it)   dist/ (gitignored)
```

**The neutral format** is a folder per skill: `SKILL.md` (frontmatter with `name` and
`description`, then the body) plus a `references/` folder of files it can open on demand. This is
the open Agent Skills layout; several coding agents read it as-is.

**Adapters** (build only the ones the user needs):

| Adapter | When | Output |
|---|---|---|
| `skills-dir` | The agent reads `SKILL.md` folders natively | the skill folders, copied to wherever that agent looks for skills |
| `claude-plugin` | Claude Code / Claude apps, distributed through git | skill folders + `.claude-plugin/` manifest |
| `agents-md` | The agent has no skills support, only an instructions file (`AGENTS.md`, `.cursor/rules`, `copilot-instructions.md`, …) | a short index file listing each skill's description and path, plus the folders |
| `api-payload` | A hosted/API agent you deploy yourself (any vendor) | system prompt + tool definitions + skill files, ready for a deploy script |
| `mcp` | Many tools must share one live source | a tiny MCP server that serves the skill files — **only if a file-based adapter can't work** |

**You only ever edit `notes/`, `agents.yaml`, `prompts/` and the tools.** Everything the build
writes is rebuilt.

---

## 3. Core concepts

### Note vs skill
- A **note** is the source: one markdown file, one subject, with frontmatter.
- A **skill** is the build output an agent loads. Several notes usually pack into one skill.
- Why split them: the same note can be packed differently for different agents.

### Three questions, three different places
Each question has exactly one right place to answer it. Mixing them is where complexity comes from.

| Question | Answered by | Example |
|---|---|---|
| Is this **relevant** to the agent's job? | `entry` in agents.yaml | The support bot doesn't need the deploy runbook |
| Can the agent **physically do** this? | `tools` in agents.yaml | No SQL tool → it can't query, whatever it reads |
| Is the agent **allowed to know** this? | `tags` on the note | This note names internal hostnames |

A new tag is only worth adding if two agents should differ on a note for a reason that is
**neither relevance nor capability**. That test usually leaves you with one or two tags.

### Tags are audiences
- Typical start: `internal` (staff only). Add `client-safe` only if a customer-facing agent exists.
- **Audiences are independent, not a ladder.** A note both need carries **both** tags.
- An agent reads a note if they **share at least one tag** (note `tags` ∩ agent `grants`).

### ⚠️ Over-tagging is the leak
- Forgetting a tag is safe — the note reaches nobody.
- Adding `client-safe` is **an act of publication.** No script can tell whether the content
  deserves it. A human reads it back first, asking: *"If a customer pasted this into a
  competitor's chat, what did we just teach them?"*
- **Client-safe is a rewrite, not an excerpt.** Never trim an internal note into a public one —
  write a new note starting from the customer's question.

### Reach rule
For each agent, start at every note in its `entry`, then follow `references:`:
1. Keep a note if it shares a tag with the agent's `grants`.
2. Follow its `references:` and repeat.
3. **Never walk through a blocked note** — what's only reachable behind a forbidden note stays out.

### Promotion rule
- A note named **directly** in an agent's `entry` becomes **its own skill** (its `description` is
  what makes the agent pick it).
- Every other reachable note **nests as a reference file** inside its parent skill — it costs
  nothing until the agent opens it.

### Hub pattern (for the internal wiki)
- One **hub** note = one skill: a short map of the company's systems that routes to every domain
  note as a nested reference.
- Why: always-on cost is one description instead of twenty. In practice this cut always-loaded
  tokens by ~20× with no loss in routing.
- **Don't copy this to customer-facing agents.** There, each skill is a different job, and separate
  descriptions are what make the right one load.

### Authored vs generated
- **Authored** (`notes/`): the why, the how, the gotchas. Changes slowly.
- **Generated** (`generated/`): lists that drift — tables and columns, jobs and schedules, repos and
  their latest commit. Written by scripts in `tools/`, read-only against live systems.
- A note **claims** a generated file; it ships as a reference inside that note's skill.
- The note's `description` and `tags` are always hand-written. A script never decides who may read
  something.

---

## 4. Rules

1. **Never hand-edit build output** (`plugins/`, `dist/`, `generated/`).
2. **Every note has at least one tag.** No tag = a build error, not a silent default.
3. **`description` is the most important line in a note.** It's always loaded, it's the only thing
   the agent sees when deciding to open the skill, and it **ships** to every audience the note is
   tagged for. Write it as "Use this when…". Keep it under ~900 characters.
4. **Shipped text = `description` + body.** It may not name a note its readers can't see.
5. **No "accessible to" lines in a note body.** Ask the resolver instead; prose copies rot.
6. **Everything that touches live systems is read-only.** Use a dedicated read-only credential.
7. **Refresh by PR, never by direct push.** A human approves every change to the wiki.
8. **The repo wins over any console.** If an agent's prompt, tools or skills were edited in a UI,
   the next deploy overwrites it. Say so in the deploy script.
9. **If client-facing exists:** keep a term blocklist (internal hostnames, codenames, vendor names,
   customer names) checked in CI. It's a **second line** — it catches copy-paste, not meaning. A
   human review is the first line.
10. **Facts over warnings.** Notes state what is true and how to do it. Measure before claiming.
11. **Small notes.** One subject per file. If a note needs a table of contents, split it.
12. **`build/check.py` prints every problem, not the first,** and CI fails on any.
13. **No vendor names in `notes/`.** Notes describe the company's systems, not the AI reading them.
    Vendor specifics live only in adapters.

---

## 5. Repo structure

```text
<company>-skill-wiki/
├── AGENTS.md                 how to work in this repo (short; points here). Copy or link it
│                             to whatever file the team's agent reads (CLAUDE.md, GEMINI.md, …)
├── README.md                 what it is, how to install, how to add a note
├── agents.yaml               who gets what: tags, agents, grants, entry, tools, adapter, output
├── notes/
│   ├── hub/                  the one map note for the internal wiki
│   ├── general/              one subject per file — the durable explanations
│   └── specific/             detail loaded on demand
│       ├── scripts/          helper scripts a note ships with (optional)
│       └── assets/           templates, fonts, non-prose files (optional)
├── generated/                written by tools/ from live systems — never hand-edit
├── prompts/                  system prompts for API/client agents, versioned with their skills
├── agent-tools/              JSON tool definitions for API agents (optional)
├── tools/                    read-only generators: gen_schema_index.py, gen_job_catalog.py, …
├── build/
│   ├── build.py              resolve → bundle → adapt → check (split only when it hurts)
│   └── test_build.py         one small test that fails if reach or tagging breaks
├── plugins/<wiki-name>/      BUILT internal pack — committed, git is how staff get it
├── dist/                     BUILT payloads for API agents — gitignored
└── .github/workflows/
    ├── ci.yml                build + check on every PR
    └── refresh.yml           scheduled: run generators, open a PR if anything changed
```

Add vendor-specific files (e.g. a plugin manifest folder) only when that adapter is chosen.

### Note format

```markdown
---
name: orders-pipeline              # equals the filename, unique across the repo
description: >-
  Use when a question touches the orders pipeline — how raw orders land, which job builds
  analytics.orders_daily, its schedule, and how to backfill a day.
layer: general                     # hub | general | specific
tags: [internal]                   # at least one; both audiences need it? list both
references:                        # other notes' names — the edges the resolver walks
  - orders-backfill-runbook
generated:                         # optional — refreshed files this note owns
  - job-catalog.md
---

# Orders pipeline
> One line: what this is and when to read it.

…body: what it is, where it lives, how to use it, gotchas…
```

---

## 6. Example `agents.yaml`

```yaml
# Who gets what. Read by build/build.py. You edit this and notes/ — never the build output.
#
#   entry    relevance   notes relevant to this agent's job; a name here becomes its own skill
#   grants   permission  audiences it may read (a note is allowed if it shares ONE tag with this)
#   tools    capability  what it can DO — a document can't grant a tool, so this isn't a tag
#   adapter  packaging   how the skills are wrapped for the agent that reads them

# ── Audiences ──────────────────────────────────────────────────────────────────────────────
# Independent, no hierarchy. A note both need carries BOTH tags.
# ⚠️ Over-tagging is the leak: adding `client-safe` publishes the note.
tags:
  internal: >-
    Staff only. Schemas, pipelines, jobs and schedules, infrastructure, runbooks, credentials
    locations, customer names, anything about how the product is built.
  client-safe: >-
    Written for customer-facing agents. Product vocabulary only — what fields mean, how to use
    the public API, what a score or status means. Never internal hostnames, table paths, job
    names, vendors, or other customers. A note with ONLY this tag is withheld from staff agents.

agents:

  # Staff wiki for the team's coding agents, shipped through git.
  acme-internal-wiki:
    grants: [internal]
    adapter: skills-dir                  # or claude-plugin / agents-md — see Q2b
    output: plugins/acme-wiki            # the directory name is what people install — don't rename
    # Hub only: one skill, every domain note nests under it as a reference.
    entry:
      - acme-platform

  # Customer-facing support agent (only if the answer to Q2 is "yes"). Vendor-neutral payload;
  # a deploy script per runtime pushes it.
  acme-support:
    grants: [client-safe]
    adapter: api-payload
    prompt: prompts/acme-support.md
    output: dist/acme-support            # ⚠️ never under plugins/ — that ships to staff
    tools: [public_api]                  # from agent-tools/public_api.json
    # One skill per job: here a question may match only one, so separate descriptions matter.
    entry:
      - api-usage
      - field-glossary
      - status-meanings
    # One id per environment; a merge deploys staging, a release tag deploys prod.
    agent_id:
      staging: <filled in after the agent is created by hand>
      prod: <filled in after the agent is created by hand>

  # Batch agent with its whole method in the prompt. Empty grants/entry is deliberate:
  # it says "this agent reads no notes", so a client-safe note never lands in a batch job.
  acme-weekly-digest:
    grants: []
    entry: []
    adapter: api-payload
    prompt: prompts/acme-weekly-digest.md
```

---

## 7. Questions to ask the user

Ask in **three short batches**. Offer a default for each so the user can just say "yes".
Record the answers in `docs/setup-answers.md` — they are the project's first decisions.

### Batch A — must know before anything (blocks the first build)
1. **Company and wiki name?** (default: `<company>-wiki`)
2. **Who is this for?**
   - Internal staff only — *default*
   - Staff **and** a customer-facing agent
   - Customer-facing only

   → This decides the tags. Internal only = one tag, no blocklist, no `prompts/`.

   **2b. Which AI agents will read it?** List every one the team uses (coding agents, chat
   assistants, IDE assistants, hosted agents). *Default: the agent running this prompt.*
   → This decides the adapters. Build one adapter per distinct agent, nothing more.
3. **What are the 5 questions people ask most?** Real ones, in their words. These become the
   first notes and the acceptance test.
4. **Where does the code live?** GitHub org, monorepo vs many repos, and which 3–5 repos matter
   most to start.

   **4b. Do you need a full map of the code?**
   - **No — just the 3–5 key repos** — *default.* Hand-written notes for those repos only.
   - **Yes — every repo/service** — a generator (`tools/gen_repo_index.py`) lists every repo
     with its purpose, owner, main language, and latest commit into `generated/repo-index.md`.
     It runs read-only through the GitHub API and is refreshed like any other generated file.
     Ask which repos to leave out (archived, forks, experiments).

   → A full map isn't part of session one. If they say yes, it's the first generator after it.

### Batch B — sources of truth (decides the generators)
5. **Where does the data live?** (Postgres, Snowflake, BigQuery, Redshift, Databricks, S3/GCS
   files, spreadsheets, "nowhere structured") — and is there a **read-only** way to list tables
   and columns?
6. **Where do things run and on what schedule?** (Airflow, cron, GitHub Actions, cloud
   schedulers, Kubernetes jobs, a vendor tool) — is there an API to list jobs and their schedules?
7. **Which cloud, if any?** (AWS / GCP / Azure / on-prem / mixed) — only matters for where
   generators get credentials and what notes describe.
8. **What docs already exist?** (Confluence, Notion, Google Docs, READMEs, ADRs) — seed material,
   to be rewritten into notes, not copied whole.

### Batch C — distribution and ownership (can default and revisit)
9. **How should staff get the wiki?** *Default:* clone/submodule the repo, or point each agent's
   skills/instructions path at `plugins/<wiki-name>`. Use the agent's org-wide plugin or rules
   distribution instead if it has one.
10. **Who owns the wiki and approves PRs?** One named person minimum.
11. **CI**: GitHub Actions OK? *(default yes)* Refresh cadence? *(default: 3×/week, opens a PR)*
12. **If customer-facing:** where does that agent run (a vendor's hosted-agent API, your own app,
    Slack, Teams, a website widget)? What tools may it call? And list **words that must never
    reach a customer** — internal codenames, hostnames, vendors, customer names. That list seeds
    the blocklist.

If an answer is "don't know", take the default, write it down as an assumption, and move on.

---

## 8. First steps (session one)

Aim: by the end, the user's agent answers their 5 questions from the wiki.

1. **Scaffold** the tree in §5. Only the folders you'll use today — skip `prompts/`,
   `agent-tools/`, `dist/` and the blocklist if the wiki is internal only.
2. **Write `agents.yaml`** with one agent (`<company>-internal-wiki`, `grants: [internal]`,
   entry = the hub, `adapter` = whatever Q2b chose).
3. **Write the hub note** — a one-screen map: the systems, where code / data / jobs live, and a
   one-line pointer to each domain note.
4. **Write 3–5 domain notes** that answer the 5 questions. Use the existing docs and the code as
   sources; read the code before stating a fact about it. Each note gets a sharp `description`.
5. **Write `build/build.py`**: load notes + agents.yaml → resolve reach per agent → bundle into the
   neutral format (hub = `SKILL.md`, reachable notes = `references/<name>.md`, body links
   rewritten) → run the agent's adapter → check.
   Checks: every note has a tag, names unique and equal to filenames, every `references:` and
   `entry` name resolves, description length, and (if client-facing) the blocklist on
   `client-safe` notes. Print every problem; exit non-zero on any.
6. **Write one test** (`build/test_build.py`) proving the reach rule: a note behind a blocked note
   is not reached, and an untagged note fails the check.
7. **Build, then install locally** the way the chosen agent loads skills or instructions (its
   skills folder, its plugin command, or its instructions file pointing at the output). Look up
   that agent's current docs rather than guessing the path.
8. **Acceptance test:** in a fresh agent session, ask the 5 questions. Each answer must come from
   the wiki (the skill or file is actually opened) and be correct. Fix the `description` first if
   a skill doesn't load.
9. **Add `ci.yml`** (build + check + test on every PR) and open the first PR.
10. **Report back:** what works, every assumption made, and the proposed next step.

### After session one (in this order, only when needed)
- **First generator** — the repo index if Q4b said "full map", otherwise whichever source drifts
  fastest (usually the table/column list). It writes `generated/<thing>.md`; a note claims it.
- **`refresh.yml`**: scheduled, read-only credential from repo secrets, runs generators, opens a PR
  with a short change summary. ⚠️ Diff the **`generated/`** folder to decide if anything changed —
  diffing the wrong folder makes the job go quiet forever while looking healthy.
- **A second adapter** when a second agent shows up — same notes, new wrapper.
- **Org distribution** through whatever the agent offers for org-wide plugins or rules, so staff
  get it with no setup.
- **Customer-facing agent** (if Q2 said so): new `client-safe` notes written from the customer's
  question down, a system prompt in `prompts/`, tool definitions in `agent-tools/`, a deploy
  script for that runtime that pushes prompt + tools + skills per environment, and the blocklist
  in CI.

---

## 9. Lessons already paid for

- **Description first.** When an agent doesn't pick up a skill, the description is the bug 9 times
  out of 10.
- **Fewer always-on skills beat more.** Nest by default; promote a note to its own skill only when
  it's a genuinely separate job.
- **A "narrowing" tag must stand alone.** If you add a tag like `client-sql-access` to gate SQL
  recipes to one client agent, put it on the note **instead of** `client-safe`, not beside it —
  under share-one-tag, a second tag widens the audience, it doesn't narrow it.
- **Naming a forbidden thing to forbid it also discloses it.** A client-safe note that says
  "never mention <codename>" just mentioned it.
- **Deploy scripts update agents; they don't create them.** Create each hosted agent once by hand
  per environment, record its id in `agents.yaml`, and let CI reconcile prompt, tools and skills
  after.
- **Keep generated lists out of notes.** A hand-copied table list is wrong within a month.
- **Keep the vendor at the edge.** Teams switch AI tools. If only the adapter knows the vendor,
  switching is one new adapter, not a rewrite of the wiki.
