# Backend workspace

In this project, **backend means the local audio engine**, not a cloud server.

- `core/`: planned portable models, route orchestration, buffering/synchronization policy, and diagnostics.
- `platform/windows/`: planned Windows Core Audio/WASAPI endpoint discovery and rendering.
- `platform/android/`: planned adapter to Android audio APIs. An optional shared C++ core can be exposed via JNI later.

Keep audio rendering independent from the UI. Do not do blocking I/O, UI work, or unpredictable allocations in real-time audio callbacks. No backend source code is included yet.
