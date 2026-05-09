# Naviq (TrainHire)

**Year:** 2025
**Status:** Production · MVP live on dev at naviq.app
**Role:** Solo developer

---

## What it is

A B2B SaaS that turns a job specification into a complete 20-lesson onboarding academy and hands it to interns through a gamified personal cabinet. Admins enrol interns from a dedicated portal, watch progress, and route data through Google Sheets, amoCRM, and a Telegram bot.

## Problem

TrainHire onboards interns into specific roles (sales, junior eng, ops). Their existing process: a coach manually wrote a 20-lesson curriculum per role, dragged it into Notion, and sent intern access individually. This took 3–4 hours per intern per cohort. They wanted the curriculum generation, intern enrolment, and progress tracking automated end-to-end so a single admin could onboard 50 interns in an afternoon.

## Architecture

Three surfaces sharing one Postgres + auth layer.

```
Admin Portal ─────┐
                  ├──► Supabase (auth, Postgres, RLS) ◄──► Course Generator (Anthropic, OpenAI fallback)
Intern Cabinet ───┤                                                │
                  │                                                ▼
Telegram Bot ─────┘                                          Notion API (publishes lessons)
                                                                   │
                                                                   ▼
                                                          Google Sheets + amoCRM
```

The course generator is the value engine. Given a job spec (role, seniority, scope), it produces 20 structured lessons with learning objectives, exercises, and grading rubrics. Anthropic Claude is primary; OpenAI is the fallback when Claude rate-limits or returns malformed output. Output goes through a Zod validator before persisting; invalid generations trigger automatic regeneration with a tightened prompt.

The gamified personal cabinet wraps each lesson in an XP system. XP awards on a 10/20/30 step curve (start, mid, complete). Level scaling at `50 · n²` (so level 5 needs 1,250 XP, level 10 needs 5,000 XP). This is quadratic on purpose: prevents grinding-fatigue at low levels and progression cliffs at high levels.

Streaks track the calendar-aware "days active in a row" counter. The non-obvious decision was using **idempotent UTC streak keys**: every streak event is keyed by `user_id + utc_date`, so a user crossing midnight in the middle of a session doesn't get double-counted, and timezone-shifted clients can't game the streak by adjusting their device clock.

## Stack

- **Frontend:** Next.js 16 (App Router), React 19, TypeScript 5, Tailwind 4
- **Backend:** Supabase (auth + Postgres + RLS + Edge Functions), Zod for validation
- **AI:** Anthropic Claude (primary), OpenAI (fallback), Notion API (publishing)
- **Integrations:** Google Sheets, amoCRM, Telegram Bot API
- **Deploy:** Vercel (web), Supabase (DB)

## Verification

- **56 Vitest tests** covering: course generator output validation, XP curve math, streak idempotency, intern enrolment flow, RLS policy enforcement.
- Edge case smoke tests: malformed Claude response, Notion API rate-limit, double enrolment, mid-session timezone change.
- MVP shipped fast (3 weeks scope), live on dev.

## Notable decisions

1. **Anthropic primary + OpenAI fallback, not load-balanced.** Claude has stronger structured output following for the curriculum schema. OpenAI is only invoked on Claude failure (rate-limit, malformed JSON, validation rejection). This avoids the "two slightly different curricula" inconsistency that load-balancing would create.

2. **Idempotent UTC streak keys.** The first version used local timezone and broke for one tester who travelled between Tashkent and Bangkok mid-cohort. Reworked the key as `(user_id, utc_date)` with a unique constraint at the DB level. This is the kind of bug that's invisible until a real user surfaces it; doing it right the first time saves a Sev-2 in production.

3. **Notion as the lesson reading surface, not custom UI.** The interns read lessons in Notion (familiar interface, mobile-friendly, search built in). The cabinet only handles enrolment, progress, XP, and streaks. This cut about 2 weeks of UI work for content rendering and let the focus stay on the engagement loop.
