# Day 6 — Overheard at Breakfast

> Part of the [Hacker Holidays 2026 findings log](../FINDINGS.md).

- **Status:** ✅ Completed
- **Target:** downloadable task files (a chat screenshot) — no lab machine needed.
- **Category:** OSINT → Social Media + Hashing (Gravatar email-hash pivot). Tags on the room card: **OSINT · Social Media · Hashing**.
- **Objective:** Analyze an overheard breakfast conversation, extract the identifying details, and pivot to a hidden account nobody was supposed to find.
- **Story hook:** @0xMia's in-game post — _"the breakfast crowd really said the quiet part out loud this morning... y'all need to actually READ what they said, not just skim it #HackerHolidays"_ — i.e. the clues are in the text of the chat, not the images.
- **The conversation (task files):** "Ponzi – Influencer" fishes "Lambo" for a social handle. Lambo deflects, but leaks three tells:
  1. He used to use a **free tool that let you upload a profile and link other media accounts** — an aggregator.
  2. That tool **"started with a G"** → **Gravatar**.
  3. He **"wiped everything"** (deleted the profile).
  4. His best contact: **`lambobytelotushotel@gmail.com`**.
- **The pivot ("funny thing about email hashes"):** Gravatar keys public profiles off the **MD5 hash of the (lowercased) email**. Hashing the leaked address exposes the wiped-but-still-reachable profile page.
  - `MD5(lambobytelotushotel@gmail.com)` = `d4a5fc5d3128890778667e24617d7cc0`
  - Profile URL: `https://gravatar.com/d4a5fc5d3128890778667e24617d7cc0`
- **The prize:** the Gravatar bio (name "Lambo · Byte Lotus Hotel") reads _"Funny thing about email hashes, they follow you places you didn't expect... Here is your prize:"_ followed by a base64 blob:
  - `VEhNe1MzY3JlVF9QcjBmaWwzX0g0c19iMzNuX0lkZW50MWZpM2R9`
  - Base64-decode → the flag.
- **Flag:** <details><summary>click to reveal</summary><code>THM{S3creT_Pr0fil3_H4s_b33n_Ident1fi3d}</code></details>
- **Lesson:** "Deleting" a profile on an aggregator service doesn't delete the deterministic lookup key. Gravatar (and many services) address accounts by `MD5(email)` / `SHA-256(email)`, so anyone who overhears the email can reconstruct the profile URL directly — an email address is effectively a permanent, unauthenticated public identifier. Never assume a wiped social profile is unreachable, and never leak an email you use to register aggregator accounts.

## Walkthrough

1. **Read the chat, don't skim it** (@0xMia's hint). The identifying details are: an aggregator tool "starting with G" = **Gravatar**, and the email **`lambobytelotushotel@gmail.com`**.
2. **Hash the email** (Gravatar uses MD5 of the email):

   ```powershell
   $e = "lambobytelotushotel@gmail.com"
   $md5 = [System.Security.Cryptography.MD5]::Create()
   ($md5.ComputeHash([Text.Encoding]::UTF8.GetBytes($e)) | ForEach-Object { $_.ToString("x2") }) -join ""
   # -> d4a5fc5d3128890778667e24617d7cc0
   ```

3. **Visit the Gravatar profile** at `https://gravatar.com/<md5>` → the bio contains a base64 "prize" string.
4. **Decode the base64** to recover the flag:

   ```powershell
   [Text.Encoding]::UTF8.GetString([Convert]::FromBase64String("VEhNe1MzY3JlVF9QcjBmaWwzX0g0c19iMzNuX0lkZW50MWZpM2R9"))
   # -> THM{S3creT_Pr0fil3_H4s_b33n_Ident1fi3d}
   ```

5. **Submit** the flag. ✅ (Difficulty: very easy — a straight OSINT read-and-hash pivot.)

## Clue chain

| Clue in task files | Resolves to |
| ------------------ | ----------- |
| "free tool… upload profile, link other media accounts" | a profile aggregator |
| "Started with a **G**" | **Gravatar** |
| room tag **Hashing** | Gravatar keys profiles on `MD5(email)` |
| `lambobytelotushotel@gmail.com` | `MD5` → `d4a5fc5d3128890778667e24617d7cc0` |
| "wiped everything" | profile deleted, but hash URL still resolves |
| base64 in bio | `THM{S3creT_Pr0fil3_H4s_b33n_Ident1fi3d}` |
