# Hacker Holidays 2026 — "The Byte Lotus" 🏝️

My personal journey log for [**TryHackMe's Hacker Holidays 2026**](https://tryhackme.com/hackerholidays) — a free, 14-day, beginner-friendly cyber security challenge set in a fictional five-star resort, _The Byte Lotus_ ("a five-star resort with a zero-star security posture").

Think of this repo as one half of a **pair**: I do all the actual hacking, decoding, and finding myself — and whenever I want something looked up online, cross-checked, byte-scanned, or written up cleanly, I hand it to my **GitHub Copilot agent** and log the result here. It's my running notebook for the path.

> 📄 Full write-up: **[FINDINGS.md](FINDINGS.md)**

---

## 🔗 Links

- **Challenge home:** https://tryhackme.com/hackerholidays
- **Room 1 — "The Concierge Knows Too Much":** https://tryhackme.com/room/hh-theconciergeknows-2d7eb4d9
- **Event window:** 27 July 2026 16:00 UTC → 12 August 2026 21:59 UTC

---

## 📝 What's inside

[FINDINGS.md](FINDINGS.md) documents the full trail:

- **Landing-page recon** — room roadmap, teaser copy, and the ARG storyline.
- **Image-embedded clues** — base64 messages hidden as _visual overlays_ in the shell/background webp assets, plus the geolocation clue (`9.5681° N, 100.0602° E` → a coffee-shop-lined beach in Thailand).
- **Room 0 (OSINT warm-up)** — the flag hidden on VERA's public Instagram (`@veratheconcierge`).
- **Room 1 (AI prompt-injection)** — instruction-hacking VERA, the AI concierge, into leaking the system instructions she was told to keep to herself; plus the privacy-failure lesson (VERA volunteers your name, room, and coffee order unprompted).
- **Deep technical sweep** — byte-level stego checks (all clean), the Next.js `__next_f` server payload (hidden event config + `ctfRoomCode`), and every other vector a human eye skips (meta tags, hidden DOM text, Rive animation, network/API traffic).

---

## 👥 Me + GitHub Copilot: how this pair works

The hacking is **mine** — the OSINT pivots, the decoding, the prompt-injection on VERA, the calls on what a clue means. What I hand off to my **[GitHub Copilot](https://github.com/features/copilot)** agent (agent mode in VS Code) is the grunt work and the online research I'd rather not do by hand, and I log it all here so the journey is reproducible:

- **Online research & lookups** — when I want something verified, expanded, or researched on the web, I ask the agent instead of context-switching.
- **Browser automation (Playwright MCP)** — navigating challenge pages, capturing snapshots, driving the DOM programmatically.
- **Byte-level asset analysis** — fetching images through the authenticated session (bypassing rate limits) and scanning raw bytes for embedded ASCII / base64 / stego payloads and PNG text chunks.
- **Server-payload parsing** — reconstructing and searching the Next.js React Server Components (`__next_f`) stream for config never rendered on screen (e.g. `ctfRoomCode`, event dates, the official hint).
- **Documentation & housekeeping** — compiling and cross-linking [FINDINGS.md](FINDINGS.md), and pushing to GitHub via the CLI.

> 💡 **Key insight from the pairing:** the shell/background "codes" are **rendered pixels (visual text overlays)**, not byte-embedded data — so byte-scanning or pixel-reading won't reveal them; I read them visually. Conversely, the agent _can_ trivially parse things a human skims past, like the streamed server payload. Different strengths, logged side by side.

---

## ⚠️ Disclaimer

TryHackMe's Hacker Holidays is a **public, sanctioned Capture-The-Flag / ARG event** — decoding these intentionally planted clues is in-scope by design. This repo contains research notes only. No spoilers for locked rooms, no exploitation of out-of-scope systems.

---

## 📄 License

Released under the [MIT License](LICENSE).
