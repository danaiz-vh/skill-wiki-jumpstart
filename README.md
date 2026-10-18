# Skill Wiki Jumpstart

Paste one prompt into your coding agent. Get a company skill wiki: one agent and five skills, answering your team's most common questions, in your first session.

Works with Claude Code, Codex, Cursor, Copilot, Gemini CLI, or any coding agent that can read a file and write code.

## The problem

Your company's knowledge lives in people's heads, old docs and the code itself. Every AI session starts from zero, so the model either guesses or relearns your system from scratch. Engineers become the help desk for "which job writes this table?"

A **skill wiki** fixes that. It is one git repo where you write what you know once, as small notes. A build turns them into skills that every AI your people use can read: coding agents, desktop assistants, and the production agents you ship.

## What you get after session one

- **One working agent** that answers your team's 5 most-asked questions from the wiki, verified with a real test.
- **A hub note plus 3–5 domain notes**, written with you, sourced from your code and existing docs.
- **A small build** (Python stdlib + PyYAML) that checks every note and packs skills per agent.
- **CI on every PR**, so a broken note or a bad link never ships.

Later, when you need them: generated facts (tables, jobs, repo index) refreshed automatically through a reviewed PR, and a customer-facing agent built from the same repo with internal knowledge kept out.

## Quick start

1. Create an empty folder (or GitHub repo) and open it in your coding agent.
2. Paste the full contents of [`skill-wiki-jumpstart-prompt.md`](skill-wiki-jumpstart-prompt.md).
3. Answer about 12 questions in three short batches. Each has a default, so "yes" is a fine answer.
4. When the agent reports back, open a fresh session and ask your 5 questions.

Plan for one focused session, roughly an hour or two, depending on how much your agent has to read.

## How it works

```text
notes/       you write these: one subject each, tagged with who may read them
generated/   scripts write these from your live systems, read-only
   ↓ resolve   agents.yaml decides what each agent may read
   ↓ bundle    allowed notes are packed into skills
   ↓ adapt     skills are wrapped for the agent that reads them
```

A few ideas carry the design:

- **Write once, every AI reads it.** One pool of notes serves engineers, analysts and production agents.
- **Audience tags are the only gate.** An untagged note reaches no one, so the default is safe.
- **Facts update themselves.** Generators keep table and job lists current; a person approves every change.
- **Vendor at the edge.** Only the last build step knows which AI reads the skills. Switching tools means one new adapter, not a rewrite.

## Read the story behind it

[The Skill Wiki: How We Made Our Agents More Intelligent, Relevant, and Easy to Build](LINK-TO-ARTICLE): how VisionHeight runs internal knowledge and five production agents from one skill wiki, and how a half-day debugging job became thirty minutes.

## Contributing

Issues and pull requests are welcome. This project is provided as-is, and responses may take a while.

## Credits

Created by Dan Aizenberg.

Released under the [MIT License](LICENSE).
