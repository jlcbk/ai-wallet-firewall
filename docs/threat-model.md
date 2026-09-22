# Threat model

## Protected assets

- The non-exportable private key held by the SE050.
- The authorization meaning of the installed ESP32 hard policy.
- User-visible transaction details and spending limits.
- Auditability of accept, reject, and unknown decisions.

## Primary attacker assumptions

- The AI/Agent can be manipulated by untrusted content.
- The Raspberry Pi or its network services can be fully compromised.
- A caller can send malformed, replayed, oversized, or semantically surprising requests.
- The host may lie about chain ID, destination, value, calldata, fees, or its own display.

## Security goals

1. A compromised Pi cannot export the key or request a signature over an arbitrary digest.
2. A compromised Pi cannot replace or relax the ESP32 hard policy in run mode.
3. An unknown transaction type or parser ambiguity produces rejection, not best-effort signing.
4. The signed payload is the transaction the ESP32 parsed and authorized.
5. Policy changes are visible, deliberate, authenticated, and physically bounded.

## Known limits of this baseline

This document does not yet prove resistance to physical extraction, supply-chain compromise, firmware replacement, side-channel attacks, SE050 configuration mistakes, or every supported chain format. Those are separate implementation and review work items.
