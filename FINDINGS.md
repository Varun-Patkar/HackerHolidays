# Hacker Holidays 2026 — "The Byte Lotus" — Findings Log

**Target:** https://tryhackme.com/hackerholidays
**Event:** TryHackMe "Hacker Holidays 2026" — a 14-day beginner-friendly cyber security challenge set in a fictional five-star resort, _The Byte Lotus_ ("A five-star resort with a zero-star security posture").
**Date documented:** 2026-07-28

> **Per-room writeups now live in [`days/`](days/).** This file keeps the landing-page recon, the cross-room analysis, and an index linking each day's writeup.

---

## 1. Landing page overview

- Themed as a resort. New room opens daily at **4PM GMT/UTC** starting **27 July 2026**.
- Challenge categories advertised: **OSINT · Web Hacking · API Hacking · AI in Security · Forensics · Boot2Root**.
- Central antagonist / mascot: **VERA, the AI concierge** — described on the page as the AI "who remembers everything about everyone" and "knows absolutely everything."
- There is a **separate free OSINT warm-up room (Room 0)** unlocked before the event, plus the daily resort rooms below. Rooms 1–2 are open/completed; the rest unlock one per day:

| #   | Room name                                              | Type / status                          | Writeup |
| --- | ------------------------------------------------------ | -------------------------------------- | ------- |
| 0   | OSINT warm-up ("Pick up your key before the check-in") | ✅ Warm-up — VERA Instagram OSINT flag | [day-0](days/day-0-osint-warmup.md) |
| 1   | The Concierge Knows Too Much                           | ✅ Completed — AI prompt-injection     | [day-1](days/day-1-the-concierge-knows-too-much.md) |
| 2   | Room 404                                               | ✅ Completed — Web / dir enum (.git)   | [day-2](days/day-2-room-404.md) |
| 3   | Complimentary                                          | ✅ Completed — Cloud / Cognito+DynamoDB | [day-3](days/day-3-complimentary.md) |
| 4   | Packed Light                                           | ✅ Completed — Forensics / PCAP beacon | [day-4](days/day-4-packed-light.md) |
| 5   | Beach Bar                                              | ✅ Completed — Boot2Root / YAML deser. + cred reuse | [day-5](days/day-5-beach-bar.md) |
| 6   | Overheard at Breakfast                                 | ✅ Completed — OSINT / Gravatar email-hash pivot | [day-6](days/day-6-overheard-at-breakfast.md) |
| 7   | Do Not Disturb                                         | ✅ Completed — Boot2Root / NoSQLi → EJS SSTI → Node inspector → disk group | [day-7](days/day-7-do-not-disturb.md) |
| 8   | Towel on the Sunbed                                    | ✅ Completed — Web / race condition (TOCTOU double-spend) | [day-8](days/day-8-towel-on-the-sunbed.md) |
| 9   | CryptoCabana                                           | ✅ Completed — Cloud / Azure Storage SAS leak → Key Vault secret versioning | [day-9](days/day-9-cryptocabana.md) |
| 10  | The Hollow Shell                                       | ✅ Completed — Web / Zip-Slip + LFI → worker RCE | [day-10](days/day-10-the-hollow-shell.md) |
| 11  | Infinity Pool                                          | ✅ Completed — Boot2Root / cmd injection → chisel pivot → FreePBX UCP → root argument injection | [day-11](days/day-11-infinity-pool.md) |
| 12  | After Hours                                            | ✅ Completed — Forensics / WMI persistence → rogue class ConfigData → .NET payload | [day-12](days/day-12-after-hours.md) |
| 13  | The Guestbook                                       | ✅ Completed — AI / indirect prompt-injection → override RCE | [day-13](days/day-13-the-guestbook.md) |
| 14  | Management Wants a Word                                | 🔒 Locked                              | — |
| —   | Reward chest                                           | 🔒 Locked until all rooms completed    | — |

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
- Resolves in Google Maps to a **beach in Thailand** (Koh Phangan / Gulf of Thailand area) — a location described as having **lots of coffee shops**. This ties directly into VERA's "favourite coffee" data leak (see the [Day 1 privacy violation](days/day-1-the-concierge-knows-too-much.md#privacy-violation-the-core-lesson-of-the-room)).

### 2b. Single shell image — decoded base64 message

> **"It was never a bug. It was the business model."**

### 2c. Three-shells image — three separate decoded base64 messages

- **Large shell:** _"The prep track was supposed to be a formality. It isn't anymore."_
- **Left shell:** _"If you're reading this, you decoded a signal the resort never meant to broadcast."_
- **Small right shell:** _"Someone left a door open on purpose."_

**Interpretation:** These are narrative/ARG breadcrumbs. Combined theme — the resort is knowingly leaking data ("business model," "door open on purpose"), and the OSINT "prep track" (warm-up) is more than a formality. They reinforce that hidden signals are intentionally planted for players who decode the assets.

---

## 3. Room writeups (index)

Each room's full writeup — objective, story hook, attack path, flag (spoiler-tagged), walkthrough, and lessons — lives in its own file under [`days/`](days/):

