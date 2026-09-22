# Architecture

## Components

| Component | Trust posture | Responsibility |
| --- | --- | --- |
| AI / Agent | Untrusted | Explain intent and prepare a candidate request. Never sees a private key. |
| Raspberry Pi | Networked, potentially compromised | Fetch chain data, build complete unsigned transactions, submit requests, and retain audit records. |
| ESP32-S3-RLCD | Offline policy boundary | Parse the complete unsigned transaction, enforce hard policy, display decision context, and compute the signing hash. |
| Removable SE050 | Hardware trust anchor | Hold the private key and perform an approved signing operation without key export. |

## Run mode

In run mode, the ESP32 exposes only the minimum request/status surface required by the Pi. Policy installation, reset, firmware update, and raw-hash signing are not run-mode operations.

The Pi can be compromised without gaining authority to change the hard policy. The ESP32 does not trust the Pi's interpretation of a transaction, displayed summary, or precomputed digest.

## Admin mode

Policy updates require an explicit physical administration action. Admin mode disables ordinary signing, authenticates the policy package, validates version and rollback rules, and records the installed policy hash before returning to run mode.

The exact button, display, and key ceremony are hardware decisions still to be implemented. The trust boundary is mandatory even while those details remain open.

## Data path

```text
intent -> Pi candidate transaction -> ESP32 full parse
      -> hard-policy decision -> local display/approval rules
      -> ESP32-computed digest -> SE050 signature
      -> signature/status -> Pi
```

No component before the ESP32 may define the digest that the SE050 signs.
