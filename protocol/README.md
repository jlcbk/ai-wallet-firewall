# Protocol baseline

This directory contains the first control-plane schemas shared by the Raspberry Pi and ESP32.

- `transaction-request.schema.json` describes a complete unsigned transaction request. It is not a raw digest request.
- `policy.schema.json` describes the policy data model before its authenticated provisioning envelope is finalized.

Schemas are validation aids, not proof of authorization. The ESP32 must enforce bounded parsing, supported transaction types, canonical encoding, hard policy, and fail-closed behavior in firmware. Wire encoding and transport framing remain implementation decisions and must not introduce an arbitrary `sign(hash)` path.
