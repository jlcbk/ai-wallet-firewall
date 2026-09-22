# Signing flow

1. The AI/Agent expresses an intent to the Raspberry Pi.
2. The Pi resolves chain data and constructs one complete unsigned transaction.
3. The Pi sends the transaction request to the ESP32 over the local transport.
4. The ESP32 validates framing, schema, sizes, chain ID, nonce, and supported transaction type.
5. The ESP32 parses all signed fields and calldata itself. It does not accept a host-supplied digest.
6. The ESP32 evaluates the parsed transaction against the installed hard policy.
7. If the transaction is unknown, malformed, unsupported, over a limit, or ambiguous, the ESP32 rejects it and does not contact the SE050.
8. If policy allows it, the ESP32 computes the signing hash from its own parsed representation and asks the SE050 to sign the approved payload.
9. The ESP32 returns only the signature and decision metadata needed by the Pi.

The signing interface must be transaction-aware. A future implementation may support multiple chain-specific encodings, but each supported encoding needs a bounded parser, canonical hashing rule, and negative test vectors.
