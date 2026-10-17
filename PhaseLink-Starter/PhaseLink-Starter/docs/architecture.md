# PhaseLink architecture (proposal)

## Principle

Keep the user interface, portable routing policy, and operating-system audio adapters separate. The first implementation is Windows-first and local-only.

```text
Qt UI
  |
  v
App/controller interface  <---->  Mock controller (early UI work)
  |
  v
Route manager / device models
  |
  v
PCM source -> fan-out -> per-output queue -> delay/resampling policy
  |
  v
Windows endpoint adapter (WASAPI)
  |
  +--> output endpoint A
  +--> output endpoint B
```

## Layers

1. **UI:** device selection, route state, output controls, diagnostics.
2. **Controller/API:** typed commands/events; validation and error reporting.
3. **Portable core:** device/route models, source abstraction, queue policy, sync calculations, diagnostics.
4. **Platform adapters:** enumerate/open/render/stop audio endpoints using the operating system's supported APIs.
5. **Test layer:** fake endpoints and deterministic PCM test data, then manual testing on physical endpoints.

## Important design choices

- Begin with generated PCM test audio; add file decoding as a separate feature.
- First prove one output, then two outputs.
- Each output needs its own queue and error state because devices have independent clocks and latency.
- Use configurable delay before automatic calibration.
- Do not promise exact synchronization before end-to-end measurement.
- System-wide audio interception is outside the first MVP and may require a virtual audio device/driver or supported OS routing.
- Android must use its supported audio routing model. The portable core cannot override restrictions imposed by Android, Bluetooth profiles, firmware, or hardware.
