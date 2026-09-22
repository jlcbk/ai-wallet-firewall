# Removable SE050 module

The SE050 is the signing trust anchor. It holds a non-exportable private key and performs signing only after the ESP32 has parsed and authorized the complete transaction.

## Boundary

- The module is removable from the ESP32 assembly for controlled provisioning, maintenance, and replacement.
- The Raspberry Pi and AI/Agent never receive the private key.
- The ESP32 must not expose a raw-hash signing capability to the Pi.
- Provisioning secrets, admin/root keys, and real device credentials stay outside Git.

## Open implementation work

- finalize the electrical connector and presence detection;
- define SE050 object IDs, access conditions, and provisioning ceremony;
- validate reset, removal, replacement, and recovery behavior;
- test that a missing or uninitialized module fails closed.
