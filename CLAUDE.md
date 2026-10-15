# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this repository is

An [Agent Skill](https://agentskills.io) package: markdown only. There is no `package.json`, build or dev
server here, and nothing in this repository executes. It teaches an agent to build self-hosted scheduling
links in *someone else's* **Next.js App Router** app, on Supabase or Firestore. The version of record is the
git tag.

The commands and code in `references/` describe the app the agent will generate, not this repository. The
one thing checked here is that the templates compile and their tests pass, in a scratch project; the recipe
is below.

The skill was written by the engineer who owns the module, and audited against the earlier implementation.
`references/provenance.md` is the rationale layer and the ledger of that audit: what the audit changed and
how the templates verify it, what was kept deliberately with the reason it is safe, and what was designed
here and has never run in production. Read it before "simplifying" anything.

Sibling directories under `../` (`bookable-events`, `booking-kiosk`, `stripe-connect-subscriptions`, and so
on) are other skills, not dependencies. They are referenced by name from `SKILL.md`, `README.md` and
`references/extensions.md`. The standard every skill follows is `../skills/STANDARD.md`.

## Structure

- `SKILL.md`: entry point, loaded whole on every activation, so it stays between 130 and 160 lines, the
  closing index line aside. The frontmatter `description` is the trigger surface. It carries a short
  adaptation contract table; the full contract is `references/adaptation.md`.
- `README.md`: the human-facing front door, in the section order every Timerise skill shares.
- `CHANGELOG.md`: Keep a Changelog, newest release first. The version lives here, in the README's
  current-release line, and in the git tag, and the three agree.
- `LICENSE`: MIT, identical to the siblings'.
- `references/*.md`: seventeen topic files plus `provenance.md`. `data-model.md` holds the rename table;
  `time-and-slots.md` the pure core; `booking-flow.md` the service and the store seam; `supabase.md`,
  `supabase-stores.md` and `firestore.md` the stores; `jobs-and-notifications.md`, `webhooks.md`,
  `calendar-sync.md` and `calendar-providers.md` the side effects; `api-routes.md` and `booking-page-ui.md`
  the surface; `operations.md` running it; `extensions.md` what is not built; `testing.md` and
  `testing-flows.md` the suites; `adaptation.md` the seam; `provenance.md` the ledger.
- `evals/`: `prompts.md` holds what an operator types after installing, in their words; the first prompt
  is the agent eval run before every release. Every other file there is one eval run: measured frontmatter
  that is never edited, then the notes of the person who ran it. Add a prompt rather than rewording one that
  has results. The procedure is section 10 of the index's STANDARD.md.
- `.github/workflows/agent-eval.yml`: the caller of the index's reusable eval workflow, copied verbatim from
  section 10 of the standard and run on every published release and on a maintainer's dispatch. It is the
  same in every skill; never edit it, and never add a trigger on `push` or `pull_request`.

## Editing conventions

- **Code blocks name their destination on the first line**: `// lib/scheduling/slot-engine.ts: description`.
  A block that continues a file already introduced omits it.
- **The code blocks are compiled and run.** Write each block to its path in a scratch project (never this
  repo) with `next`, `react`, `zod`, `@supabase/supabase-js`, `@supabase/ssr`, `firebase-admin`,
  `@vercel/config` and `vitest`, a `tsconfig.json` that is `strict`, sets `noUncheckedIndexedAccess` and
  maps `"@/*": ["./*"]`, and run:

  ```bash
  npx tsc --noEmit -p . && npx vitest run && npx next build
  npx vitest run test/slot-engine.test.ts
  firebase emulators:exec --only firestore --project demo-scheduling "npx vitest run test/store-contract.test.ts"
  SUPABASE_URL=http://127.0.0.1:54321 SUPABASE_SERVICE_ROLE_KEY=<from supabase status> npx vitest run test/store-contract.test.ts
  ```

  The second line runs one suite. The store contract always runs against the in-memory store. It runs against
  Firestore when `FIRESTORE_EMULATOR_HOST` is set (the emulator needs Java and a `firebase.json` with port
  8098), and against Supabase when both env vars are set (`supabase start` with the migration applied). The
  SQL is checked by applying it to that local stack; the anon key must then be refused with `42501`
  (`references/testing.md`). `public/scheduling-embed.js` is checked by reading. Do not add a `package.json`
  to this repository.
- **Test counts are claims.** `testing.md` and `testing-flows.md` state 8 + 13 + 12 + 18 = 51 unit tests and
  5 contract tests per store; `README.md`, `SKILL.md` and `CHANGELOG.md` repeat them. Change a suite, change
  the numbers.
- **Keep the three indexes in sync** with `references/`: the quick start and the reference directory in
  `SKILL.md`, and the file table in `README.md`.
- **Identifiers are shared across files.** `computeSlots`, `zonedToInstant`, `localDateOf`, `addDays`,
  `SchedulingStore`, `CalendarStore`, `CalendarProvider`, `CalendarWriter`, `createBooking`,
  `rescheduleBooking`, `cancelBooking`, `jobsFor`, `runDueJobs`, `manageTokenFor`, `channelToken`,
  `currentHostId`, `slotKey`, `chainId`, the job kinds and the SQL function names. Rename in all files or
  none.
- **Two kinds of name.** The domain words the host renames are the rename table in `data-model.md`. The
  identifiers above are the authoring contract of this repository.
- **The non-negotiables are never presented as optional.** The five hard rules in `SKILL.md`, the five
  non-negotiables in `README.md` and the list in `adaptation.md` are one list, in one order. Changing one is a
  MAJOR release.
- **Measured and vendor numbers are load-bearing.** `MAX_CANDIDATES` 20000, `MAX_WINDOW_DAYS` 62, reminder
  offsets 1440 and 60 minutes, 8 job attempts with backoff capped at 6 hours, 120-second job and sync leases,
  `SYNC_WINDOW_DAYS` 120, `STALE_SYNC_MINUTES` 30, watch renewal 24 hours before expiry, Graph `clientState`
  128 characters, Outlook subscriptions at most 10,080 minutes, the 22-character signature, the 5-minute
  webhook tolerance. Change one, change every file that states it.
- **Env var names in `references/operations.md` are canonical.**
- **A template that needs something new from the host adds a row** to the seam contract in `adaptation.md`
  and `SKILL.md`.
- **Do not remove the odd-looking parts.** Each is a ledger entry in `provenance.md`.
- **Mark additions as additions.** Anything designed in the skill and never run in the earlier implementation
  belongs in the "Designed here, not yet run in production" section of `provenance.md`, stated as such, or in
  `extensions.md` as a design. The skill's credibility is that it distinguishes the two.
- **Evals are not skill content.** A new prompt or an eval result is committed as `chore(evals): ...`, never
  causes a version bump and never rides in a release commit. After each eval, the README's eval badges point
  at the newest prompt 1 result per agent, updated in the commit that adds the run's notes.
- **Origin wording.** Call the codebase the audit ran against "the earlier implementation", and follow
  section 2 of the standard: none of its banned words, no denials, no corpus figures, and no defect counts or
  failure stories on the front door (README intro, `SKILL.md`, the description, this file).
- **Plain punctuation.** No em-dashes, en-dashes, arrows in prose, middle dots, ellipsis characters or smart
  quotes anywhere in this repository's markdown; diagrams are plain ASCII. `LC_ALL=C grep -rn '[^ -~	]'
  --include='*.md' .` must print nothing. Prose wraps at 110 columns; table rows and commands stay on one
  line.
- **Claims are verifiable.** A changed factual claim says how it was verified. Never from memory.
- **Commits follow Conventional Commits**, and no file or commit message names a tool or a model as author.
