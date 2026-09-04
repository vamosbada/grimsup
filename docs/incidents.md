# Incident log — Grimsup

> **Why this is a separate document** (created 2026-08-29 · format revised 2026-09-03)
>
> Incident notes buried inside a daily work log disappear into a pile of dates within days.
> But what has value in this project is **the record of responses, not a record of zero incidents.**
> So it gets its own place.
>
> **Newest first.** Each entry has four parts: **detect → diagnose → recover → what we learned.**
> This order because *"how did we notice, and how did we narrow it down"* is more useful next time than
> *"what broke"*. The cause fits in one line. The way we detected it is different every time.

---

## 📋 At a glance

> One row per incident. If the "detected by" column never says "alert", that is this project's biggest
> open gap. "Noticed → over" is measured from the user's point of view, not from deploy time.

| # | Date | Grade | Detected by | Noticed → over | Open items |
|---|------|-------|-------------|----------------|------------|
| 2 | 09-01 | 🟡 | chance | unknown (old JS kept running in open tabs) | alerting · log retention · frontend tests |
| 1 | 08-29 | 🔴 | a person | ~20 min | alerting · log retention |

**Grading — one question: "could the user do what they came to do?"**

```
🔴 Blocked        They could not, and there was no workaround.           #1: sign-up impossible
                  **Data loss or exposure is 🔴 regardless of user count or workaround** (we handle minors' photos)
🟡 Degraded/cost  They could (or never could in the first place).        #2: screen fine, server got 5 req/s
                  But: slow, needed a workaround, server cost, polluted logs
🟢 No impact      Nothing changed for users. Caught before it blew up    → template B, four lines
```

- **Grade ≠ severity.** #2 was a bug nobody could have reported, yet it is 🟡. The grade only asks
  **what happened to users**, not how scary the bug was. How hard it was to find belongs in "detect".
- **User count does not change the grade.** One person unable to sign up is 🔴. Count goes in "impact".

---

## 🔴 Rules for writing an entry

```
① No personal data     Never write parents' or students' names, emails, or phone numbers.
                        Use "one user" or "parent A". This document is public

② No polishing         Write what was **missed**, not "we handled it well".
                        The value of this document is the missed causes and the wrong guesses

③ Same day             While memory is fresh. **If you cannot write, at least save the logs**
                        (raw copies stay in the private repository).
                        (Revised 9/3: in #2 the logs were gone within a day and the "what we saw" box
                        stayed empty. The write-up can wait a day. The raw logs cannot)
                        ⚠️ Raw logs are never published. Tracebacks print INSERT parameters, which contain personal data

④ Stay in the template  Do not add or remove the template's headings. Do not delete a box you cannot fill;
                        write "unknown" — an empty box is itself a record.
                        If the template does not fit, **change the template and leave one line here on why**.
                        (#1 and #2 keep the pre-revision format. Records are not rewritten)

⑤ 🟢 stays short       No-impact entries and near misses use the "no impact" template (B), one block.
                        If writing is expensive, rule ③ breaks again

⑥ Markers              🔴 marks **urgency** only.
                        In the at-a-glance table, "open items" are never deleted when fixed; add ✅ + date
                        (e.g. "✅ alerting 9/20"). That one cell is the evidence that we learned and fixed it
```

---

## 📝 Template A — incident (🔴 / 🟡)

<!-- Copy from "## 🚨 Incident #N" through the end of "### What we learned" and paste it at the top of
     the "Log" section. Keep the headings, fill in the parentheses. Add one row to the at-a-glance table -->

## 🚨 Incident #N — one-line title (YYYY-MM-DD)

> (Why this one is worth recording — one line. Not "what broke")

```
Impact   (user count · what they could not do · server/cost. No identities)
Grade    🔴 blocked / 🟡 degraded — criteria under the at-a-glance table
         was there a workaround · could the user have known
Timeline occurred → noticed → fix commit (hash) → deployed → impact over
         (write "unknown" if unknown. Deploy time is not the end time)
```

### Detect — how we noticed

**By: alert / a person / chance**   ← one of the three, first line. If not "alert", that is the day's biggest lesson

(what we were doing · what we saw)

### Diagnose — how we narrowed it to the cause

**What we saw**: (commands and raw log lines. If gone: "unknown — reason")

**Order of elimination**:
1. (candidate → why ruled out → next candidate)

**Wrong guesses**: ("none" if none. Otherwise, why it looked plausible)

### Recover — what changed

```
Changed        (code · config · data — files and the gist)
Temporary/root (if temporary, the root fix goes to the to-do list — write only "temporary" here)
Deployed       (fly deploy / main push / migration yes or no)
Verified       (what showed it was fixed — zero errors with zero traffic is not evidence)
Regression test (path / none — why)
Rollback       (revert hash / downgrade — if unsafe, why)
```

