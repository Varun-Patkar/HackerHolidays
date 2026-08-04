# Day 7 — Do Not Disturb

> Part of the [Hacker Holidays 2026 findings log](../FINDINGS.md).

- **Status:** ✅ Completed (2026-08-02)
- **Target:** `http://<MACHINE_IP>` — sessions during play: `10.146.140.136`, then `10.146.155.176` (Node.js/Express app on 80, OpenSSH 9.6 on 22). Attacker IP (tun0): `192.168.139.84`.
- **Category:** Web → Boot2Root (Medium, 90 pts)
- **Objective:** Find the user flag and the root flag.
- **Story hook:** _"You have access you were never given, and so does he"_ + _"a session goes warm on a sunbed, and a stranger sits down in it"_ + _"whoever's already inside has been moving for far longer than you have"_ + _"Byte Lotus never forgets · Stay Noticed™"_. Each phrase maps to a real step: the NoSQL/session bypass (the "warm session"), the already-running `pipelinesvc` inspector process ("someone already inside"), and raw-disk read as the finish.
- **Attack path (summary):**
  1. **Recon:** `nmap` → 22 (OpenSSH 9.6p1, Ubuntu 24.04) + 80 (Node.js/Express, `X-Powered-By: Express`). Login page "Byte Lotus — Poolside" posts `username`/`password` to `/login`. `gobuster` found `/logout` (302) and `/staff` (403 "Staff access only").
  2. **NoSQL auth bypass (foothold to app):** the `/login` endpoint also accepts a **JSON** body and returns JSON errors → operator injection. `{"username":"attendant","password":{"$ne":"x"}}` returned `200 {"ok":true,"role":"staff"}` and set a signed `connect.sid` session cookie. (`attendant` = staff, `guest` = guest.)
  3. **`/staff` → EJS SSTI:** with the staff session, `/staff` rendered the **Cabana Desk** — a booking-confirmation **EJS template** editor posting to `/staff/preview`. EJS renders server-side → SSTI. `<%= 7*7 %>` → `49` confirmed it.
  4. **RCE + user flag:** `require` isn't global (the page context), so used `process.mainModule.require('child_process')` / `execSync` for quick reads and **`exec` (async)** for the reverse shell. Landed a shell as **`poolside`** (uid 996, `/opt/poolside`). User flag in `/opt/poolside`.
  5. **Enumeration → open Node inspector:** `pspy64` revealed a second Node service run by uid 995: `/usr/bin/node --inspect=127.0.0.1:9229 processor.js` (`/opt/pipelinesvc/telemetry/processor.js`). The **debug inspector was listening on 127.0.0.1:9229** = code exec as `pipelinesvc` for any local user.
  6. **Node inspector RCE → `pipelinesvc`:** connected to the inspector's WebSocket (Chrome DevTools Protocol) with a stdlib-only Python client and ran `Runtime.evaluate`. `require` was undefined (ESM context), so used **`process.getBuiltinModule('child_process').execSync(...)`** (Node 22+) → code exec as **`pipelinesvc`** (uid 995). Key detail in the output: `groups=995(pipelinesvc),6(disk)`.
  7. **Privesc via `disk` group → root flag:** `pipelinesvc` is in the **`disk`** group = raw read/write to the root block device. Root partition = `/dev/nvme0n1p1`. Read the flag straight off the raw filesystem, bypassing all permissions:
     `debugfs -R "cat /root/root.txt" /dev/nvme0n1p1`.
- **User flag:** <details><summary>click to reveal</summary><code>THM{w4rm_s3ss10n_h1j4ck3d}</code></details>
- **Root flag:** <details><summary>click to reveal</summary><code>THM{r4w_d1sk_4cc3ss_w4s_t00_much}</code></details>
- **Lessons:**
  - Never pass user input straight into a Mongo/NeDB query object — cast credential fields to strings so `{"$ne":...}`/`{"$gt":...}` operators can't be injected (NoSQL auth bypass).
  - Never render user-controlled templates with a server-side engine (`ejs.render` on attacker input = RCE). Treat the message body as data, not a template.
  - Never expose the Node **`--inspect`** debugger in production — even bound to `127.0.0.1` it is full RCE as the service user for any local foothold. Remove `--inspect`/`--inspect-brk` from prod launch args.
  - The **`disk`** group is effectively root — it grants raw block-device access (`debugfs`/`dd`) to read/write any file including `/root` and `/etc/shadow`. Never add a service account to `disk`.
  - On a single-threaded Node target, use `exec()` (async) for shells — `execSync()` on a long-running reverse shell freezes the entire event loop and hangs the whole app.

