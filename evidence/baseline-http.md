# Baseline HTTP Evidence

**Date:** 2026-10-10 (portal clock; real UTC was 2026-10-14)  
**Host:** rivermark-portal  
**Log sources:**
- nginx: `/var/log/nginx/access.log`
- app: `journalctl -u rivermark-portal.service`

---

## Action 1 — Failed login as analyst_yassine

- Kali UTC time: `2026-10-14 13:55:39`
- Portal clock time: `2026-10-10 21:07:43` *(4-day drift documented separately)*
- Request: `POST /login` with wrong password
- HTTP status: 200

**nginx line:**

10.20.30.10 - - [10/Oct/2026:21:07:43 +0000] "POST /login HTTP/1.1" 200 344 "-" "curl/8.20.0"


**App line:**


2026-10-10 21:07:43,238 login failure user=analyst_yassine from=10.20.30.10


- Keep one: **app line** — nginx knows a POST happened; only the app knows it was a *failed login for a specific user*.

---

## Action 2 — Successful login as analyst_yassine

- Kali UTC time: `2026-10-14 13:55:44`
- Portal clock time: `2026-10-10 21:07:48`
- Request: `POST /login` with correct password
- HTTP status: 302

**nginx line:**


10.20.30.10 - - [10/Oct/2026:21:07:48 +0000] "POST /login HTTP/1.1" 302 5 "-" "curl/8.20.0"


**App line:**

2026-10-10 21:07:48,489 login success user=analyst_yassine from=10.20.30.10


- Keep one: **app line** — same reason. nginx shows the status code change; only the app names the user.

---

## Action 3 — List drafts (with valid cookie)

- Kali UTC time: `2026-10-14 13:55:54`
- Portal clock time: `2026-10-10 21:07:58`
- Request: `GET /drafts` with valid session cookie
- HTTP status: 200

**nginx line:**


10.20.30.10 - - [10/Oct/2026:21:07:58 +0000] "GET /drafts HTTP/1.1" 200 278 "-" "curl/8.20.0"


- App line: **none** — the app does not log draft reads.
- Keep one: **nginx line only** — no app entry exists for this action.

---

## Action 4 — Open one draft (with valid cookie)

- Kali UTC time: `2026-10-14 13:56:00`
- Portal clock time: `2026-10-10 21:08:04`
- Request: `GET /drafts/draft-001-acme.txt` with valid cookie
- HTTP status: 200

**nginx line:**


10.20.30.10 - - [10/Oct/2026:21:08:04 +0000] "GET /drafts/draft-001-acme.txt HTTP/1.1" 200 268 "-" "curl/8.20.0"


- App line: **none**
- Keep one: **nginx line only**.

---

## Action 5 — /drafts without cookie

- Kali UTC time: `2026-10-14 13:56:07`
- Portal clock time: `2026-10-10 21:08:11`
- Request: `GET /drafts` with no cookie
- HTTP status: 302 (redirect to `/login`)

**nginx line:**


10.20.30.10 - - [10/Oct/2026:21:08:11 +0000] "GET /drafts HTTP/1.1" 302 5 "-" "curl/8.20.0"


- App line: **none**
- Keep one: **nginx line only** — the 302 tells you the request was redirected.

---

## Action 6 — Request a path that does not exist

- Kali UTC time: `2026-10-14 13:56:14`
- Portal clock time: `2026-10-10 21:08:18`
- Request: `GET /admin`
- HTTP status: 404

**nginx line:**


10.20.30.10 - - [10/Oct/2026:21:08:18 +0000] "GET /admin HTTP/1.1" 404 50 "-" "curl/8.20.0"


- App line: **none**
- Keep one: **nginx line only**.

---

## Can nginx alone answer "did Yassine log in?"

**Partially.**

Because the app returns **200 for failed login** and **302 for successful login**, nginx's access log **does distinguish success from failure** by status code. So nginx can answer:

*"A POST /login from 10.20.30.10 at 21:07:48 returned 302 — a successful login happened."*

**But nginx cannot answer:**

- **Which username** was submitted (the POST body contains the username, but nginx does not log request bodies).
- **Why** the login failed (wrong password? locked account? nonexistent user?).
- Any distinction between accounts.

To answer "did Yassine log in?", the app log is required — only the app knows the username and the result.

**Detection implication:**

- Rules for "login failure" must come from the **app log**.
- Rules for "POST to /login with unusual status code" can come from the **nginx log**.
- Rules for "unusual post-login activity" must come from the **nginx log**.

Neither log alone is sufficient. A complete SOC pipeline would forward both.

---

## Log scope summary

| Question | nginx | app |
|---|---|---|
| Every HTTP request | Yes | No |
| HTTP status codes | Yes | No |
| Source IP per request | Yes | Only for login |
| Username in login | No | Yes |
| Login result (success/failure) | Only via status code | Yes, explicit |
| Reads of specific drafts | Yes | No |
| User-Agent header | Yes | No |