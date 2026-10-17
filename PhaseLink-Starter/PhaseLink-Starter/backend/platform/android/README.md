# Android audio adapter (planned)

**Planned stack:** Kotlin Android audio APIs; optionally C++ via Android NDK + JNI later.

Investigate the platform's supported output routing and `AudioTrack` behaviour before implementing multi-output assumptions. This adapter must report real device/API limitations rather than promising independent control of every Bluetooth device.

No implementation is included yet.
