# Hacker Holidays 2026 — "The Byte Lotus" 🏝️

Investigation notes, OSINT findings, and write-ups for [**TryHackMe's Hacker Holidays 2026**](https://tryhackme.com/hackerholidays) — a free, 14-day, beginner-friendly cyber security challenge set in a fictional five-star resort, *The Byte Lotus* ("a five-star resort with a zero-star security posture").

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
- **Image-embedded clues** — base64 messages hidden as *visual overlays* in the shell/background webp assets, plus the geolocation clue (`9.5681° N, 100.0602° E` → a coffee-shop-lined beach in Thailand).
- **Room 0 (OSINT warm-up)** — the flag hidden on VERA's public Instagram (`@veratheconcierge`).
- **Room 1 (AI prompt-injection)** — instruction-hacking VERA, the AI concierge, into leaking the system instructions she was told to keep to herself; plus the privacy-failure lesson (VERA volunteers your name, room, and coffee order unprompted).
- **Deep technical sweep** — byte-level stego checks (all clean), the Next.js `__next_f` server payload (hidden event config + `ctfRoomCode`), and every other vector a human eye skips (meta tags, hidden DOM text, Rive animation, network/API traffic).

---

## 🤖 Built with GitHub Copilot

This entire investigation and documentation was carried out interactively with **[GitHub Copilot](https://github.com/features/copilot)** (agent mode in VS Code). Copilot's tooling did the heavy lifting that a human would otherwise skim past:

- **Browser automation (Playwright MCP)** — navigated the challenge pages, captured accessibility snapshots, and drove the DOM programmatically.
- **Byte-level asset analysis** — fetched every image through the authenticated browser session (bypassing rate limits) and scanned raw bytes for embedded ASCII / base64 / stego payloads and PNG text chunks.
- **Server-payload parsing** — reconstructed and searched the Next.js React Server Components (`__next_f`) stream to surface config never rendered on screen (e.g. `ctfRoomCode`, event dates, the official challenge hint).
- **Documentation** — compiled, cross-linked, and iteratively refined [FINDINGS.md](FINDINGS.md).
- **Git & GitHub** — initialized the repo and published it via the GitHub CLI.

> 💡 **Key insight surfaced by the agent:** the shell/background "codes" are **rendered pixels (visual text overlays)**, not byte-embedded data — so byte-scanning or pixel-reading won't reveal them. They must be read visually from the images. Conversely, an AI agent *can* trivially parse things humans skip, like the streamed server payload.

---

## ⚠️ Disclaimer

TryHackMe's Hacker Holidays is a **public, sanctioned Capture-The-Flag / ARG event** — decoding these intentionally planted clues is in-scope by design. This repo contains research notes only. No spoilers for locked rooms, no exploitation of out-of-scope systems.

---

## 📄 License

Released under the [MIT License](LICENSE).
