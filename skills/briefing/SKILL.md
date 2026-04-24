---
name: briefing
description: Deliver a warm, crisp morning briefing — weather, calendar, urgent messages, one priority, and a signoff. Use when the user says "good morning", "morning briefing", "brief me", "gm", or anything signaling they want a snapshot of their day before it starts, even when phrased casually. The briefing is personal and tailored — don't substitute a generic day-planner response.
---

# Morning briefing

Produce a one-screen summary (<400 words, plain markdown) so the user can read it in ~30 seconds and feel caught up without opening Gmail, LinkedIn, or Slack.

## Output shape

Use this exact structure, in this order:

```
Good morning —

[Weather: one sentence with a dressing cue]

**Today**
- [Calendar events with times; flag any that need prep]

**Arriving today**
[1–3 items: carrier + item description, or "Nothing expected."]

**Evening workout**
[Class + time, or "none scheduled"]

**Body**
[One line — a health-supportive cue for today. See logic below.]

**Needs a reply**
- [1–3 urgent unreplied items from the last 12h, sender + gist]

**Career**
- [1–3 job outreach emails from the last ~7 days, sender/company + role gist, or "Nothing new."]

**Heads up** *(include only if non-empty — otherwise drop the whole section)*
- [Aging-but-still-actionable threads: expiring quotes, pending RSVPs, approaching deadlines]

**First thing to tackle**
[One item — see priority logic below]

Until tomorrow.
```

Voice: warm, crisp. Short sentences. No filler, no hedging, no "I hope this helps." Write to the user, not about them.

## Data to gather

Pull these in parallel when possible.

1. **Weather** — user's city, in their preferred units. Use WebFetch or WebSearch. Get current temp, conditions, high/low, and chance of rain. Translate that into a real dressing cue — "grab a jacket," "wear layers," "sunglasses," "umbrella" — not just the numbers. The dressing cue is the point; the numbers support it.

2. **Calendar** — today's events across *all* calendars, including any shared/sub calendars. Use the Google Calendar MCP. First call `list_calendars` to enumerate them, then `list_events` per calendar with `startTime` / `endTime` (ISO 8601 with timezone offset, e.g. `2026-04-23T00:00:00-04:00`) — the params are *not* `timeMin`/`timeMax`. Subcalendars that return no events today are genuinely empty; don't mention them in the output. For each event, decide whether it needs prep. Client calls, interviews, 1:1s, and presentations usually do; lunches, blocks, and recurring standups usually don't. Flag prep items inline, e.g. "2:00 — call with Acme (needs agenda)".

3. **Evening workout** — if the user has a fitness studio or class pattern, their workouts may land on their primary calendar via an auto-import task. **Check the calendar first**: look for any event today whose title/description mentions a known studio name, class type (pilates, yoga, flow, etc.), or studio address. Surface it with class + time + (location if useful). Only fall back to a Gmail search if nothing turns up on the calendar. If neither finds anything, write "none scheduled."

4. **Urgent unreplied messages** — search Gmail from the last 12h. Use judgment, not keyword matching. The bar is: *would a thoughtful friend tell the user about this before they open their inbox?* Surface things directly addressed to them with a question, deadline-flagged ("today," "EOD," "by Thursday," quote expires in X hours), or from a real person who looks like they expect a reply.

   Skip automated senders entirely — Vercel/GitHub/CI notifications, marketing, newsletters, receipts, social notifications, calendar invites, out-of-office auto-replies, delivery updates. A simple test: if the sender domain is a bot/service and the message is templated, it doesn't count as "needs a reply" even when it sounds urgent ("deploy failed!!"). LinkedIn *message* notifications (someone sent a DM) do count — those are a real person reaching out. LinkedIn activity digests ("X liked your post") do not.

   Show 1–3 items max, each as *sender — one-line gist*. If nothing urgent, write "Nothing urgent."

5. **Arriving today** — search Gmail for delivery notifications indicating a package will arrive today. Look for phrases like "out for delivery," "arriving today," "delivering today," or "will be delivered" from carriers (UPS, FedEx, USPS, Amazon, OnTrac, etc.) and retailers. Show 1–3 items as *carrier/sender — item or order description*. If nothing, write "Nothing expected."

6. **Career** — search Gmail from the last ~7 days for job outreach: recruiter emails, LinkedIn InMail notifications (someone reached out about a role), headhunter messages, or emails from real people or firms about a specific opportunity. Surface anything that looks like a genuine human reaching out about a role — even if it arrived a few days ago. Skip automated job-board digests ("Jobs you might like"), LinkedIn activity digests, and generic "we're hiring" marketing blasts. Show 1–3 items as *sender / company — role or gist*. If nothing new in the last 7 days, write "Nothing new."

7. **Body (one health-supportive cue)** — weave health into the routine with one concrete, specific line. Shape it to the day's context — weather, workout, calendar — not generic wellness advice. Pick whichever angle is strongest:

   - **Pre/post-workout cue** when a class is scheduled — hydration, fuel timing, recovery note.
   - **Outdoor/movement prompt** when the weather invites it or the day is light on movement — e.g., "walk the returns drop if you can," "pair lunch with a loop around the block."
   - **Micro-reset** on heavy meeting days — a breath, a step outside between calls, a screen break.
   - **Hydration/sleep/fuel** cue when the day's rhythm calls for it.
   - **Mental reset** (short meditation, phone-down window, journal prompt) when nothing else fits.

   One line. Specific, warm, tied to something real in the user's day. Avoid anything that sounds like it came from a wellness app ("remember to hydrate!", "self-care is key"). If you can't generate one that's genuinely useful and specific, write "Take one outside break today — even five minutes counts." as a fallback.

8. **Heads up (time-sensitive threads, any age)** — scan Gmail from the last ~7 days for threads that are still *active and time-sensitive today*, regardless of when they arrived. Examples: a quote that expires in the next ~72h, an RSVP not yet sent, a deadline approaching this week, a reservation hold about to release. The distinguishing signal vs. "Needs a reply" is age: these items are older than 12h but still require action soon.

   Keep it to 1–2 items max, each one line. If nothing qualifies, **omit the entire "Heads up" section from the output** — don't write "Nothing pending." Empty means the section disappears.

9. **First thing to tackle** — pick in this order:
   1. Most urgent item from the user's reminders/to-do file (if one exists — see Context files).
   2. Prep for the earliest meeting today that needs it.
   3. Your gut-call judgment, weighing calendar + reminders + urgent messages.

## Guardrails

- **Unavailable sources**: if a data source errors, times out, or isn't connected, keep the section but replace its content with a one-liner like `Weather: unavailable` or `Calendar: unavailable`. Don't silently skip. Don't fabricate.
- **Never include** contents from channels, threads, or docs marked private or sensitive.
- **Word budget**: stay under 400 words total. If sections bloat, trim — fewer words per item, not fewer sections.
- **No meta**: don't describe the data sources, the process, or that you're following a skill. This is the briefing itself.

## Context files (optional)

If the user maintains these files, read them for richer context:

- A reminders/to-do file (e.g. `~/reminders.md`) — used for "first thing to tackle" logic.
- A schedule-preferences file (e.g. `~/schedule-preferences.md`) — how the user likes their day structured.

Configure the paths by editing this skill or via your CLAUDE.md.