### What we learned — why we did not catch it earlier

**Why automated checks passed**: (tests · types · build — which layer had the hole)

**Why docs or rules did not warn us**: ("n/a" if none)

**Next candidate of the same kind**: (where the same shape could blow up next. "none" if none)

---

## 📝 Template B — no impact (🟢)

<!-- No user impact · caught before it blew up. One block, done.
     Numbering shares **one sequence** with incidents (if #3 was 🔴, the next 🟢 is #4).
     It goes in the at-a-glance table too — the "caught before impact" ratio should be visible in one table -->

## 🟢 No impact #N — one-line title (YYYY-MM-DD)

```
Detected  alert / a person / chance — (while doing what)
Cause     (one line)
Action    (one line · commit hash)
```

---

# Log

<!-- Newest first. #2 and #1 keep the pre-revision format -->

## 🚨 Incident #2 — an outage nobody could report, caught in the logs (2026-09-01, around 07:00)

> **The screen looked fine.** When a user without gallery access (a visitor, or a parent waiting for
> approval) opened the gallery, the info card rendered correctly. Behind it, the browser was hitting the
> server with `/rooms` (403) → `/me` (200) **every 190 ms.** The user felt nothing, so this was an outage
> that was **structurally impossible to report.**

```
Impact   one user (one account, confirmed) · nothing the user could not do — they had no gallery access to begin with
         cost was on the server: one tab ≈ 5 request pairs per second · logs flooded with 403s, hiding everything else
Timeline occurred: unknown · noticed 9/1 ~07:00 (from memory) · fix commit 9/2 13:27 · deployed 9/3 11:46 (main push)
Grade    🟡 degraded/cost · not blocked. But **a kind the affected user cannot notice or report**
```

> ⚠️ **This entry was written on 9/3, breaking rule ③ (same day).** The commands, raw logs, and wrong
> guesses from 9/1 are gone. That is why the "detect" section below is thin.

### Detect — **by chance, not by an alert**

Around 07:00 on 9/1 I opened the logs meaning to do a routine check and **just saw it.** The same
account was logging `/rooms 403` and `/me 200` in an endless loop at 190 ms intervals. No alert, no
dashboard, no user report.

🔴 **The same lesson as incident #1, repeated three days later.** On 8/29 I wrote down *"no error
alerting"* as a to-do and postponed it. That hole bit again. In #1 at least a person told us. This time
**there was nobody who could have told us**: the screen was fine. Had I not opened the logs that
morning, it would still be running.

⚠️ And **an "error alert" alone would not have caught it.** A 403 is not an error. It is the correct
response, logged at INFO. Catching this takes anomaly detection like **"request rate from the same
user"**. If alerting only watches error levels, this kind slips through again.

### Diagnose — the backend was innocent; two pieces of frontend code formed a loop

The 403 was right. The gallery service checks `status == "active" and role in ("director", "parent")`
as an allow-list and returns 403 otherwise, exactly as designed. The problem was two pieces of
normal frontend code (the gallery page component) combining:

```
①  The gallery-eligibility value (canUseGallery) was computed **below** an early return (`if (!me) return null`)
    → hooks have to sit above it, so the polling effect could never see that value
    → it kept sending /rooms for users who were not eligible
②  The 403 handler assumed "permissions may have changed" and called refresh() to reload `me` (correct — handles removal)
    → that `me` is a dependency of the polling effect, so the effect re-ran → /rooms again → 403 again
```

Each piece had a reason to exist. ② without ① is fine. ① without ② means one 403 per 3-second poll and
nothing more. **Together they formed a loop with no exit condition.**

A bonus finding: the frontend's eligibility expression was `director || (parent && active)`, which
**differed from the backend**: director skipped the status check. A `director/pending` account (a case
that already exists in the backend tests) would pass this guard and fall into the same loop.

### Recover — one frontend file, four spots

```
①  Moved the canUseGallery computation above the hooks so the polling effect sees the same value
②  Added `if (!canUseGallery) return;` to the polling effect — no request at all when not eligible
③  Rewrote the eligibility expression in the **same shape** as the backend: active && (director || parent)
④  Removed the duplicate computation in the render branch below — written once
Deployed  frontend only, no fly deploy. main push → Cloudflare Pages build
```

Matching the *shape* in ③ was not taste. **When the same rule lives in two places, you have to be able
to put them side by side and compare by eye** to notice when they drift.

**Verified**: 9/3 11:45, locally, opened the gallery four times as a guest/pending account → logs show
`/me 200` three times and `/rooms` **zero** times. Before the fix, this produced five pairs per second.
The 403 count in production is **weak evidence**: with no traffic, zero means "nobody came", not "fixed".
And because the frontend is a static export, users with an open tab keep running the old JavaScript
until they reload. So production is watched for a few days to see **whether 403s trend down**, as a
secondary signal.

**Rollback**: `git revert` of the fix commit. One frontend file, no DB or migration involved.

### 🔴 What we learned — every automated check passed

```
pytest, 153 tests   all green — the backend really was correct. Backend tests cannot catch this
frontend tests      zero. This loop was a frontend state-transition bug, and this was the only layer that could catch it
```

> **One side being correct does not prevent an outage.** The failure was the **combination** of a server
> rule and a client state transition, and a combination is caught by neither layer's tests.

🔴 **The evidence vanished within a day.** On 9/2 I went back for the 9/1 logs and they were gone.
I knew the host keeps only recent logs, but it was *"probably"*. That day it became **measured.**
"Log retention" went from a guess to an item with evidence.

---

## 🚨 Incident #1 — a long Google profile-picture URL blocked sign-up (2026-08-29 20:05)

> **The first real-user outage, five minutes after launch.**

```
Impact   one parent · sign-up failed completely (no account was created)
Timeline occurred 20:05 · noticed 20:1x · deployed 20:2x
Grade    🔴 fully blocked · no workaround — there was nothing the user could do
         (retrying fails at the same spot for the same reason)
```

### Detect — **by a person, not by an alert**

The teacher relayed it: *"someone says they got a server internal error"*.

🔴 **This is the day's biggest lesson.** The traceback was sitting in the logs at 20:05, already
structured as JSON, and **nobody was looking at it.** Observability is not "is it recorded?" but
**"does the record call me?"**, and we had only the first half.

⚠️ Why this hurts more: the managed DB's free plan has a **6-hour** restore window. In our setup,
**slow detection is itself data loss.** "No alerting" was never a convenience problem. It was a
recoverability problem.

### Diagnose — one log line ended it

```bash
flyctl logs --no-tail | grep -A2 "sqlalchemy.exc"
```
```
sqlalchemy.exc.DataError: (psycopg2.errors.StringDataRightTruncation)
value too long for type character varying(500)
[SQL: INSERT INTO users (...)]
```

Order of elimination:

1. `users` has **two** `varchar(500)` columns: `picture` and `parent_note`
2. In the INSERT parameters, `parent_note` was `None` → **length 0, cannot be the culprit**
3. Counted the actual length of `picture` → **1,132 characters** (2.3× the limit)

```
most users   .../a/ACg8oc…=s96-c        ≈ 100 chars
this user    .../a-/ALV-Uj…=s96-c       1,132 chars
                  ↑ URLs starting with a-/ALV- are the long form
```

🔴 **Not bad luck. An account property.** Google returns one of two URL forms depending on the account.
This was not "one person happened to hit it" but **"everyone with this form is blocked."** Treated as a
one-off, it would have quietly turned away several people over the following days.

⚠️ **Turning on `LOG_JSON=true` earlier that afternoon paid off here.** A routine check that day found
that the production environment variable was missing and logs were in human format. Six hours later
that log format cut the first outage.

### Recover — two layers

```
① Column   users.picture  varchar(500) → text
           migration (written by hand, not autogenerated)
② Guard    MAX_PICTURE_LEN = 2000 + _safe_picture()
           at **both** places: new sign-up and re-login
③ Tests    pytest 151 → 153
Deployed   fly deploy (release_command runs alembic upgrade head)
```

🔴 **② is the real fix.** With ① alone, *"so how many characters next time?"* stays open. The essence
was not length. It was that **a decorative field (the profile picture) blocked the core action (sign-up)**:
a parent who came to see their child's drawing was turned away at the door by a profile-picture URL.
With the guard, the worst case is *"no profile picture"*, which is not in the same league as *"cannot
sign up"*.

🔴 **Why the re-login path too**: guarding only sign-up means an existing parent who changes their
profile picture would crash at the same spot **on every login.** That is someone who was using the
gallery getting kicked out, which is worse than a failed sign-up.

### 🔴 What we learned — **why 153 tests did not catch it**

The fake Google (`_FakeGoogle`) **always returned short URLs.**

> **When you fake an external system with only its normal values, the fake becomes the blind spot.**

All of our code was tested. What was not tested was **our assumption about what Google can return.**
Coverage cannot show that hole.

The regression test was split **in two**:

```
① A long URL still completes sign-up (the picture is dropped)
② The actual 1,132-character URL is stored as-is
```

With ① alone, *"drop everything to None"* would pass too, and **the column fix would go unverified.**
The guard and the schema mask each other, so each needs its own test.
