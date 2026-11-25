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
  <a href="#quick-start">Quick start</a> &middot;
  <a href="#how-it-works">How it works</a> &middot;
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
| **[rails-sidekiq-job](skills/rails-sidekiq-job/SKILL.md)** | Write a Sidekiq background job that is safe to run more than once — enqueue ids not objects, a thin worker, an idempotency guard, a deliberate retry/backoff policy, locking for money and external calls, and a spec that proves a second run is a no-op. |

Between them they cover a whole feature: a clean API on the server, a correct way to consume it on the client, and safe background work behind it.

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

## Quick start

**1. Install a skill** into your project with the [`skills`](https://skills.sh) CLI — no global install, no config:

```bash
# pick the skill(s) you want
npx skills add https://github.com/karthik1210100/agent-skills --skill rails-api-endpoint
npx skills add https://github.com/karthik1210100/agent-skills --skill react-data-hook
npx skills add https://github.com/karthik1210100/agent-skills --skill rails-sidekiq-job
```

The CLI detects every agent you have (Claude Code, Codex, Copilot, …) and installs the skill for all of them.

**2. Then just work normally.** Ask your agent for the thing — *"add a `POST /articles` endpoint"* — and it matches your request against each skill's `description`, loads the right playbook, and follows it. You don't invoke it by hand; the `description` is the trigger.

> **Tip:** install a skill into any repo where you want that standard enforced. The files are committed with the project, so your teammates' agents follow the same playbook too.

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
