# Policy model

The hard policy is the ESP32's local authorization contract. The Pi may propose a transaction but cannot change this contract during run mode.

## Policy package

A policy package contains:

- a schema version and monotonic policy version;
- supported chain IDs;
- hard per-transaction and rolling-period limits;
- allowed contract addresses and method selectors;
- explicitly blocked methods;
- operations requiring a separate physical approval;
- an administrative signature and policy hash in the provisioning format.

The JSON schema in [`protocol/policy.schema.json`](../protocol/policy.schema.json) is a control-plane baseline. Exact canonical serialization, admin-key rotation, rollback protection, and on-device storage format must be specified before production use.

## Evaluation order

The ESP32 should reject on the first failed check:

1. framing, size, and schema validation;
2. supported chain and transaction type;
3. complete transaction parsing and canonical reconstruction;
4. destination, method, and value restrictions;
5. per-transaction and period limits;
6. replay and nonce rules;
7. explicit physical-approval requirements.

Unknown fields, unknown methods, missing limits, and parser ambiguity fail closed.

## Installation boundary

Policy installation is an admin-mode operation initiated by a physical action, with signing disabled while the update is active. The Pi cannot invoke policy installation through the run-mode protocol.
