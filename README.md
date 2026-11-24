<h1 align="center">🧩 agent-skills</h1>

<p align="center"><strong>Portable AI agent skills for full-stack developers — opinionated playbooks your coding agent can follow.</strong></p>

<p align="center">
  <a href="https://github.com/karthik1210100/agent-skills/stargazers"><img src="https://img.shields.io/github/stars/karthik1210100/agent-skills?style=for-the-badge&logo=github&label=%E2%AD%90%20Star%20this%20repo&color=yellow" alt="Star this repo" /></a>
  &nbsp;
  <img src="https://img.shields.io/badge/format-SKILL.md-4f46e5?style=for-the-badge" alt="SKILL.md format" />
  &nbsp;
  <img src="https://img.shields.io/badge/license-MIT-22c55e?style=for-the-badge" alt="MIT" />
</p>

<p align="center">
  <a href="#what-is-this">What it is</a> &middot;
  <a href="#skills">Skills</a> &middot;
  <a href="#how-it-works">How it works</a> &middot;
  <a href="#install">Install</a> &middot;
  <a href="#writing-your-own">Write your own</a>
</p>

---

## What is this?

A small, curated set of **[Agent Skills](https://skills.sh)** — plain-Markdown playbooks that teach an AI coding agent *how* to do a specific task well, consistently, every time. Each skill is a single `SKILL.md` with a `name`, a `description` that controls when it triggers, and a focused, opinionated body.

Your agent stops improvising the same decisions on every feature. It follows a playbook you reviewed once — so the Rails endpoint it writes on Friday looks like the one it wrote on Monday.

These are written in the portable `SKILL.md` format, so the same files work across **Claude Code, Codex, GitHub Copilot, OpenCode** and other agents via the [`skills` CLI](https://skills.sh).

## Skills

| Skill | What it does |
|---|---|
| **[rails-api-endpoint](skills/rails-api-endpoint/SKILL.md)** | Build a Rails JSON endpoint that is thin, authorized, consistently shaped, and tested — routing, strong params, serialization, Pundit authorization, a single error shape, N+1 avoidance, and a request spec. |
| **[react-data-hook](skills/react-data-hook/SKILL.md)** | Fetch server data in React + TypeScript the right way — a typed, cancellable custom hook with loading/error/data states, `AbortController` cleanup, and no updates after unmount. |

Together they cover both ends of a feature: a clean API on the server and a correct way to consume it on the client.

## How it works

```
  npx skills add …              install the SKILL.md into your repo
        │
        ▼
  your agent reads each skill's `description`  (when should I use this?)
        │
        ▼
  you ask:  "add a POST /articles endpoint"
        │
        ▼  description matches → loads rails-api-endpoint
  agent follows the playbook — same way, every time
        ├─ RESTful route · thin controller · strong params
        ├─ Pundit authorization · one consistent error shape
        └─ N+1 check · a request spec that actually runs
```

No prompt-wrangling at the moment you need the feature. The thinking happened once, when you wrote the skill.

## Install

Install any skill into your project with the [`skills`](https://skills.sh) CLI (no global install needed):

```bash
# a single skill
npx skills add https://github.com/karthik1210100/agent-skills --skill rails-api-endpoint

# the frontend one too
npx skills add https://github.com/karthik1210100/agent-skills --skill react-data-hook
```

The CLI installs the skill for every agent it detects (Claude Code, Codex, Copilot, …). Your agent then loads the playbook automatically when the task matches the skill's `description`.

## Use

Once installed, just work normally. When you ask your agent to, say, *"add a `POST /articles` endpoint"*, it picks up `rails-api-endpoint` and follows the playbook — RESTful route, thin controller, authorization, the standard error shape, and a request spec — instead of improvising.

## Writing your own

Each skill is one folder:

```
skills/
└── <skill-name>/
    └── SKILL.md
```

```markdown
---
name: my-skill
description: One or two sentences on WHEN to use this, including trigger phrases.
metadata:
  version: 1.0.0
---

# My Skill

Focused, opinionated instructions. Short, high-signal, with a checklist.
```

The `description` is the most important line — it's how the agent decides to load the skill, so make it concrete and list the phrases that should trigger it.

## License

[MIT](./LICENSE) © Karthikeyan Subramanian

---

<div align="center">

Built by <a href="https://github.com/karthik1210100">Karthikeyan Subramanian</a> &middot; <a href="https://www.linkedin.com/in/karthik-webdev/">LinkedIn</a> &middot; <a href="https://karthik1210100.github.io/portfolio_2026/">Portfolio</a>

<br />

<strong>If these playbooks save you a round of review, star the repo ⭐</strong>

<br /><br />

<a href="https://github.com/karthik1210100/agent-skills/stargazers"><img src="https://img.shields.io/github/stars/karthik1210100/agent-skills?style=for-the-badge&logo=github&label=%E2%AD%90%20Star&color=yellow" alt="Star this repo" /></a>

</div>
