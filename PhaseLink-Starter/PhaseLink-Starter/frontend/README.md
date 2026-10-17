# Frontend workspace

The frontend contains platform-specific user interfaces. UI code should call a small app/controller interface; it should not directly manage WASAPI, Bluetooth internals, audio callbacks, or output buffers.

- `windows/`: planned Qt 6 desktop UI. Start with a mock device provider and simulated route states.
- `android/`: planned Kotlin + Jetpack Compose app. Implement after the Windows MVP proves the device/route concepts.

No frontend source code is included yet.
