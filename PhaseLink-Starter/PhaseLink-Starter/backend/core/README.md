# Portable audio core (planned)

Potential responsibilities:

- Shared device and route models.
- Source abstraction and PCM format definition.
- Fan-out orchestration and per-output queues.
- Manual delay policy, then later latency estimation/calibration.
- State transitions, diagnostics, and error types.

Keep the portable core free of Qt UI types and Windows-only headers where practical. Platform-specific endpoint code belongs in `backend/platform/`.

No implementation is included yet.
