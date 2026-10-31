# Community — four boards on one permission table

> Replaced the external app the school used for notices. Most of it is an ordinary board.
> Only the decisions specific to this project are written down here.

---

## 1. Four boards

| Board | Read | Write | Comment |
|---|---|---|---|
| Notices | parents · teacher | teacher | parents · teacher |
| School news | everyone logged in | teacher | — |
| Kids' drawings | students · parents · teacher | students · parents · teacher | students · parents · teacher |
| Talk (private posts allowed) | students · parents · teacher | students · parents · teacher | students · parents · teacher |

All of it requires an approved (`active`) account. The tables are **allow-lists**: a board or role that
is not written in them is closed by default.

## 2. One function decides whether a post can be seen

Every endpoint that touches a single post — open, mark as read, comments, reactions, delete — goes
through the same check. The endpoints are independent, so blocking only one of them would leave the
others open.

A board you cannot read returns **403** (the board is in the menu anyway). A post you cannot see returns
**404**, so its existence is not confirmed.

**Private posts** on the talk board are visible only to the author and the teacher. In lists, others see
a 🔒 line; the server does not send the content.

## 3. Who gets notified

```
Notices · news      everyone who can read the board
Kids' boards        the teacher only — one drawing ringing every family's phone would get
                    notifications turned off, and then the notices would be missed too
Comments            the teacher · the post author · the author of the comment being replied to (once each)
Reactions           the post author, only the first time
```

The kids' drawings feed is grouped by month in **Korea time** — the database stores UTC, so a drawing
posted on the morning of the 1st would otherwise land in the previous month.
