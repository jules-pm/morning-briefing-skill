---
name: morning-briefing-email
description: Generate Jules's morning briefing and email it to her at 7 AM daily.
---

Generate Jules's morning briefing and email it to her.

**Step 1: Load the skill.**

Read `~/Desktop/context/Admin/skills/briefing/SKILL.md`. That file is the source of truth for output shape, voice (warm + crisp), section order, data sources, guardrails, and word budget (under 300 words). Follow it exactly. Also read these files it references:

- `~/Desktop/context/Admin/reminders.md` — her to-do list (used for "first thing to tackle" priority)
- `~/Desktop/context/Admin/schedule-preferences.md` — how she likes her day structured

**Step 2: Gather the data** (in parallel where possible):

1. **Brooklyn, NY weather in °F** — fetch directly from `https://wttr.in/Brooklyn,NY?format=j1` via WebFetch. Parse `current_condition[0].temp_F` for current temp, `current_condition[0].weatherDesc[0].value` for conditions, `weather[0].maxtempF`/`mintempF` for high/low, and `weather[0].hourly[].chanceofrain` for rain chance. Do not use WebSearch for weather. Translate to a real dressing cue.
2. **Today's Google Calendar events across all calendars** — call `list_calendars` first to enumerate them, then `list_events` per calendar using params `startTime` and `endTime` (NOT `timeMin`/`timeMax`) in ISO 8601 with TZ offset (e.g. `2026-04-23T00:00:00-04:00`). Include the "Jules & Andrew!" subcalendar. Subcalendars returning no events today are genuinely empty — don't mention them.
3. **Evening workout** — check today's calendar events first for titles/descriptions/locations mentioning `paloga`, `flow`, `east river pilates`, `pilates`, `yoga`, or `1161 Bedford`. If nothing on calendar, fall back to a Gmail search for those senders in the last ~24h. If neither finds anything, write "none scheduled."
4. **Urgent unreplied Gmail from last 12h** — judgment-based. Would a thoughtful friend tell her about this? Skip automated senders (Vercel, GitHub CI, marketing, newsletters, receipts, social notifications). LinkedIn *message* notifications count; LinkedIn activity digests don't. 1–3 items max, sender + one-line gist. If nothing, write "Nothing urgent."
5. **Arriving today** — search Gmail for delivery notifications indicating a package arrives today. Look for "out for delivery," "arriving today," "delivering today" from carriers (UPS, FedEx, USPS, Amazon, etc.). 1–3 items as carrier/sender + item description. If nothing, write "Nothing expected."
6. **Career** — search Gmail from the last ~7 days for job outreach: recruiter emails, LinkedIn InMail notifications about a specific role, headhunter messages. Jules is actively interested in career development — surface any genuine human reaching out about an opportunity. Skip automated job-board digests and "we're hiring" blasts. 1–3 items as sender/company + role or gist. If nothing, write "Nothing new."
7. **Heads up (time-sensitive older threads, last ~7 days)** — expiring quotes, pending RSVPs, deadlines approaching this week. Older than 12h but still actionable. 1–2 items max. If nothing qualifies, omit the entire "Heads up" section.
8. **First thing to tackle** — pick in this priority order: (a) most urgent item from `reminders.md`, (b) prep for earliest meeting today that needs it, (c) gut-call judgment.
9. **Body (one health-supportive cue)** — one specific line tied to today's weather/workout/calendar. Not generic wellness advice. See SKILL.md for angles.

**Step 3: Compose the briefing.**

Follow the exact output shape in SKILL.md. Structure:
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

Under 300 words total. Warm, crisp voice. No filler, no hedging, no meta commentary about sources.

**Step 4: Email it.**

Send via Gmail SMTP using Python — works from any machine, Mac-independent:

```python
import smtplib, datetime
from email.mime.text import MIMEText

with open('~/.claude/gmail-app-password') as f:
    password = f.read().strip()

subject = "Good morning — " + datetime.date.today().strftime("%B %-d, %Y")
body = """<the briefing markdown>"""

msg = MIMEText(body, 'plain')
msg['Subject'] = subject
msg['From'] = 'YOUR_EMAIL@gmail.com'
msg['To'] = 'YOUR_EMAIL@gmail.com'

with smtplib.SMTP_SSL('smtp.gmail.com', 465) as server:
    server.login('YOUR_EMAIL@gmail.com', password)
    server.sendmail('YOUR_EMAIL@gmail.com', 'YOUR_EMAIL@gmail.com', msg.as_string())
```

Run this as a Bash tool call: `python3 -c "<the script above with briefing inserted as body>"`

**Step 5: Verify.**

Confirm the `python3` command exited 0. If it errored, report the error but do NOT retry automatically — a second send would result in a duplicate email, which is worse than a missed one. The scheduled task runs again tomorrow.

**Guardrails (from SKILL.md, repeated for safety):**
- If a data source errors or isn't connected, keep the section but replace its content with a one-liner like `Weather: unavailable`. Never silently skip. Never fabricate.
- Never include contents from channels, threads, or docs marked private or sensitive.
- Stay under 400 words.
- Don't describe the process, tools, or the fact that this is automated. The briefing is the briefing.

**Success criteria:** an email with subject `Good morning — <today>` lands in YOUR_EMAIL@gmail.com's inbox, body matches the SKILL.md structure, under 300 words, all five "How I'll know it's working" criteria from the spec met (knows how to dress, knows the evening workout, doesn't have to open Gmail/LinkedIn/Slack to feel caught up).