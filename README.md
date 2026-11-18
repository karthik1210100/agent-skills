<h1 align="center">🧩 agent-skills</h1>

<p align="center"><strong>Portable AI agent skills for full-stack developers — opinionated playbooks your coding agent can follow.</strong></p>

<p align="center">
  <img src="https://img.shields.io/badge/format-SKILL.md-4f46e5?style=for-the-badge" alt="SKILL.md format" />
  <img src="https://img.shields.io/badge/license-MIT-22c55e?style=for-the-badge" alt="MIT" />
</p>

---

## What is this?

A small, curated set of **[Agent Skills](https://skills.sh)** — plain-Markdown playbooks that teach an AI coding agent *how* to do a specific task well, consistently, every time. Each skill is a single `SKILL.md` with a `name`, a `description` that controls when it triggers, and a focused, opinionated body.

These are written in the portable `SKILL.md` format, so the same files work across **Claude Code, Codex, GitHub Copilot, OpenCode** and other agents via the [`skills` CLI](https://skills.sh).

## Skills

| Skill | What it does |
|---|---|
| **[rails-api-endpoint](skills/rails-api-endpoint/SKILL.md)** | Build a Rails JSON endpoint that is thin, authorized, consistently shaped, and tested — routing, strong params, serialization, Pundit authorization, a single error shape, N+1 avoidance, and a request spec. |
| **[react-data-hook](skills/react-data-hook/SKILL.md)** | Fetch server data in React + TypeScript the right way — a typed, cancellable custom hook with loading/error/data states, `AbortController` cleanup, and no updates after unmount. |

Together they cover both ends of a feature: a clean API on the server and a correct way to consume it on the client.

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

[MIT](./LICENSE) © Karthikeyan
