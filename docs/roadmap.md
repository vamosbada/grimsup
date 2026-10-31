# Roadmap — what comes next, and what has to be in place first

> **Legend**: ✅ confirmed (order and scope are set) · 🧭 direction (decided to do it; scope waits on real-user answers)
> The two are never mixed. A feature marked as direction may not be built as written.

What is live today: auth (Google · Kakao) · gallery · community · school page · alerts and backups.
Next is the AI agent, Grimmy. The operating foundation was filled in **before** the community went
wide, because two incidents set that order.

---

## 0. Observability ✅ done first · the rest continues

Incidents #1 and #2 were both found **by a person or by chance, not by an alert.** Records existed, but
the records did not call me. So this went in before more features were added.

**Done**

```
Alerting        Error alerts and an uptime check (/health every minute · alert after 3 failures in a row),
                by email. Live since 10-14. Measured: ~3 min from outage to alert.
                #3 and #4 were both caught this way
Health check    The uptime check is what calls /health now. It deliberately does not touch the DB —
                a DB query every minute would keep the managed DB from ever sleeping
Own backups     Daily pg_dump → R2, kept 30 days · a restore was actually tested once.
                A cron monitor alerts if the backup fails **or never runs at all** —
                a failed job leaves logs, a job that never started leaves nothing
```

**Next**

```
Request-rate alerts  Error alerts alone would still miss #2. A 403 is a correct response, logged at INFO
Log retention        Only what the host keeps. The evidence for #2 did not survive one day (measured)
Frontend tests       Zero. #2 was a frontend state-transition bug, and this was the only layer that could catch it
SLO definition       Without "how often should I look?" there is no basis for alert thresholds
CI · deploy          The backend is deployed by running fly deploy from a laptop. Checks are run by hand
Phone-only response  #3: the alert arrived in flight and there was no lever to pull from a phone
```

**Remote operation ✅ running now.** For three months the system runs from another country **on
maintenance alone.** That needed a setup where "the system calls me even when I am not looking", and
alerts and backups went in before leaving. #3 showed the next layer: being called is not the same as
being able to act.

---

## 1. Community ✅ done — the real users' first request

The goal was concrete: **turn off the external community app.** Notices lived there, so parents had to
check two places. The real user said *"notices are what matter"*, and the work was split in that order.

```
① Second login provider   ✅ Kakao, added first. The account structure was split
                          (one person = one account, several login methods) while tables were still small.
                          Linking a second login to an existing account came after
② Notices + comments      ✅ The external app was turned off for notices
③ Kids' posts             ✅ Kids and parents post drawings and stories · private posts on the talk board
                          Free-form input grew here, so XSS was handled before it opened
```

---

## 2. Grimmy — AI Agent 🧭

An agent built on **LangGraph**, running in the same image as FastAPI. It answers parents' questions from
school information (**RAG** on PostgreSQL's pgvector) and handles **write actions** like absence and
make-up requests through **tool calling** into our own API. Reading (answers) and writing (requests)
**do not open at the same time.** Read-only tools stabilize first. Write tools are added only after the
safeguards below are in place.

**Scope is not final.** We asked the real user "what would you want an agent to do for you?" and got
**36 first-round requests** (teacher's work 25 · parents 6 · visitors 5). Not all of them will be built.
We pick from this list. The main branches:

```
Teacher's work   recurring messages (attendance · make-ups · unpaid fees) · a daily "today's tasks" briefing ·
                 a student history summary before a consultation · draft notices and plans · weekly/monthly
                 reports · first response to new inquiries
Parents          absence / make-up information and requests · payment history · a nudge to those away for a while
Visitors         one-day class booking with a reminder the day before · post-visit enrollment info · first-visit survey
```

Three criteria for picking: ① daily repetitive work first ② reading (summaries · information) before
writing (sending · processing requests) ③ **anything involving money is code, not the agent.**

🔴 One thing already settled in discussion: **deducting class credits and recording attendance are done
by code, not by the LLM.** Where money is involved, "mostly right" is not acceptable. The agent *calls*
that code.

### Before write actions touch the production DB — 7 safeguards

> Listed in 2026-07. To be confirmed again when work starts. For now this is direction.

| # | Safeguard | Why |
|---|---|---|
| 1 | **Permission boundary** | The agent's DB access never goes beyond "the requesting parent's own children". It passes the same auth and permission checks as every other API. No back door because it is an agent |
| 2 | **Audit + rollback** | Writes such as requests record who · when · what, and can be undone. Soft delete already exists; this builds on it |
| 3 | **Transactions** | Multi-step actions like "approve + register student + link guardian" are atomic. No half-done state is left behind |
| 4 | **Prompt‑injection defense** | Parents' input goes into the prompt. The tools the agent may call are an allow-list. Whatever the input says, nothing outside the list can run |
| 5 | **Observability** | Log and trace LLM calls and tool calls (LLM observability). Incident response needs "why did it answer that?" to be answerable later |
| 6 | **PII minimization** | Student and parent data leaves for an external LLM API. Put as few fields in the prompt as possible |
| 7 | **Cost and rate limits** | Control LLM cost and call frequency. One user cannot spend the whole budget |

One reason PostgreSQL was chosen sits here: RAG's vector search (pgvector) can live in the same DB.
It is not in use yet.

---

## 3. Pickup alerts 🧭

Notify the guardian when a child arrives or leaves. Planned for later.

---

## 4. Real-user requests — small, but first

Separate from the feature stages, these came straight from the real user. Often they go before the big
features.

```
Photo watermark     Requested three times. It goes into the upload path — a separate tool would mean
                    stamping every photo by hand
                    + let the teacher swap the school page photos without a developer (precondition for remote operation)
Admin roster        ✅ done. The admin screen used to show only pending approvals. What was wanted was a
                    **roster** of everyone who signed up, not a count
```

---

## Deliberately not built

```
Payments            Building it while adoption is undecided means throwing it all away
Preview deploys     One person works here; there is nobody to show previews to. An unlocked URL is only a leak risk
Kubernetes          Nothing here demands it
```

The yardstick is not *"does this traffic need it?"* but **"is this piece missing from the production
cycle?"** So even with few users, **audit log · privacy policy · incident records · alerting · backups**
go in, and **tools that only scale demands** stay out.
