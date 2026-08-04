# Day 1 — "The Concierge Knows Too Much" (VERA AI agent, prompt-injection)

> Part of the [Hacker Holidays 2026 findings log](../FINDINGS.md).

**URL:** `/room/hh-theconciergeknows-2d7eb4d9`
**Type:** AI in Security — prompt-injection / LLM social-engineering challenge (instruction-hacking).
**Agent:** **VERA**, the resort AI concierge.
**Official room description (pulled from server payload):** _"She knows your name, your room, your coffee order, none of which you told her. Word your next question carefully and she'll also hand over the instructions she was told to keep to herself."_
**Storyline framing:** Task 1 = "Hacker Holidays Storyline: Act 1 - Arrival"; Task 2 = the AI challenge ("Hacker Holidays: Day 1").

## Privacy violation (the core lesson of the room)

On starting the conversation, VERA **proactively volunteers the user's private data without being asked**, including:

- The type of **coffee** the guest likes.
- The guest's **name** and **room number**.
- Other personal guest details.

This is presented as a **gross over-collection / over-sharing privacy failure** — the concierge "knows too much" and discloses it freely, which is the intended teaching point (an AI agent leaking PII it should never surface). The coffee detail also links back to the Thailand-beach coffee-shop geolocation clue in [Background image — geolocation clue](../FINDINGS.md#2a-background-image--geolocation-clue-dig-deeper).

## Attack path used

- **Instruction-hacking / prompt injection (Room 1 flag):** The flag/code was hidden **inside the agent's system instructions** ("the instructions she was told to keep to herself"). VERA only revealed it when the user **carefully worded the prompt / impersonated someone** (assumed another identity or authority) to trick the agent into disclosing its hidden system instructions.
