# Windows audio adapter (planned)

**Planned stack:** Windows SDK, Core Audio interfaces, WASAPI, C++20.

Responsibilities:
- Enumerate active input/output endpoints.
- Observe endpoint additions/removals/state changes.
- Open/close supported render streams.
- Render queued PCM data to each selected output and report endpoint errors.

Start in shared mode and with a generated test tone. Keep COM/WASAPI implementation details behind the core's platform adapter interface.

No implementation is included yet.
