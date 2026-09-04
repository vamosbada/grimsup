# Login · onboarding · permissions — who may see which child's photos

> Google proves **"whose account is this?"** It cannot know **"whose parent is this?"**, so the teacher
> approves that part. This document covers those two steps and the permission rules built on top of them.
>
> OAuth login itself is the same everywhere, so it is short here. **Decisions specific to this project**
> get the space. Why permissions matter more than usual in this project: **every upcoming feature behaves
> differently per role.** In the community only the teacher posts notices. Grimi gives parents, the
> teacher, and visitors different tools. All of that rests on the roles and states defined here.
> If this layer wobbles, everything above it wobbles.

---

## 1. Login — Google OAuth

```
/auth/google/login      validate the return path (next) against an allow-list → set oauth_state cookie → go to Google
                        state: 10 minutes · HttpOnly · SameSite=Lax
Google login screen     the user picks an account. We never see the password
/auth/google/callback   compare state (compare_digest) → create or update the user → JWT cookie → redirect to next
                        JWT: 7 days · HttpOnly · SameSite=Lax · Secure in production only
```

**Why a cookie.** If the JWT came back in the response body, the frontend would keep it in `localStorage`,
and one XSS would leak it. An HttpOnly cookie cannot be read by JavaScript, which blocks "steal the token
and reuse it on another device". (It does not block XSS itself. React's escaping does that, one layer deep,
and a rule that bans `dangerouslySetInnerHTML` protects that one layer.)

**Only things that never change go in the token.** `sub` holds one UUID. If role or status were in the
token, revoking an approval would not matter: the token already issued would keep passing for 7 days.
We pay one DB read per request to buy "revocation takes effect immediately".

**Why UUID instead of an integer primary key.** An integer is a sign-up sequence number; one value reveals
how many users there are. It also keeps the token independent of Google as a login provider: when another
provider is added, the auth layer does not change.

### 🔴 The return path (`next`) is an allow-list of 7 paths

A deny-list has to know every variant. Block `//evil.com` and `/\evil.com` remains; block that and
`%2f%2fevil.com` remains. *"We prefix our own domain, so it is safe"* is also wrong: with `@evil.com`,
everything before the `@` is read as a username and the user lands elsewhere. So only paths on the
allow-list (`/`, `/about`, `/gallery` …) pass. Anything else goes to the home page.

⚠️ `state` is not reused as `next`. `state` answers exactly one question, *"did this browser start this
flow?"* Where to return is a different question.

---

## 2. Onboarding — account states

Every new account **starts as `pending`.** A Google login by itself opens nothing.

```
guest / pending     logged in only. The onboarding screen appears
     │
     ├─ picks "parent"  →  parent / pending     submits child's name · contact. The teacher checks the roster
     │                        │
     │                        └─ teacher approves  →  parent / active     the gallery opens
     │
     └─ picks "visitor" →  guest / active      school page only. No child data is stored
```

- **Only the teacher's admin screen moves an account to `active`.** Onboarding code writes `pending` at most.
- If someone submits as a parent and then switches to visitor, **the child's name and phone number are
  deleted** so nothing stays on an outsider's account (minimum retention).
- Re-submitting overwrites. The teacher always sees the latest submission.
- Submitting again while already `parent/active` returns **409**, so an approved account cannot slide
  back to `pending` by accident.
- No onboarding in any role without consent. The email and name from Google are already stored, so
  consent to that storage comes first.

### Approval is one transaction

When the teacher approves, **register the student + link the guardian + change the status + create the
gallery room** commit together. Split them and you get *"an approved parent with no room"*, an account
that can never open the gallery.

**Duplicates are blocked by DB constraints, not by application code.** We do not check "is this student
already linked?" before inserting; the unique constraint does it. A check followed by an insert
(check-then-act) can be interleaved by another request and still break. Rooms work the same way:
one room per guardian is a unique constraint, so a second room cannot exist.

### The `else` of an enum branch is a 500

If `role` is neither `parent` nor `guest`, we do not quietly move on. **We raise a 500.** Quietly moving on
means an account could become `active` without approval, and that is worse than an error. When a new role
arrives (the community's `student`), this spot is guaranteed to blow up.

---

## 3. Permission rules

**The URL does not say whose data it is.** The first rule we fixed.

```
❌ GET /api/students/42/photos     ← change 42 to 43? Now every endpoint needs its own check
✅ GET /me/rooms                    ← the cookie says who you are. There is nothing in the URL to change
```

Instead of guarding the attack surface with checks, we chose **not to build the surface at all.**
Forgetting a permission check on a new endpoint becomes structurally hard.

Smaller rules that go with it:

- **Internal integer ids never leave the fence.** Only UUIDs go outside.
- **The admin screen returns 404, not 403, to anonymous users.** A 403 says "there is something here".
- **Responses list their fields explicitly.** Returning an ORM object as-is lets internal fields
  (`provider_id` and friends) leak quietly.
- **Validation lives in schemas.** Data that reaches a handler is already safe: phone numbers normalized,
  roles limited with `Literal`.
- **No raw SQL.** That means building SQL statements as strings and sending them to the database.
  The ORM sends the statement and the values separately, so user input can never be read as a command.
  String SQL bypasses that separation, and the defense at that spot is gone the first time it is used.
  The only way to keep the rule was to allow no exceptions.
- **API docs are closed in production** (`/docs` · `/redoc` · `/openapi.json`). This does not fix a
  vulnerability. It **withholds the map**: one file lists the names, arguments, and response shapes of
  20 endpoints. Turning off `/docs` alone closes nothing. The three are one set.

---

## 4. When a second login provider arrives

Today there is only Google, so `users` carries `provider` and `provider_id` directly. When a second
provider (Kakao) arrives, it must become **one person = one account, several login methods**, so login
methods move to their own table. It is not done now because there is nothing to validate yet. It is done
before the community because **schema changes are cheap while the tables are small.**
