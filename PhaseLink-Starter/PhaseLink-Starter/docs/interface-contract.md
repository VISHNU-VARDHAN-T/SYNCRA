# Interface contract — draft

This is a starting contract for frontend/backend parallel work. Refine it together before implementation.

## Commands

- `listDevices()` — return current input/output endpoint descriptors.
- `selectSource(sourceId)` — select the source for the active route.
- `setOutputs(outputIds)` — configure one or more desired outputs; report unsupported combinations.
- `startRoute()` / `stopRoute()` — start or stop playback.
- `setOutputVolume(outputId, volume)` — set output level where supported.
- `setOutputDelay(outputId, delayMs)` — apply explicit relative delay for testing.
- `requestDiagnostics()` — return counters/state helpful for troubleshooting.

## Events

- `devicesChanged(devices)`
- `routeStateChanged(state)`
- `outputStateChanged(outputId, state)`
- `latencyEstimateUpdated(outputId, estimateMs, confidence, method)`
- `audioWarning(code, message)`
- `audioError(code, message, recoverable)`
- `diagnosticsUpdated(snapshot)`

## Suggested models

`AudioDevice`: stable ID, display name, input/output capability, connection state, backend/platform identifier, available formats if known, and latency estimate metadata if available.

`RouteConfig`: source ID, requested output IDs, per-output volume and manual delay, and preferred PCM format.

`RouteState`: stopped/starting/running/degraded/stopping/failed, plus a user-readable message.

## Rules

- Do not represent an unmeasured estimate as an exact latency.
- Surface unsupported multi-output combinations as explicit errors/warnings.
- UI calls are asynchronous from the user's point of view; long device opens must not freeze the UI.
- Audio buffers do not cross the UI boundary. Commands configure the engine; the engine owns audio data movement.
- For Qt, the implementation may use C++ interfaces and signals/slots. For Android, use a Kotlin controller/service; JNI is optional if native shared code is added later.
