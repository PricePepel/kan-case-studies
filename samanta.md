# samanTA

**Year:** 2026
**Status:** Live at [ta.saman.academy](https://ta.saman.academy) · Bloomin product
**Role:** CTO, since March 2026

---

## What it is

An AI-powered quiz and tutoring platform for schools and academies, delivered mainly through Telegram. It has three parts: a web frontend, a FastAPI backend, and a Telegram bot with a Mini App quiz player.

It launched in March 2026 with about 80 daily users and has reached 117 unique users, 70 deployed quiz bots, and 562 quiz sessions. Its usage results helped Bloomin win a €20,000 grant from EdTech4BY.

## Problem

Schools needed quizzes built from their own materials, delivered where students already are, and a way to bring back students who drift off. So samanTA turns a PDF into a validated quiz, runs it inside Telegram, and follows up automatically.

## Architecture

```
Web app (React + Vite) ──┐
                         ├──► FastAPI (RS256 JWT via Supabase JWKS) ◄──► OpenAI (gpt-4o)
Telegram bot + Mini App ─┘                 │
                                           ▼
                                  Supabase (Postgres, Auth, RLS)
```

**Quiz generation.** A PDF goes through `pdf_processor.py`, then `gpt-4o` drafts a structured JSON quiz with `single_select`, `multi_select`, and `matching` questions. A deterministic schema check runs before anything is saved.

**Notification escalation.** A 5-stage ladder runs Reminder, Gentle Nudge, Streak Urgency, Empathy, then Last Call. Frequency is user-configurable at 24, 48, 72, 120, or 168 hours, with timezone-aware delivery windows. A SQL view, `notification_settings_due`, computes `is_due` declaratively, and APScheduler runs a 60-second job loop in the bot.

**Streaks.** A midnight job calls `/streak/reset-all-daily`. The database trigger `reset_notification_stage_on_activity` resets the escalation state the moment a student returns.

**Auth.** There are two tiers, one for the bot service and one for web users. In the bot, a `require_linked_account` decorator guards handlers, so their logic only runs for a linked account.

## Stack

- **Web:** React, Vite, Tailwind, shadcn/ui
- **Backend:** FastAPI, Supabase
- **Bot:** Python, python-telegram-bot, APScheduler
- **AI:** OpenAI gpt-4o and gpt-4o-mini
- **Deploy:** Vercel (web), Render (bot)

## Notable decisions

1. **Due notifications as a SQL view.** The rule for who is due lives in one declarative place, and the bot's job loop only has to read it and send.

2. **A database trigger for resetting escalation.** Any activity resets the ladder, whichever code path recorded it, so a returning student never receives a "Last Call" meant for someone who left.

3. **Telegram first.** The quiz player runs inside Telegram's WebView as a Mini App, with deep links from chat straight into a quiz session, so students don't need to install anything else.
