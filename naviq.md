# Naviq (TrainHire)

**Year:** 2026
**Status:** MVP live at [naviq.vercel.app](https://naviq.vercel.app) · Active build
**Role:** Founder

---

## What it is

An intern hiring and training platform. Paste a job description and it generates a 20-lesson training academy, then hands that to interns through a gamified personal cabinet. Admins enrol interns from their own portal, watch progress, and route data through Google Sheets, amoCRM, and a Telegram bot.

It was built for the UzCombinator program, and it ranks candidates on a rolling 1 to 10 quiz score.

## Problem

Training interns for a specific role usually means someone writes a curriculum by hand, every time. Naviq generates the curriculum from the job description and automates enrolment and progress tracking around it.

## Architecture

Three role-scoped portals sharing one Postgres and auth layer.

```
Dev Panel ─────────┐
                   ├──► Supabase (auth, Postgres, storage) ◄──► Course generator (Anthropic, OpenAI fallback)
Employer Portal ───┤                                                     │
                   │                                                     ▼
Internship Academy ┘                                             Notion API (publishes lessons)
                                                                         │
                                                                         ▼
                                                              Google Sheets + amoCRM
```

The course generator is the core of the product. Given a job description, it produces 20 structured lessons. Anthropic Claude is the primary model with OpenAI as a fallback, and output is validated with Zod.

The personal cabinet wraps lessons in XP levels. Levels scale at `50 · n²`, so level 5 needs 1,250 XP and level 10 needs 5,000 XP.

Streaks use **idempotent UTC keys**: each streak event is keyed by user and UTC date, so crossing midnight mid-session can't count twice.

## Stack

- **Frontend:** Next.js 16 (App Router), React 19, TypeScript 5, Tailwind 4
- **Backend:** Supabase (auth, Postgres, storage), Zod for validation
- **AI:** Anthropic Claude, OpenAI fallback, Notion API for publishing
- **Integrations:** Google Sheets, amoCRM, Telegram bot
- **Languages:** English and Russian
- **Deploy:** Vercel

## Verification

- **56 Vitest tests across 7 files**, covering LMS core logic, gamification math, i18n key parity, schemas, navigation roles, and training label fallback.

## Notable decisions

1. **One primary model with a fallback, not load balancing.** OpenAI only steps in when Claude can't deliver, so one academy doesn't end up with lessons in two different voices.

2. **Idempotent UTC streak keys.** Keying on the UTC date makes streak writes safe to repeat and independent of the student's local clock.

3. **Notion as the publishing surface.** Lessons publish through the Notion API, so the app can focus on enrolment, progress, XP, and streaks.
