# Security policy

## Scope

`ai-wallet-firewall` is an early-stage design and firmware project. It is not yet a production wallet or an audited signing appliance.

## Never commit these secrets

Do not commit, paste into issues, or place in logs or test fixtures:

- real private keys;
- seed phrases or mnemonics;
- SE050 provisioning secrets;
- admin keys or root keys;
- RPC credentials, API tokens, or webhook secrets;
- device backups containing any of the above.

Use clearly fake placeholders and keep local secrets outside the repository. The included `.gitignore` is a convenience, not a security boundary.

## Security boundary

The AI/Agent and Raspberry Pi are treated as potentially compromised. They may construct requests, but the ESP32 must independently parse the complete unsigned transaction and enforce hard policy before the removable SE050 is asked to sign. The private key remains non-exportable inside the SE050.

## Reporting

Do not open a public issue for a suspected vulnerability that could expose a key or authorize an unintended transaction. Until a private security contact is documented, preserve evidence locally and contact the repository owner through a private GitHub channel.
