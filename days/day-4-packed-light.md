# Day 4 — Packed Light

> Part of the [Hacker Holidays 2026 findings log](../FINDINGS.md).

- **Status:** ✅ Completed
- **Target:** offline packet capture `traffic.pcapng` (downloaded via the room's "Download Task Files" button) — no lab machine needed.
- **Category:** Network Forensics → PCAP analysis + Cryptography (Easy)
- **Objective:** Find the covert channel in the capture, work out where the exfiltrated data is hidden, reassemble and decode it.
- **Story hook:** _"Tiny packets. Odd hours. Suspiciously regular. Someone's smuggling out the data equivalent of a hotel towel every night, folded neatly inside traffic that looks ordinary until you decode it."_ Backed by @0xMia's in-game post: _"my laptop ping some random :8080 address every single second like clockwork... the request headers are giving 'not a real app'."_
- **The covert channel:** a fake client (`User-Agent: ... ByteLotusClient/1.1`) beacons `GET /` to `byte-lotus-hotel.thm:8080` roughly once per second. Each request carries a **`Cookie: hotel_sess_state=<base64>`** header holding exactly **one byte** of payload. The HTTP responses are a full, innocuous resort homepage — pure decoy.
- **Encoding chain:** `1 byte → XOR 0x48 → base64 → cookie value`. Decoding is the reverse: base64-decode each cookie to a single byte, concatenate **in frame order**, XOR every byte with `0x48`.
- **Flag:** <details><summary>click to reveal</summary><code>THM{V3r4_1s_w4tch1ng_0veR_y0u}</code></details>
- **Lesson:** Beaconing is identified by **regularity, not content** — a fixed-interval request to a fixed endpoint is suspicious regardless of how legitimate each individual packet looks. Exfil hides in request *metadata* (cookies, headers, URI paths, DNS labels), not just bodies, and single-byte-per-request chunking keeps every packet small enough to slip past size-based detection.

## Walkthrough

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

## Narrative note

The decoded message — _"VERA is watching over you"_ — puts **VERA** behind the exfiltration, tying Room 4 back to the Room 1 over-sharing concierge and the landing-page breadcrumb _"It was never a bug. It was the business model."_ The resort isn't leaking data by accident; VERA is shipping it out one byte at a time.
