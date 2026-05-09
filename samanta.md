# samanTA

**Year:** 2025
**Status:** EdTech · Live tutor flow used daily by students
**Role:** Solo developer

---

## What it is

An AI-powered quiz and tutoring platform that runs across three surfaces: a React web app, a FastAPI backend on Supabase, and a Python Telegram bot. Coaches assemble quizzes; students take them on the web or inside Telegram; OpenAI generates and grades content.

## Problem

A tutor wanted to scale her practice from 1-on-1 sessions to a small cohort, but couldn't manually grade open-ended responses or track each student's revision schedule. She needed an automated quiz engine, a way for students to take quizzes wherever they were (web or Telegram), and a reminder system that adapted to each student's revision needs.

## Architecture

Three clients sharing one auth source-of-truth and one data layer.

```
Web app (React + Vite) ─┐
                        │
                        ├──► FastAPI (Supabase JWKS auth) ◄──► OpenAI (generation + grading)
                        │
Telegram bot (Python) ──┘                │
                                         ▼
                                Supabase (Postgres, auth, RLS)
                                         │
                                         ▼
                                Stripe (Premium gate)
```

The auth flow was the trickiest part. The web client uses Supabase JS to sign in; the Telegram bot uses a `/link` command that pairs a Telegram chat ID to a Supabase user ID via a one-time code. Once linked, every Telegram handler calls a `require_linked_account` decorator that resolves `supabase_user_id` from the chat ID before the handler logic runs. This means the bot's business logic never sees an unlinked user: the framework guards it at the boundary.

Auth tokens are verified via JWKS (JSON Web Key Set) on the FastAPI side, not by hitting Supabase auth on every request. The bot gets a JWT once, then verifies it locally against the cached JWK until rotation.

Reminders run on a configurable cadence: 24, 48, 72, 120, and 168 hours after each quiz attempt, depending on student performance and the spaced-repetition curve the coach selects. The reminder service runs as a separate worker that polls a `due_reminders` view in Postgres.

Stripe gates a "Premium" tier at $9.99/mo that unlocks the AI Concierge feature (unlimited generated practice quizzes on demand). Webhook handler keeps the subscription state in sync with Postgres.

## Stack

- **Web:** React 18 + Vite, TanStack Router, Supabase JS
- **Backend:** FastAPI (Python 3.12), Supabase Postgres, JWKS auth
- **Bot:** Python (python-telegram-bot), shared FastAPI client
- **AI:** OpenAI (gpt-4o for generation, gpt-4o-mini for cheap grading paths)
- **Payments:** Stripe (subscription model, webhook-driven)
- **Deploy:** Render (FastAPI + bot worker), Vercel (web), Supabase (DB)

## Verification

- **56 of 56 unit tests pass** (auth flow, reminder cadence math, JWKS verification, Stripe webhook idempotency, decorator behaviour).
- SQL dry-runs against a staging schema before every migration.
- End-to-end smoke test on a real test user across all three surfaces (web sign-in → Telegram link → take quiz on web → take quiz in bot → reminder fires).

## Notable decisions

1. **`require_linked_account` as a decorator, not in every handler.** The first version had inline checks at the top of each handler. By the third handler I'd already written one bug (an early return that skipped the check on a specific code path). Refactored into a decorator that runs before the handler body; the framework now guarantees `supabase_user_id` is set before any business logic. This is one decorator file (~40 lines) that prevents an entire category of "user did X without being properly linked" bugs.

2. **JWKS verification cached locally, not Supabase round-trip.** Verifying a JWT against Supabase auth on every request would have added 100–200ms latency to every bot reply. Caching the JWK and verifying locally drops that to <1ms. Keys rotate on Supabase's schedule; cache TTL matches.

3. **Reminders as a separate worker with a `due_reminders` view, not cron jobs.** Cron-per-user doesn't scale and is hard to debug. The view computes "what's due right now" from the schedule + last-attempt timestamp; the worker just polls and fires. Adding a new reminder cadence is a SQL change, not a code deploy.
