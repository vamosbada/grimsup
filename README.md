# Grimsup — a production system I own end to end, with real users

**[grimsup.com](https://grimsup.com)** · live since 2026-08-29 · in production with real users

> **The code is private** — it handles real users' photos. This repository is the system's design, operations, and incident record.

> 🇰🇷 한국어: [README.ko.md](./README.ko.md)

Grimsup is the **operations system for one offline children's art school.** Before this, the school
ran on **scattered tools**: photos went out through a messenger app, notices through another app,
questions by phone. None of it belonged to the school. This project pulls that into **one system the
school owns.**

Live today: Google and Kakao login with teacher approval · a per-child gallery · a community board
(teacher notices, kids' drawings, a talk board with private posts, comments, emoji reactions) ·
phone push notifications · an admin screen · a public page that search engines can find ·
audit log and structured logging · error alerts and daily backups.
Kids and parents sign up and post on their own; the teacher approves who gets in.
An AI agent comes next. **One person owns the whole cycle**: planning, design, code,
deployment, real-user support, and incident response.

> This repo **explains the system. It does not contain the code.**
> It is less about *what* was built and more about **how it runs, why it was built this way, and what
> broke and how it was fixed.**
>
> Common implementations that look the same everywhere are left out. Only **decisions made in this
> project** are written down: where there was an alternative, and why this side won.

---

## What this project is trying to show

**Completeness of the cycle, not scale.** This is a small service for one school. Selling it as
"operations experience at scale" would cost trust. What it does show is **one person owning a small
farm end to end**: from planting seeds to dealing with pests. Scale can be learned at a company.
The feel of owning the whole cycle is harder to teach.

So there is one yardstick. Not *"does this traffic need it?"* but **"is this piece missing from the
production cycle?"** Even with few users, audit logs, incident records, alerts, and backups go in.
Tools that only scale demands (Kubernetes and friends) stay out.

**Success is not measured in users.** A school runs on **trust** with the families who come.
This service handles children's photos. One photo shown to the wrong person ends that trust.
So "is it convenient?" came second to **"can it be shown to the wrong person?"**

📌 **No numbers yet.** Visitor counts are not the point. Once DAU and per-feature usage data have
built up, real values will be added here along with operating metrics (uptime · incident count ·
time to recover · cost).

---

## Features

This is **one web app for both phone and PC**, not an app-store app. Parents mostly use phones:
add it to the home screen and it opens like an app and receives push. The teacher uses both phone and
PC. The screens are designed separately for each, but the code and the deployment are one.

| Feature | What | Who | Status |
|---|---|---|---|
| **Login · onboarding** | Google or Kakao login → choose parent, student, or visitor → parents and students wait for teacher approval (for a student, the teacher records guardian consent first). One person can link both logins to one account. Google and Kakao prove *whose account* this is. The teacher proves *who this is at the school* | anyone | ✅ live · [details](./docs/features/auth.md) |
| **Gallery** | A photo the teacher uploads is visible only to that child's guardians. Per-child album · emoji reactions · read receipts · scroll back to old work | parents ↔ teacher | ✅ live · [details](./docs/features/gallery.md) |
| **Community** | Four boards: teacher notices · school news · kids' drawings · a talk board with private posts. Rich-text posts · comments · emoji reactions · unread badges. Replaced the external app the school used for notices | whole school | ✅ live · [details](./docs/features/community.md) |
| **Web push** | Install to the home screen and the phone rings when a photo, a new post, or a comment arrives | parents · students · teacher | ✅ live |
| **Admin** | Approve parents and students · register students · link guardians · full member roster · manage accounts | teacher | ✅ live |
| **School page** | Location · classes · photos. Public page with search and link previews · privacy policy | anyone | ✅ live |
| **Monitoring · backups** | Error and uptime alerts by email · daily database backup, with a restore actually tested | developer | ✅ live |
| **Grimmy (AI Agent)** | An agent with **different tools for parents, teacher, and visitors**. From answering questions and handling absence/make-up requests to the teacher's morning briefing and pickup alerts. Not a chatbot wrapped around an API — an agent with permission boundaries, audit, and tracing | whole school | planned |
| **Pickup alerts** | Notify the guardian's phone when a child arrives or leaves. Whether this lives inside Grimmy or stands alone will be decided on site | parents | planned |

**Every finished feature gets one document under `docs/features/`**: why it was built that way,
and which decisions are actually wired into the code.

---

## Architecture

![Architecture overview](./docs/images/architecture.png)

<sub>Colored box · solid line = running today. Gray dashed = planned. Sources (.excalidraw) live in [docs/architecture/](./docs/architecture/).</sub>

```
Frontend   Next.js static export → Cloudflare Pages
Backend    FastAPI (Docker) → Fly.io
Storage    PostgreSQL (managed) · Cloudflare R2 (photos)
```

→ Layout · permissions · data model · deployment · why these choices: **[docs/architecture.md](./docs/architecture.md)**

---

## Operations — a record of responses, not of zero incidents

Incidents are treated as harvest, not accidents. Each one is written up in four parts:
**detect → diagnose → recover → what we learned**. Missed causes and wrong guesses stay in the record.
There were two incidents in the first week after launch, and **neither was found by an alert**:
one was reported by a person, the other spotted by chance in the logs. So alerting and daily backups
went in next, with a restore actually tested.

Then the alerts started doing their job. A race between two identical requests **was caught by an
alert before any user noticed**, fixed, and locked in with a regression test. The next day, a host
failure at the cloud provider **was caught by an alert three minutes in.** I saw it right away, on a
flight to Mexico, and could do nothing about it. The provider recovered on its own 17 minutes later.
Detecting a problem and being able to respond to it turned out to be two different things.

→ Full incident write-ups and the recording rules: **[docs/incidents.md](./docs/incidents.md)**

---

## What comes next

```
Grimmy (AI Agent)     LangGraph · RAG (pgvector) · tool calling. Per-role tools · 7 safeguards.
                      Stabilize with read-only tools first → write tools only after the safeguards stand.

Operations            Log retention · frontend tests · CI · SLOs.

Remote operation      Running now: three months from another country, maintenance only.
                      Alerts and backups were the precondition, and they went in before leaving.

Pickup alerts         Needs on-site testing, so it waits until I am back.
```

→ Confirmed vs. direction, kept apart: **[docs/roadmap.md](./docs/roadmap.md)**

---

## Stack

| Area | What |
|---|---|
| Frontend | Next.js (App Router · static export) · TypeScript · Tailwind · PWA + Web Push |
| Backend | FastAPI · SQLAlchemy · Alembic |
| DB | PostgreSQL (managed) · 16 tables · 19 migrations |
| Infra | Cloudflare Pages + R2 · Fly.io (Docker) · Cloudflare Registrar |
| Auth | Google · Kakao OAuth · HttpOnly cookie JWT · teacher approval for membership |
| Testing | pytest against real PostgreSQL · TypeScript · ESLint |
| Ops | Sentry (errors · uptime · cron) · GitHub Actions (daily backups) |
| Planned | LangGraph (AI agent) · pgvector (RAG) |

---

## Read more

| Document | What |
|---|---|
| [Architecture](./docs/architecture.md) | How the system fits together and why |
| [Feature docs](./docs/features/) | One document per feature — the decisions wired into the code |
| [Incidents](./docs/incidents.md) | Incident write-ups — grows with every incident |
| [Roadmap](./docs/roadmap.md) | What comes next — confirmed vs. direction |
| ADRs (architecture decision records) | *in preparation* |
| [TIL](https://github.com/vamosbada/TIL) | Learning notes — retrospectives for this project link here |
| Tech blog (Medium) | *in preparation* — design decisions and incidents as posts |

---

<sub>Planning · development · operations: [@vamosbada](https://github.com/vamosbada) ·
The service code is private. It handles real users' photos and conversations, so that repository
stays closed. This repository explains the system.</sub>
