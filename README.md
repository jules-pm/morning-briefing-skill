# Morning briefing — a Claude Code skill

A pair of [Claude Code](https://claude.com/claude-code) skills that produce a calm, structured morning briefing from your calendar, inbox, weather, and to-do list — delivered either in-chat or auto-emailed every morning.

The output is designed to replace the first 10 minutes of your day: no opening Gmail, no scanning LinkedIn, no Slack — just one screen that tells you what's on, what needs a reply, and what to tackle first.

## What it does

Triggered by a casual phrase — "good morning", "brief me", "gm" — or fired on a schedule, the skill gathers data in parallel and returns something like:

```
Good morning —

62°F with light rain moving in this afternoon — grab a light jacket and an umbrella.

**Today**
- 9:30 — team standup
- 11:00 — 1:1 with manager (bring last week's notes)
- 2:00 — interview prep block
- 4:00 — Acme partner call (needs agenda)

**Arriving today**
- UPS — replacement keyboard

**Evening workout**
7:00 — Pilates Flow at the usual studio

**Body**
Before the 4pm call, step outside for a five-minute walk to reset.

**Needs a reply**
- Sarah (Acme) — asking for revised pricing by EOD
- Mom — dinner Sunday?

**Career**
- Recruiter at Horizon — Senior PM, agent platforms team

**Heads up**
- Flight credit from Delta expires Friday

**First thing to tackle**
Draft the Acme pricing reply — it's blocking the 4pm call.

Until tomorrow.
```

## Why this is interesting

Most "morning dashboard" tools dump everything at you. The hard part isn't aggregation — it's **editorial judgment**: deciding which of 40 emails actually needs a reply vs. looks urgent but isn't, deciding which calendar events need prep, deciding what the *first* thing to tackle should be.

This skill encodes that judgment in plain English prompts. It's a showcase of how far you can get with skill authoring patterns — voice guardrails, word budgets, priority logic, data-source orchestration — before reaching for code.

## How it's structured

```
skills/
  briefing/
    SKILL.md              ← the core skill: output shape, voice, data sources, guardrails
scheduled-tasks/
  morning-briefing-email/
    SKILL.md              ← scheduled variant that emails the briefing every morning
```

The scheduled variant delegates to the core skill rather than duplicating it, so voice and structure stay in one place.

## What it uses

- **Claude Code** — runs the skill
- **Google Calendar MCP** — pulls events across primary and subcalendars
- **Gmail MCP** — searches for urgent replies, deliveries, career outreach, and time-sensitive threads
- **WebSearch / WebFetch** — current weather for the user's city
- **Apple Shortcuts** (optional) — for the scheduled email variant, a Shortcut that reads stdin and sends via Mail.app
- **Claude Code scheduled tasks** — for daily auto-run

## Setup

1. **Install Claude Code** and connect the Google Calendar + Gmail MCP servers.
2. **Drop the core skill** into your skills directory:
   ```
   ~/.claude/skills/briefing/SKILL.md
   ```
   Or keep it wherever you store personal context and point to it from a slash command.
3. **Personalize.** Edit the SKILL.md to set:
   - Your city and preferred temperature units
   - Known studio/class names for the workout-detection heuristic
   - Paths to any reminders or schedule-preference files you maintain
4. **(Optional) Set up the email variant:**
   - Create an Apple Shortcut named `Email Me Briefing` that takes stdin as input and emails it to yourself via Mail.app with subject `Good morning — <date>`.
   - Drop `scheduled-tasks/morning-briefing-email/SKILL.md` into `~/.claude/scheduled-tasks/morning-briefing-email/`.
   - Register it as a scheduled task in Claude Code (daily at 7 AM, or your preferred time).

## Design choices worth calling out

- **Voice guardrails in the skill itself.** "Warm, crisp. Short sentences. No filler, no hedging, no 'I hope this helps.'" — encoded once, applied every run.
- **Word budget.** Under 400 words total. The skill tells the model: if sections bloat, trim — fewer words per item, not fewer sections.
- **Empty-section logic.** "Heads up" disappears entirely when empty, rather than saying "Nothing pending." Other sections keep a one-liner ("Nothing urgent.") so the structure is predictable. The difference is intentional.
- **No retries on email failure.** A duplicate morning email is worse than a missed one. The scheduled task runs again tomorrow.
- **Judgment, not keyword matching.** Instead of a blocklist of automated senders, the skill frames the decision as: *would a thoughtful friend tell me about this?*

## License

MIT.