| Day | Room | Category | Writeup |
| --- | ---- | -------- | ------- |
| 0 | OSINT warm-up | OSINT | [day-0-osint-warmup.md](days/day-0-osint-warmup.md) |
| 1 | The Concierge Knows Too Much | AI / prompt-injection | [day-1-the-concierge-knows-too-much.md](days/day-1-the-concierge-knows-too-much.md) |
| 2 | Room 404 | Web / dir enum | [day-2-room-404.md](days/day-2-room-404.md) |
| 3 | Complimentary | Cloud / AWS Cognito+DynamoDB | [day-3-complimentary.md](days/day-3-complimentary.md) |
| 4 | Packed Light | Forensics / PCAP + crypto | [day-4-packed-light.md](days/day-4-packed-light.md) |
| 5 | Beach Bar | Boot2Root / YAML deser. + cred reuse | [day-5-beach-bar.md](days/day-5-beach-bar.md) |
| 6 | Overheard at Breakfast | OSINT / Gravatar hash pivot | [day-6-overheard-at-breakfast.md](days/day-6-overheard-at-breakfast.md) |
| 7 | Do Not Disturb | Boot2Root / NoSQLi → SSTI → disk group | [day-7-do-not-disturb.md](days/day-7-do-not-disturb.md) |
| 8 | Towel on the Sunbed | Web / race condition (TOCTOU) | [day-8-towel-on-the-sunbed.md](days/day-8-towel-on-the-sunbed.md) |
| 9 | CryptoCabana | Cloud / Azure SAS leak → Key Vault versioning | [day-9-cryptocabana.md](days/day-9-cryptocabana.md) |
| 10 | The Hollow Shell | Web / Zip-Slip + traversal LFI → theme-worker RCE | [day-10-the-hollow-shell.md](days/day-10-the-hollow-shell.md) |
| 11 | Infinity Pool | Boot2Root / cmd injection → chisel pivot → FreePBX UCP → root arg injection | [day-11-infinity-pool.md](days/day-11-infinity-pool.md) |
| 12 | After Hours | Forensics / WMI Event Subscription persistence → rogue class `ConfigData` → embedded .NET payload | [day-12-after-hours.md](days/day-12-after-hours.md) |
| 13 | The Guestbook | AI / indirect prompt-injection → authorization bleed → `override:` shell RCE | [day-13-the-guestbook.md](days/day-13-the-guestbook.md) |

**Locked (not yet released):** Room 14 — Management Wants a Word · Reward chest (unlocks once every room is completed). Day files will be added as each opens (one daily at **4PM UTC**).

---

## 4. Key takeaways / working theory

- The ARG rewards **decoding image-embedded signals** (base64 in webp/jpg) rather than reading page text.
- The narrative thread: the resort **intentionally leaks data** and **left a "door open on purpose"** — social engineering and privacy failure are the recurring motifs.
- VERA is the linchpin: an over-sharing AI concierge vulnerable to **impersonation-based prompt injection**, with an OSINT trail leading to **Instagram `@veratheconcierge`**.
- Geolocation clue (**9.5681° N, 100.0602° E**, Thailand beach, coffee shops) cross-references VERA's leaked "favourite coffee" data point.

---

## 5. Deep technical sweep — things the human eye skips (agent-parsed)

> Goal: surface content that isn't human-visible on the rendered page. Result: the image codes are **visual overlays** (confirmed below), and the only genuinely non-visible data of interest is in the Next.js server payload.

### 5a. Image byte-level analysis (stego check) — all clean

- Byte-scanned every asset (`background`, `shell`, `shells`, `house`, `key`, `roadmap-chest`, `lotus-outline.svg`) for embedded ASCII / base64 runs and appended data. **No hidden strings** — only lossy-WebP codec noise.
- The OG image `img/meta/the-hacker-holidays-og.png` (lossless PNG, where stego would survive) contains **only a `tEXtSoftware=Figma` chunk**; no text chunks, no trailing data after `IEND`.
- **Conclusion:** the shell/background codes are **rendered pixels (visual text overlays)**, not byte-embedded data. Pixel-reading or byte-scanning an agent does will NOT reveal them — they must be read visually from the image, exactly as you noted.

### 5b. Next.js server payload (`__next_f` RSC stream) — hidden event config

Not visible on-page, but parsed from the streamed React payload:

- **EventConfig:** `eventName: "Hacker Holidays"`, `pageUrl: hackerholidays`, `eventCode: "Hacker Holidays"`
- **startDate:** `2026-07-27T16:00:00.000Z` **endDate:** `2026-08-12T21:59:00.000Z`
- **`ctfRoomCode: 6a639245d468dcd0da08e52a`** — internal backend identifier for the CTF room (not shown anywhere in the UI).
- Room 1 description string (also hidden in payload): _"...Word your next question carefully and she'll also hand over the instructions she was told to keep to herself."_ — this is the **official hint** for the instruction-hacking attack.

### 5c. Other vectors checked — nothing hidden

- **Meta / OG / Twitter tags:** standard marketing copy only; OG image `img/meta/the-hacker-holidays-og.png`.
- **HTML comments, `data-*` attributes, off-screen / zero-opacity / tiny-font text:** only framework + analytics boilerplate (Intercom, Segment, GA, customer.io), no clues.
- **Additional media asset referenced:** `full-background.*.webp` (a second, larger background variant distinct from the coordinates image) — byte-clean.
- **Animated hero:** a **Rive** animation ("Animated Hacker Holidays palm trees") renders to a canvas via WASM; no exposed text layer in the DOM.
- **Network / API traffic:** only analytics/telemetry endpoints (GA, Intercom, Segment, gist.build). No room/flag data exposed to the client.
- **JSON-LD:** `Organization` + `Event` schema; links to official `instagram.com/realtryhackme` (not the in-game `@veratheconcierge`).

---

## 6. Verified artifacts / references

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
