## Security: verify TLS + message signatures on the Pulsar stream

Two small hardening changes in `openpulsar.py`:

- The WebSocket connected with `ssl=False` (no cert validation) → now uses a real
  default SSL context.
- `_verify_v2_sign()` existed but was never called → now we verify v2 message
  signatures and drop ones that fail.

Conservative: only enforces the signature check when a `sign` field is actually
present, so legacy/v1 messages and the happy path are unaffected.
