---
layout: single
title: "Second Brain, part 2"
date: 2026-09-30
categories: [ai]
tags: [second-brain, obsidian, claude, vscode, llm, productivity, people-management]
description: "Upgrading from Antigravity to Claude Code, switching models, cleaning up the instruction architecture, and wiring Obsidian's Dataview plugin into the corpus."
---

[Part 1]({% post_url 2026-09-28-second-brain-1 %}) was my first iteration of an [AI Second Brain](https://fdicostanzo.com/posts/2026/20260427_karpathy_llm_second). Two days in and I have some changes to document. 

<figure>
    <a href="/assets/second-brain/ai-brain.jpg"><img src="/assets/second-brain/ai-brain.jpg"></a>
    <figcaption>AI Second Brain</figcaption>
</figure>

The short version: I swapped Antigravity for Claude Code via VS Code, switched LLM from Gemini to Claude, cleaned up the instruction architecture, and added a few plugins to Obsidian.

---

### Why I Swapped Tools

Part 1 used **Antigravity** from Google. This is a decent agent with a good UI, but two annoying things got me to migrate to Claude Code:
- it was constantly prompting me for permission to do anything and I couldn't just blanket allow operations on local files
- recently I've been getting "model unavailable" errors with retries in a fall-back fashion. Super annoying and basically unuseable. 

**Claude Code** — Anthropic's agentic CLI — is built for exactly this. It runs an autonomous loop, issues its own file reads and writes, and doesn't drift mid-task. The VS Code extension puts a Claude Code terminal in the same window as your vault. For a system that manages files as its primary output, this is the right tool. Also, many colleagues use it so I have people to go to.

---

### Model switch: Gemini → Claude Sonnet

Antigravity used Gemini by default. Claude Code lets you pin a specific model in `settings.json`:

```json
	"claudeCode.environmentVariables": [
    {
      "name": "ANTHROPIC_AUTH_TOKEN",
      "value": "xxxxx"
    },
    {
      "name": "ANTHROPIC_BASE_URL",
      "value": "https://<intranet_url>/anthropic"
    },
    {
            "name": "ANTHROPIC_MODEL",
            "value": "claude-sonnet-4-6"
    }
]
```

I'm using Claude Sonnet 4.6. For a second brain, the job is following complex multi-step instructions reliably — not reasoning over hard math problems or generating creative writing. Sonnet hits the right balance of speed, cost, and instruction-following for this use case. Opus is more capable but noticeably slower for what is essentially document management.

---

### Updating the scaffolding

Part 1's setup wasn't terrible but there was some overlap:

- **`CLAUDE.md`** — Claude Code's instruction file
- **`AGENTS.md`** — the universal agent rulebook for any AI tool
- **`.agent/rules/leadership-rules.md`** — a condensed copy for tools that load rules from that path

**The fix:** `CLAUDE.md` is now 25 lines. It delegates everything to `AGENTS.md` and only adds Claude Code-specific configuration — which skills directory to use, where memory lives, file naming conventions. `AGENTS.md` is the single source of truth. `.agent/rules/leadership-rules.md` now just says "read AGENTS.md."

`AGENTS.md` also gained five new sections: a daily notes standard, inbox routing rules, staleness thresholds (14-day follow-up flag, 21-day 1:1 flag, 60-day radar archive recommendation), a `LOG.md` spec, and definitions for the new slash commands.

---

### Five New Slash Commands

| Command | What It Does |
|---|---|
| `/morning` | Reads open follow-ups, flags overdue items, suggests top 3 priorities for the day |
| `/capture <note>` | Drops a timestamped note into `inbox/` without context-switching |
| `/process-inbox` | Routes every file in `inbox/` to its correct vault location |
| `/weekly-review` | Digest from the week's daily notes plus staleness check |
| `/lint` | Health check: broken links, stale follow-ups, orphaned pages |

`/morning` is the one I expect to use daily. It's read-only, takes about ten seconds, and gives me a clear starting point without having to open the master index manually.

`/lint` is the one I'm most curious about long-term. The staleness problem — notes that quietly become outdated — killed every previous note system I tried. Having an automated health check changes the maintenance model from "trust yourself to remember" to "trust the agent to flag it."

---

### People Profile Frontmatter

Every person ledger in `people/` now has YAML frontmatter:

```yaml
---
name: [Name]
role: Senior Solutions Architect
status: active
last_updated: "2026-09-30"
last_1on1: "—"
---
```

This is the piece that enables everything else. Dataview queries read from frontmatter. `/lint` compares `last_1on1` against today's date. The `/morning` brief surfaces anyone with an overdue cadence. Without frontmatter, all of this requires the agent to parse prose — fragile and slow. With it, the data is structured and queryable.

The agent is also required to update these fields on every write. The dates stay accurate automatically.

---

### Obsidian extensions: Dataview + Omnisearch

Two community plugins:

**Dataview** turns Obsidian into a live database over your vault. The master index used to be a manually-maintained Markdown table. It now renders from a live query:

~~~dataview
TABLE WITHOUT ID
  link(file.path, name) AS "Name",
  role AS "Role",
  last_1on1 AS "Last 1:1",
  last_updated AS "Last Updated"
FROM "people"
WHERE status = "active" AND role != "Manager (skip-level)"
SORT last_1on1 ASC
~~~

The table sorts ascending by last 1:1 date, so the people I most need to check in with are always at the top. When the agent updates frontmatter after a debrief, the table updates automatically. No manual editing.

I also created `dashboard.md` — a new file with six live views: 1:1s never formally logged, the full team sorted by last 1:1, active initiatives, radar items, recent daily notes, and inbox status. It's the first file I open each day.

**Omnisearch** replaces Obsidian's built-in search with a fuzzy full-text engine that actually works. Searching for a customer or topic across dozens of daily notes returns results instantly. Install it, forget Obsidian has a native search.

---

### LOG.md: The Audit Trail

Every file mutation the agent makes now gets one appended line in `LOG.md`:

```
- 2026-09-30 `/1on1-debrief Dave`: updated `people/dave-remington.md` (1:1 history, follow-ups, frontmatter), `INDEX.md`
```

This matters for two reasons. First, debugging: when the agent does something unexpected, there's a record of what it changed and when. Second, trust: I was more comfortable giving the agent write access to my vault once I knew I could audit its actions without reading every file.

`LOG.md` is append-only by instruction. Neither I nor the agent edits prior entries.

---

### What's Next - maybe

- **Templater + Periodic Notes:** daily notes are still created manually. Templater auto-creates `daily/YYYY-MM-DD.md` on schedule with the right structure. This week's task.
- **Claude Desktop + MCP:** the Obsidian MCP plugin exposes your vault to Claude Desktop, so you can query it from the Claude.ai interface without opening VS Code. Useful for mobile and quick lookups.
- **Terminal plugin:** runs a shell inside Obsidian's sidebar. Means I can type `/morning` without switching apps.
- **Canvas team map:** a visual node graph of the team, color-coded by specialty and development path. Good for skip-level prep and org conversations with leadership.

The core principle from Part 1 holds: the agent is the programmer, the vault is the codebase. What changed is the quality of the tooling. Better agentic loop, cleaner instructions, structured data. The system is more reliable because it makes fewer demands on me to keep it consistent.

---

**Summary:** Switched from Antigravity to **Claude Code** for its agentic file loop. Pinned **Claude Sonnet 4.6** in `settings.json`. Reduced `CLAUDE.md` to a thin delegation layer, made `AGENTS.md` the single source of truth, and added five new slash commands. Added YAML frontmatter to every people profile. Installed **Dataview** and **Omnisearch** in Obsidian, replaced the static master index table with a live query, and created `dashboard.md` as the default home view. Added **LOG.md** as an append-only audit trail for all agent mutations.