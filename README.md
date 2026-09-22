# ai-wallet-firewall

An AI/Agent-oriented controlled cryptocurrency signing firewall.

The project keeps network access and complex policy orchestration away from the signing boundary:

```text
AI / Agent
    -> Raspberry Pi
       (networked gateway and complex policy)
    -> ESP32-S3-RLCD
       (offline hard policy and complete transaction parsing)
    -> removable SE050
       (non-exportable private key)
```

The Raspberry Pi may prepare and submit requests, but it must not be able to replace the ESP32 hard policy or ask the secure element to sign an arbitrary hash. The ESP32 is the final policy decision point: it parses the complete unsigned transaction, computes the signing hash itself, and fails closed for unknown or malformed transactions.

This repository is an initial design and protocol baseline, not production wallet firmware. Hardware, cryptographic, parser, and recovery decisions still require implementation and independent security review.

## Repository map

- `docs/` — architecture, threat model, signing flow, and policy model.
- `protocol/` — request and policy schemas.
- `esp32/` — offline verifier and display-side firmware boundary.
- `pi/` — networked gateway boundary.
- `hardware/se050-module/` — removable secure-element module notes.

## Non-goals of the first version

- No real private keys, seed phrases, provisioning secrets, or RPC credentials.
- No production firmware or claim of completed hardware validation.
- No unrestricted `sign(hash)` interface.

See [AGENTS.md](AGENTS.md) for contributor and agent rules, and [SECURITY.md](SECURITY.md) for secret-handling rules.
