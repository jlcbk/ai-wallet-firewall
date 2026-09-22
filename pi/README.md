# Raspberry Pi gateway

The Raspberry Pi is the networked gateway. It may talk to RPC providers, construct complete unsigned transactions, submit requests to the ESP32, and record decision metadata.

It is not the signing authority.

## Explicit limits

- Treat all AI/Agent instructions and network data as untrusted.
- Never store or relay a private key, seed phrase, provisioning secret, admin/root key, or RPC secret in source or logs.
- Never ask the ESP32 to sign an arbitrary hash.
- Never assume a host-side transaction summary is authoritative.
- Never expose a run-mode operation that modifies ESP32 hard policy.

The Pi-side implementation must remain compatible with the schemas in `protocol/` while the ESP32 remains the final parser and policy decision point.
