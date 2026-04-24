---
name: morning-briefing-email
description: Generate the user's morning briefing and email it to them on a schedule (e.g. 7 AM daily).
---

Generate the morning briefing and email it to the user.

**Step 1: Load the main skill.**

Read `skills/briefing/SKILL.md`. That file is the source of truth for output shape, voice (warm + crisp), section order, data sources, guardrails, and word budget. Follow it exactly. Also read any context files it references (reminders, schedule preferences) if they exist.

**Step 2: Gather the data** (in parallel where possible):

1. **Weather** via WebSearch or WebFetch — current temp, conditions, high/low, rain chance. Translate to a real dressing cue.
2. **Today's Google Calendar events across all calendars** — call `list_calendars` first to enumerate them, then `list_events` per calendar using params `startTime` and `endTime` (NOT `timeMin`/`timeMax`) in ISO 8601 with TZ offset (e.g. `2026-04-23T00:00:00-04:00`). Subcalendars returning no events today are genuinely empty — don't mention them.
3. **Evening workout** — check today's calendar events first for titles/descriptions/locations mentioning configured studio names or class types (pilates, yoga, flow, etc.). If nothing on calendar, fall back to a Gmail search for those senders in the last ~24h. If neither finds anything, write "none scheduled."
4. **Urgent unreplied Gmail from last 12h** — judgment-based. Would a thoughtful friend tell the user about this? Skip automated senders (Vercel, GitHub CI, marketing, newsletters, receipts, social notifications). LinkedIn *message* notifications count; LinkedIn activity digests don't. 1–3 items max, sender + one-line gist. If nothing, write "Nothing urgent."
5. **Arriving today** — search Gmail for delivery notifications indicating a package arrives today. Look for "out for delivery," "arriving today," "delivering today" from carriers (UPS, FedEx, USPS, Amazon, etc.). 1–3 items as carrier/sender + item description. If nothing, write "Nothing expected."
6. **Career** — search Gmail from the last ~7 days for job outreach: recruiter emails, LinkedIn InMail notifications about a specific role, headhunter messages. Surface any genuine human reaching out about an opportunity. Skip automated job-board digests and "we're hiring" blasts. 1–3 items as sender/company + role or gist. If nothing, write "Nothing new."
7. **Heads up (time-sensitive older threads, last ~7 days)** — expiring quotes, pending RSVPs, deadlines approaching this week. Older than 12h but still actionable. 1–2 items max. If nothing qualifies, omit the entire "Heads up" section.
8. **First thing to tackle** — pick in this priority order: (a) most urgent item from the reminders file if configured, (b) prep for earliest meeting today that needs it, (c) gut-call judgment.
9. **Body (one health-supportive cue)** — one specific line tied to today's weather/workout/calendar. Not generic wellness advice. See SKILL.md for angles.

**Step 3: Compose the briefing.**

Follow the exact output shape in the main SKILL.md. Structure:

```
Good morning —

[Weather + dressing cue]

**Today**
- [Calendar events]

**Arriving today**
[Carrier + item, or "Nothing expected."]

**Evening workout**
[Class + time, or "none scheduled"]

**Body**
[One health cue]

**Needs a reply**
- [urgent items or "Nothing urgent."]

**Career**
- [job outreach items or "Nothing new."]

**Heads up** (only if non-empty)
- [time-sensitive older threads]

**First thing to tackle**
[One item]

Until tomorrow.
```

Under 400 words total. Warm, crisp voice. No filler, no hedging, no meta commentary about sources.

**Step 4: Email it.**

Write the briefing to a temp file, then pipe it through an Apple Shortcut you've configured to send email via Mail.app:

```bash
BRIEFING_FILE=$(mktemp)
cat > "$BRIEFING_FILE" <<'EOF'
<the briefing markdown>
EOF
cat "$BRIEFING_FILE" | shortcuts run "Email Me Briefing"
rm "$BRIEFING_FILE"
```

The shortcut receives the markdown via stdin and emails it with a subject like `Good morning — <date>`. See the README for shortcut setup.

**Step 5: Verify.**

Confirm the `shortcuts run` command exited 0. If it errored, report the error but do NOT retry automatically — a second send would result in a duplicate email, which is worse than a missed one. The scheduled task runs again tomorrow.

**Guardrails (from the main SKILL.md, repeated for safety):**
- If a data source errors or isn't connected, keep the section but replace its content with a one-liner like `Weather: unavailable`. Never silently skip. Never fabricate.
- Never include contents from channels, threads, or docs marked private or sensitive.
- Stay under 400 words.
- Don't describe the process, tools, or the fact that this is automated. The briefing is the briefing.

**Success criteria:** an email with subject `Good morning — <today>` lands in the user's inbox, body matches the SKILL.md structure, under 400 words, the user knows how to dress, knows their evening workout, and doesn't have to open Gmail/LinkedIn/Slack to feel caught up.
