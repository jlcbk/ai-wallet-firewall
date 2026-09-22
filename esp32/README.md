# ESP32-S3-RLCD boundary

The ESP32-S3-RLCD is the offline signing firewall. It is responsible for the hard policy, complete transaction parsing, display of the relevant decision context, and the local request to the removable SE050.

## Required behavior

- no production networking;
- no arbitrary `sign(hash)` API;
- parse the complete unsigned transaction locally;
- compute the signing hash locally from the parsed transaction;
- reject malformed, unknown, unsupported, oversized, or ambiguous input;
- keep policy installation behind physical admin mode;
- stop normal signing while admin mode is active.

The firmware is not implemented by this baseline. Any implementation must preserve the invariants in the repository root `AGENTS.md`.
