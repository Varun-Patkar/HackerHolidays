# Day 8 — Towel on the Sunbed

> Part of the [Hacker Holidays 2026 findings log](../FINDINGS.md).

- **Status:** ✅ Completed (2026-08-03)
- **Target:** `http://<MACHINE_IP>:3000` — session during play: `10.144.149.43:3000` (Node.js/Express app, `X-Powered-By: Express`).
- **Category:** Web (Medium, 90 pts)
- **App:** **Ponzi Portfolio** — a crypto "wellness rewards" dashboard. Register an account → start at **50 PONZI** → **Claim Reward** grants **+50 PONZI every 24 hours** → reach **150 PONZI** to unlock the **Whale Vault** (holds the flag).
- **Objective:** Get past the Whale Vault's 150-PONZI gate and retrieve the flag from the vault.
- **Story hook:** _"He set his towel down, claimed his daily reward... came back to find the sunbed had been 'claimed' three times over"_ + _"The app disagrees, politely, once every 24 hours. Somewhere between his request and the server's clock, there's a gap wide enough to walk a whale through."_ + @0xMia's story: _"bro really thinks the clock is the only thing checking him."_ Every line points at a **check-then-act race condition (TOCTOU)** on the claim endpoint.
- **Vulnerability:** **Race condition / TOCTOU double-spend** on `POST /claim`. The server **checks** the 24h cooldown, then **writes** the new `last_claim` timestamp with a window in between. Firing many claims concurrently lets several pass the cooldown check *before any of them writes the timestamp* — each credits +50, so one eligible moment yields multiple rewards.
- **Attack path (summary):**
  1. **Recon / map the app in Burp:** registered a guest account (`POST /auth/register`), noted the endpoints — `POST /claim` (the reward, returns JSON), `GET /dashboard/api/me` (balance), auth via signed `connect.sid` session cookie. Dashboard showed **50 / 150 PONZI** and a greyed-out **Open Vault** button with a 24h countdown.
  2. **Confirmed the guard is time-only:** clicking Claim once set `next claim in: 23:59:xx`; further claims returned `429 {"error":"Reward already claimed. Please wait before claiming again.","secondsRemaining":85969}`. The **only** check is the cooldown timer → race candidate.
  3. **Set up the race in Turbo Intruder:** sent `POST /claim` to Turbo Intruder. The **single-packet attack failed** here — the target is **HTTP/1.1 over plain http** (not HTTP/2) and the request has **`Content-Length: 0`** (no body byte to withhold for last-byte sync). Switched to a **plain threaded concurrency blast** (`Engine.THREADED`, `concurrentConnections=30`, 30 queued requests, no gate).
  4. **Dead end — racing an already-claimed account:** first burst on the original account returned **all 429** ("already claimed", `secondsRemaining ~85969` ≈ 23.9h). Key realization: the race window **only exists on the first eligible claim** — once the cooldown is set, every request correctly loses. The account had already burned its window by claiming once in the browser.
  5. **Winning move — race a fresh account's first claim:** registered a **brand-new account** (null `last_claim` = immediately eligible), swapped its new `connect.sid` cookie into the Turbo Intruder request, and — **without clicking Claim first** — fired the 30-request threaded burst straight away. Multiple requests hit the eligible cooldown check simultaneously → several returned **200** success (+50 each).
  6. **Vault:** balance blew past the gate to **1,500 PONZI** (WHALE tier). The **Open Vault** button activated and revealed the flag.
- **Flag:** <details><summary>click to reveal</summary><code>THM{t0w3l_0n_th3_sunb3d_d0ubl3_sp3nt}</code></details>
- **Lessons:**
  - Guard state-changing, rate-limited actions with an **atomic** operation, not check-then-write. Enforce the cooldown at the database with a **conditional/atomic update** (e.g. `UPDATE ... WHERE last_claim < now()-24h` and check rows-affected, a unique constraint, `SELECT ... FOR UPDATE`, or an atomic compare-and-set) so concurrent requests can't all pass the check.
  - A **time/cooldown check alone is not a concurrency control** — "once every 24 hours" enforced by reading-then-writing a timestamp is trivially defeated by parallel requests (classic TOCTOU double-spend).
  - The race window is at the **first eligible claim** — a fresh account (or the instant a cooldown expires), not on an account already on cooldown.
  - **Attacker tooling note:** the single-packet attack needs HTTP/2 and/or a request body to withhold; for HTTP/1.1 body-less endpoints, a threaded concurrency blast is the reliable way to trigger the race.

## My exploration log (what actually happened, including dead ends)

1. **Register + map (Burp):** created a guest account and watched the site map — `POST /auth/register` (201), `POST /claim` (200 JSON), `GET /dashboard/api/me` (balance), `GET /dashboard`. Auth = `connect.sid` cookie. Started at 50/150 PONZI.
2. **Turbo Intruder attempt #1 (single-packet):** used the gated single-packet script (`Engine.BURP2`, `openGate`). It **hung with `Reqs: 0 | Fails: 2 | Completed`** — nothing sent. Cause: HTTP/1.1 + `Content-Length: 0` means there is no final body byte to hold, so the sync gate can't work.
3. **Turbo Intruder attempt #2 (threaded blast):** switched to `Engine.THREADED`, `concurrentConnections=30`, queued 30 copies of `target.req`, no gate. Now `Reqs: 30 | RPS: 30 | Connections: 60` — all fired. But **every response was `429` "Reward already claimed"** (`secondsRemaining` ≈ 85969). The Cookie header confirmed the requests were authenticated, so the block was the cooldown, not auth.
4. **Diagnosis:** the account had already claimed (timer running) → no eligible window to race. Racing can only win when the claim is currently eligible.
5. **Winning run:** registered a **fresh account**, copied its new `connect.sid`, pasted it over the Cookie in the Turbo Intruder request, and ran the same threaded burst **immediately** (never clicking Claim manually first). Several requests returned 200 and stacked +50 each.
6. **Result:** balance jumped to **1,500 PONZI → WHALE**, Whale Vault unlocked, flag displayed in the vault (`THM{t0w3l_0n_th3_sunb3d_d0ubl3_sp3nt}` — the name spells out the *double-spend*).

## Turbo Intruder script (the one that worked)

```python
def queueRequests(target, wordlists):
    # Plain threaded concurrency blast — reliable for HTTP/1.1, body-less POSTs
    # where the single-packet / last-byte-sync attack cannot be used.
    engine = RequestEngine(endpoint=target.endpoint,
                           concurrentConnections=30,
                           requestsPerConnection=1,
                           engine=Engine.THREADED)
    for i in range(30):
        engine.queue(target.req)

def handleResponse(req, interesting):
    table.add(req)
```

## Key facts

| Item | Value |
| ---- | ----- |
| App | Ponzi Portfolio — Express, port 3000 |
| Vulnerable endpoint | `POST /claim` (check-then-write cooldown) |
| Auth | `connect.sid` session cookie |
| Cooldown | +50 PONZI / 24h; `429 {"secondsRemaining":...}` when on cooldown |
| Gate | 150 PONZI → Whale Vault |
| Race window | first eligible claim on a **fresh** account (null `last_claim`) |
| Winning technique | threaded burst (30 concurrent `POST /claim`) on a just-registered account |
| Result | 1,500 PONZI (WHALE) → vault opened |