## My exploration log (what actually happened, including dead ends)

1. **Recon.** `nmap -sC -sV -p22,80` → OpenSSH 9.6p1 (Ubuntu 24.04) + `Node.js (Express middleware)`, title "Byte Lotus — Poolside". `whatweb` confirmed `X-Powered-By: Express`, `PasswordField[password]`.
2. **Login page.** Static HTML form → `POST /login` with `username`/`password`. Notably the page CSS styled a `textarea` and a `pre` block that **weren't on the login page** — a tell that an authenticated page with a text box + rendered-output area existed.
3. **Dir enum.** `gobuster` (raft-medium) → `/logout` (302) and `/staff` (403, body "Staff access only"). Everything else 404.
4. **Dead ends:**
   - Password guessing on `attendant` (`attendant`, `password`, `poolside`, `bytelotus`, …) → all `401`. **By design** — creds weren't the way in (`attendant`'s real password was `crypto.randomBytes(18)` per the later source read).
   - No session cookie was issued by `/` or by a failed login, so there was nothing to tamper yet.
   - Source/secret probing (`.env`, `package.json`, `server.js`, `.git/HEAD`, `robots.txt`, …) → all 404.
   - `X-Forwarded-For: 127.0.0.1` on `/staff` → still 403.
5. **The turn:** sending the login as `Content-Type: application/json` returned `{"error":"Invalid credentials"}` (JSON) — proving a JSON API and opening **NoSQL operator injection**. `{"password":{"$ne":"x"}}` logged in as `role: staff`.
6. **UI method:** since a plain HTML form can't send a `{"$ne":...}` object, logged in from the browser DevTools console with `fetch('/login',{method:'POST',headers:{'Content-Type':'application/json'},body:JSON.stringify({username:'attendant',password:{$ne:'x'}})})` — the server set `connect.sid`, then `/staff` opened normally and the EJS textarea was usable directly.
7. **SSTI foothold:** `<%= 7*7 %>` → `49`; `execSync('id')` → shell as `poolside`.
8. **Mistake / recovery:** first reverse shell used **`execSync`**, which blocked Node's single thread and **froze the whole web app**; `Ctrl-C` killed the listener. Fix = terminate/redeploy the box and use **`exec` (async)** for the shell. (Lesson recorded above.)
9. **Source read (`/opt/poolside/app.js`):** session secret `byte-lotus-poolside`, seed users `guest:sunshine` (guest) and `attendant:<random hex>` (staff), backend = **NeDB** (`findOneAsync`), not Mongo — confirming why the operator injection worked. `sunshine` was **not** reused for any system user (tested `su` for root/ubuntu/pipelinesvc → all failed).
10. **Privesc recon:** real-shell users = `root`, `ubuntu`, plus service accounts `poolside` (996) and `pipelinesvc` (995). `pspy64` exposed the `--inspect=127.0.0.1:9229 processor.js` service as uid 995 → the intended vector.
11. **Inspector RCE gotcha:** `require is not defined` (ESM) — switched to `process.getBuiltinModule('child_process')`; RCE as `pipelinesvc` revealed the `disk` group membership.
12. **Root:** `debugfs -R "cat /root/root.txt" /dev/nvme0n1p1` → root flag. (`/dev/root` → `nvme0n1p1` from `df`/`lsblk`.)

## Key host facts

| Item | Value |
| ---- | ----- |
| App (foothold) | Express + **NeDB** at `/opt/poolside/app.js`, run as `poolside` (uid 996) |
| Auth-bypass sink | `db.findOneAsync({ username, password })` with unsanitised JSON → `{"$ne":"x"}` |
| RCE sink | EJS template rendered from user input at `POST /staff/preview` |
| Privesc service | `node --inspect=127.0.0.1:9229 /opt/pipelinesvc/telemetry/processor.js` as `pipelinesvc` (uid 995) |
| Root primitive | `pipelinesvc` ∈ `disk` group → `debugfs` raw read of `/dev/nvme0n1p1` |
| Session secret | `byte-lotus-poolside` (in `app.js`) |
