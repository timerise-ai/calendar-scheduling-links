# calendar-scheduling-links

[![Agent Skills](https://img.shields.io/badge/Agent_Skills-open_format-059669)](https://agentskills.io)
[![skills.sh](https://img.shields.io/badge/skills.sh-npx_skills_add-059669)](https://www.skills.sh)
[![Claude Code](https://img.shields.io/badge/Claude_Code-compatible-059669)](https://docs.claude.com/en/docs/claude-code/skills)
[![Codex CLI](https://img.shields.io/badge/Codex_CLI-compatible-059669)](https://developers.openai.com/codex/skills)
[![Gemini CLI](https://img.shields.io/badge/Gemini_CLI-compatible-059669)](https://github.com/google-gemini/gemini-cli/blob/main/docs/cli/skills.md)
[![Eval claude-opus-5-5](https://img.shields.io/badge/eval_claude--opus--5--5-pass_7%2F8-059669)](evals/2026-10-15-claude-code-p1.md)
[![Eval gpt-6.1-sol](https://img.shields.io/badge/eval_gpt--6.1--sol-pass_3%2F8-059669)](evals/2026-10-15-codex-p1.md)
[![Eval gemini-3.8-flash](https://img.shields.io/badge/eval_gemini--3.8--flash-pass_6%2F8-059669)](evals/2026-10-15-gemini-cli-p1.md)

An [Agent Skill](https://agentskills.io) that teaches an agent to build **self-hosted scheduling links** in a
**Next.js App Router** app on Vercel: a public booking page per host and event type, weekly availability with
date overrides, buffers, minimum notice, daily limits and a rolling horizon, one-on-one and group event
types, Google Calendar and Microsoft 365 busy-time sync with bookings written back, confirmation e-mails with
an ICS invite, reminders, reschedule and cancel by a private link, signed webhooks and an embeddable widget.
An open alternative to Calendly that you own and extend, on Supabase by default or on Firestore.

A scheduling page is a list of free times and a button; the hard part is the write. The skill computes slots
with **one pure function that the page lists and the booking route calls again for the requested start**,
writes each booking **under one lock per host with a Postgres exclusion constraint as the backstop**, and
turns every side effect, e-mail, calendar write, reminder and webhook, into **a leased, idempotent job
written in the same transaction as the booking**.

This skill was written by the engineer who builds the scheduling core of a booking platform, and audited
against the earlier implementation of that core. The templates hold the properties a scheduling module has
to hold: a start is booked only if the engine offers it at write time; of any number of simultaneous
requests for one slot, one wins; local days survive both DST changes; a reschedule moves the same calendar
event and the same invitation; a reminder belongs to the start it was made for; calendar access, manage
links and webhooks cannot be forged. 52 unit tests and a store contract, run on local Supabase and the
Firestore emulator, state each one, and [`references/provenance.md`](references/provenance.md) has the
record.

## Install

One command, via the [skills.sh](https://www.skills.sh) CLI, which installs the skill into every
skills-compatible agent it detects, including Claude Code, Codex CLI and Gemini CLI:

```bash
npx skills add timerise-ai/calendar-scheduling-links
```

Name the agents instead with `-a`, for example
`npx skills add timerise-ai/calendar-scheduling-links -a claude-code -a codex`.

### Manual install

Nothing here is Claude-specific: the skill is a plain [Agent Skills](https://agentskills.io) folder,
`SKILL.md` plus markdown references with no file that calls a model, so cloning it into an agent's skills
directory is all an install is. For Claude Code:

```bash
git clone https://github.com/timerise-ai/calendar-scheduling-links.git ~/.claude/skills/calendar-scheduling-links
```

To scope it to a single project instead, clone it into that project's `.claude/skills/` directory. For another
agent, clone into that agent's skills directory, or symlink the Claude Code copy so one `git pull` updates
every agent:

```bash
mkdir -p ~/.agents/skills
ln -s ~/.claude/skills/calendar-scheduling-links ~/.agents/skills/calendar-scheduling-links
```

Update with `git pull` in the clone. The current release is **0.1.1**. See [CHANGELOG.md](CHANGELOG.md).

## Activation

The skill activates on tasks such as "add Calendly-style booking links to this app", "let clients book a time
with me", "sync availability with Google Calendar", or "our booking page double-books". Each host matches
skills its own way, so on a first run invoke it explicitly: `/calendar-scheduling-links` in Claude Code,
`$calendar-scheduling-links` in Codex CLI, and through `/skills` in Gemini CLI.

## What's inside

| File | Contents |
|---|---|
| `SKILL.md` | Entry point: trigger, architecture, adaptation contract, critical facts, hard rules, quick start, reference directory |
| `references/data-model.md` | Vocabulary, types, what each field is for, records, the rename table |
| `references/time-and-slots.md` | Time helpers on `Intl`, the slot engine and the order of its rules |
| `references/booking-flow.md` | The store seam, the booking service, errors, host-side validation |
| `references/supabase.md` | Migration with RLS, grants, the exclusion constraint and the atomic SQL functions |
| `references/supabase-stores.md` | `supabaseStore` and `supabaseCalendarStore` |
| `references/firestore.md` | The Firestore store, its lock document, indexes and rules |
| `references/jobs-and-notifications.md` | The jobs table and runner, e-mail, ICS |
| `references/webhooks.md` | Signed webhooks, the payload and the private-address guard |
| `references/calendar-sync.md` | The provider contract, OAuth with signed state and PKCE, sync, watches, the writer |
| `references/calendar-providers.md` | Google Calendar and Microsoft Graph, with the vendor facts they rely on |
| `references/api-routes.md` | Runtime wiring, the auth seam, tokens and sealing, every route |
| `references/booking-page-ui.md` | Public page, widget, manage page, embed script, dashboard rules |
| `references/operations.md` | Environment, Vercel Cron or `pg_cron`, Google and Microsoft setup, deploy, monitoring |
| `references/extensions.md` | Round robin, collective, routing, payments and more, with prompts |
| `references/testing.md` | Vitest setup, fixtures, the time and engine suites, running against real stores |
| `references/testing-flows.md` | The booking, edges and store contract suites |
| `references/adaptation.md` | Host prerequisites, seam contract, order of work, the non-negotiables |
| `references/provenance.md` | The audit ledger: fixed, kept deliberately, designed here |
| `evals/` | `prompts.md` with what an operator types, and one file per agent eval run |
| `.github/workflows/agent-eval.yml` | Runs the agent evals on every published release |

The seam is the `SchedulingStore` and `CalendarStore` interfaces plus `currentHostId()` and `Mailer`. They
bound everything the host app supplies: its database, its auth, its e-mail provider. The look of every
screen is the host app's too.

## The five non-negotiables

1. **The write recomputes the slot with the engine that listed it.** A start is booked only if
   `computeSlots` offers it at write time. The booking suite books what the page listed and refuses an
   off-grid start, a start inside the notice and one outside hours.
2. **Every booking write is atomic per host.** Seats, the daily limit and overlap are checked under one lock
   per host, with the overlap constraint as the backstop. The store contract sends four simultaneous
   requests for one slot to each store and expects one winner.
3. **Side effects are jobs, written with the change and run after it.** No e-mail, calendar call or webhook
   inside the transaction; every job idempotent and re-checking the booking. The booking suite runs the
   queue twice and expects one send, retries a failing mailer, and plants a stale reminder that must not be
   sent.
4. **Instants in storage, zones only at the edges.** Store UTC instants; convert with an explicit IANA zone,
   iterate local dates, never add 24 hours. The time and engine suites cover both DST changes, a 5:45 offset
   and a late-evening walk across a spring-forward night.
5. **Secrets stay sealed and links stay unforgeable.** Manage tokens stored as hashes, OAuth tokens and
   webhook secrets encrypted, OAuth state and channel tokens signed, cron auth failing closed. The edges
   suite checks each.

Everything else, the look, the words, the e-mail provider, the extra fields, is the host app's.

## Requirements

Next.js App Router on the Node.js runtime, TypeScript, `zod` 4, and `@supabase/supabase-js` with
`@supabase/ssr` (or `firebase-admin`). A scheduler that calls one URL every five minutes. For calendars, a
Google OAuth client and a Microsoft Entra app registration; Google requires verification of the calendar
scopes before the public can connect.

## Verification

The TypeScript blocks compile as one project under `strict` and `noUncheckedIndexedAccess`, the app they
describe passes `next build`, and the suites in [testing.md](references/testing.md) pass under Vitest: 47
unit tests, plus 5 contract tests on each store, run on local Supabase and on the Firestore emulator. The
SQL was applied to a local Supabase stack, which also served an end-to-end check over HTTP: listing, booking,
the second request for the same time refused with 409, the ICS, reschedule, the old link refused, and the
cron refusing a request without its secret.

## Not this

| Problem | Go to |
|---|---|
| Payment for a booking, deposits, no-show holds, ticketed events with seats on one date | `bookable-events` |
| A marketplace that pays providers | `stripe-connect-subscriptions` |
| A walk-up booking terminal | `booking-kiosk` |
| Host notifications in Slack | `slack-ai-bot` |
| Knowing who opened a booking link | `visit-logger` |
| Invoices for paid sessions | `invoicing` |
| Round robin, collective events, routing forms | [extensions.md](references/extensions.md), no templates yet |

## Contributing

Issues and pull requests are welcome here. Pure markdown, with no build step, but the code blocks are checked:
every block names its destination on the first line, every TypeScript block is written to compile as one
project under `strict` and `noUncheckedIndexedAccess`, and the suites pass under Vitest. Claims in this skill
are meant to be verifiable: if you change a factual claim, say how you verified it, whether against the
Google Calendar API reference, the Microsoft Graph reference, the Supabase or Vercel documentation, RFC 5545
and RFC 5546, the IANA time zone database, or a reproduction.

Adding, removing or renaming a file in `references/` means updating the quick start and the reference
directory table in `SKILL.md`, the file table above, and any relative cross-links. Every odd-looking part of
the templates is there for a reason, and `references/provenance.md` is the ledger that must stay truthful:
read it before simplifying anything, and add an entry for anything you change. Commits follow Conventional
Commits and releases follow [STANDARD.md](https://github.com/timerise-ai/skills/blob/main/STANDARD.md) in the
index; `CLAUDE.md` carries the full editing conventions.

## Part of the Timerise Skills

This is one of the [Timerise Skills](https://github.com/timerise-ai/skills): modules for **Next.js App
Router** apps written by our own senior engineers from the modules they have shipped, not synthetic, each
published as its own repository and indexed there. They share one layout, so an agent that has read one knows
how to read the next: a `SKILL.md` entry point, `references/` loaded on demand, and a seam contract carrying
the module's non-negotiables.

## Author

Built and maintained by [Timerise](https://timerise.ai).

## License

MIT. See [LICENSE](LICENSE).
