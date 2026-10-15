---
prompts:
  - prompt: "Add Calendly-style booking links to this Next.js app on Supabase: hosts set their weekly availability and event types, visitors book a free time on a public page, both get an e-mail with a calendar invite, and the visitor can reschedule or cancel from a link in that e-mail."
    stack: Supabase
  - prompt: "Connect our booking pages to the host's Google Calendar and Outlook: hide times when they are busy and put every booking into their calendar."
    stack: Supabase
  - prompt: "Audit our scheduling module: people get double-booked, reminders arrive for meetings that were moved, and around the clock change some days have no slots."
---

# Prompts

What an operator types after installing this skill, in their own words. An agent eval installs the skill
into an empty Next.js app, gives the agent one of these prompts and no further help, then type-checks, builds
and tests the result; the first prompt runs before every release. The results are the other files in this
folder. Section 10 of [STANDARD.md](https://github.com/timerise-ai/skills/blob/main/STANDARD.md) says how a
run is made. The prompts and the newest runs are on
[the skill's page](https://timerise.ai/skills/calendar-scheduling-links) on timerise.ai.
