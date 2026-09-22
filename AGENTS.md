# Agent instructions

This repository defines a security boundary, not merely a transaction relay. Keep changes small, explicit, and reviewable. Do not weaken a security invariant to make a demo pass.

## Mandatory security invariants

1. Never expose an arbitrary `sign(hash)` or equivalent raw-digest signing API.
2. Private keys must never be exported from the SE050, ESP32, Raspberry Pi, logs, fixtures, or documentation.
3. The Raspberry Pi has no authority to modify the ESP32 hard policy in run mode.
4. The ESP32 must receive and parse the complete unsigned transaction and compute the signing hash itself before requesting a signature.
5. Unknown, malformed, unsupported, or ambiguously decoded transactions fail closed.
6. Policy installation requires an explicit physical administration flow; ordinary Pi traffic cannot install or reset policy.
7. Production firmware must disable networking on the ESP32 signing device. Communication is limited to the intended local transport.
8. Admin mode must not silently continue normal signing while policy changes are being made.
9. Every protocol change must update its schema, documentation, and negative test vectors before implementation is treated as complete.

## Working rules

- Treat all Pi and AI/Agent input as untrusted.
- Prefer deterministic, canonical encodings and bounded parsers.
- Reject unknown fields when they could change authorization semantics.
- Keep simulator evidence, firmware evidence, and physical-device evidence separate.
- Never add credentials or real transaction secrets to examples, tests, commits, or issue text.
- A compile or schema check does not prove device security or transaction correctness.
