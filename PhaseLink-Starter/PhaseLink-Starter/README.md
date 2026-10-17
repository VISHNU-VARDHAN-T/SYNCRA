# PhaseLink

**PhaseLink** is a planned multi-output audio application that aims to send one audio source to more than one supported audio endpoint—such as wired headphones and Bluetooth speakers—and keep playback as closely synchronized as practical.

This repository is a **starter structure only**. The frontend and backend implementation folders are intentionally empty. It contains no working app, driver, audio engine, or dependency lockfiles yet.

## Contents

1. [What the project is and why it is useful](#1-what-the-project-is-and-why-it-is-useful)
2. [Technology plan and required packages](#2-technology-plan-and-required-packages)
3. [Frontend-to-backend workflow](#3-frontend-to-backend-workflow)
4. [Work split for two people](#4-work-split-for-two-people)
5. [How to start and work in parallel](#5-how-to-start-and-work-in-parallel)
6. [Scope, constraints, and milestones](#6-scope-constraints-and-milestones)
7. [Repository structure](#7-repository-structure)
8. [Useful official documentation](#8-useful-official-documentation)

---

## 1. What the project is and why it is useful

### The idea

A user selects an audio source and one or more supported output devices. PhaseLink routes the same audio to those outputs and applies per-output buffering and delay compensation to reduce the audible echo caused by different device latencies.

Example target setup:

```text
             Audio source
                  |
                  v
          +----------------+
          | PhaseLink core |
          | buffer + sync  |
          +-------+--------+
                  |
          +-------+----------------+
          |              |         |
          v              v         v
       Wired          Bluetooth  Bluetooth
     headphones       earbuds     speaker
```

### Why it could be useful

- Play audio through a wired device and a Bluetooth device together.
- Share music across compatible devices in a room.
- Inspect available audio outputs from one interface.
- Adjust per-device delay and volume where supported.
- Learn and demonstrate real-time audio, operating-system APIs, buffering, resampling, latency measurement, and device lifecycle management.

### A realistic promise

The goal is **synchronized multi-output playback**, not a guarantee that every device can be combined. Each operating system and audio device imposes constraints. Ordinary Bluetooth devices generally cannot be turned into relays by a third-party app, and merely being Bluetooth-connected does not mean a device can transmit or receive arbitrary relayed audio.

The first milestone should use a known audio source or test tone. Capturing audio from *every* unrelated app (for example, Spotify, a browser, games, and calls) is a separate, harder milestone. On Windows, system-wide routing may require a virtual audio endpoint/driver or a supported system-routing setup. The Android implementation must stay within Android's audio-routing and capture permissions/capabilities.

## 2. Technology plan and required packages

### Recommended development order

Build **Windows first**, prove the audio pipeline, then implement the Android version. Keep both platform folders in the repository from day one, but don't attempt to finish two full platforms simultaneously.

### Windows desktop MVP

| Area | Recommended technology | Purpose |
|---|---|---|
| Language | C++20 | Native audio engine and desktop application |
| Build | CMake 3.24+ | Configure and build the C++ project |
| Compiler/IDE | Visual Studio 2022 with **Desktop development with C++**; VS Code is optional | MSVC compiler, debugger, Windows SDK |
| Audio output | Windows Core Audio / WASAPI | Open and render to Windows audio endpoints |
| Device discovery | WASAPI Core Audio interfaces / endpoint notifications | List devices and handle connect/disconnect events |
| UI | Qt 6 with Qt Quick/QML **or** Qt Widgets | Device list, output selection, sync controls, logs |
| Qt build integration | Qt 6 CMake integration | Build the Qt UI with the native core |
| Optional test audio | Generated PCM test tone first; add a decoder later | Keep the first milestone small and reproducible |

**Windows setup checklist**

- Windows 10/11 development machine.
- Visual Studio 2022 and its C++ workload, including a Windows SDK.
- CMake 3.24 or newer; Ninja is optional but useful.
- Qt 6 installed with the selected compiler kit and the chosen UI modules.
- Git.
- Two output endpoints for real-device tests (for example, wired headphones plus a Bluetooth speaker). A second physical device is not required for unit tests.

WASAPI is part of the Windows audio platform/SDK; it is not a separate pip/npm package. The MVP should use shared-mode endpoint rendering unless testing demonstrates a specific need otherwise.

### Android app (planned after the Windows MVP is proven)

| Area | Recommended technology | Purpose |
|---|---|---|
| Language | Kotlin | Android app and platform integration |
| UI | Jetpack Compose | Native Android interface |
| IDE/build | Android Studio, Android SDK, Gradle | Build, run, and package the app |
| Playback API | Android `AudioTrack` / suitable Android media APIs | Push PCM data to an Android-managed output route |
| Audio lifecycle | Android audio focus and routing APIs | Respond to focus changes, route changes, and interruptions |
| Shared native code (optional later) | Android NDK + CMake + JNI | Reuse portable synchronization/buffering code from C++ |

Android requirements: Android Studio, a compatible JDK (the Studio-managed JDK is fine), Android SDK platform/build tools, Gradle via the project wrapper once the Android project is created, and an Android phone/emulator. Actual Bluetooth multi-output behaviour depends on the Android version, vendor implementation, output-device capabilities, and available platform APIs. Do not assume an ordinary Android app can independently open every connected Bluetooth output.

### Shared audio-core dependencies

Start with the C++ standard library and platform APIs only. Add libraries only when a concrete requirement appears.

- **Required:** C++20 compiler, CMake, Git.
- **Windows-specific:** Windows SDK / WASAPI / COM APIs.
- **UI:** Qt 6 for the Windows UI; Android Compose is managed by its Gradle dependencies.
- **Optional later:** a tested audio decoder/resampler library if needed. Evaluate licensing, platform support, latency, and maintenance before selecting one.
- **Not required for the first MVP:** cloud server, database, account system, REST API, Electron, Docker, ESP32, or a custom Bluetooth driver.

Keep exact versions in the eventual build files and dependency lock/configuration files. This starter repository intentionally does not pin versions because no implementation/build project has been created yet.

## 3. Frontend-to-backend workflow

In PhaseLink, “backend” initially means the **local audio engine**. It does not mean a remote web server. The UI and engine should run on the same computer/phone; raw audio should not be sent to a cloud service.

### Planned data flow

```text
User
 |
 v
Frontend UI
  |  1. Request device list
  |  2. User selects source and output devices
  |  3. Send route configuration
  v
Audio-engine interface
  |  4. Validate selected endpoints and capabilities
  |  5. Open source and output streams
  |  6. Convert source audio to a common internal PCM format
  v
Shared audio pipeline
  |  7. Fan out audio to per-device queues
  |  8. Apply resampling/buffering and per-output delay
  |  9. Monitor errors, queue depth, and device state
  v
Platform audio backends
  |-------------------|-------------------|
  v                   v                   v
Wired endpoint     Bluetooth A         Bluetooth B
```

### Main frontend responsibilities

- Show available endpoints and their connected/available state.
- Let the user choose the current source and target outputs.
- Start/stop a route and display errors or device changes.
- Display sync/latency estimates only when they are actually measured; label estimates clearly.
- Send **commands and settings**, not audio buffers, through the UI boundary.

### Main backend responsibilities

- Discover endpoints and react to device notifications.
- Validate requested routes; report unsupported combinations instead of silently claiming success.
- Acquire audio from the selected source.
- Keep the audio callback/render path real-time-safe: avoid blocking locks, file/network I/O, and UI work in critical audio callbacks.
- Fan audio out to independent output queues.
- Handle format conversion, per-output buffering, delay compensation, underruns, and disconnections.
- Emit state/diagnostic events to the frontend.

### Planned command/event contract

Use a small typed interface rather than making UI components call platform APIs directly. An initial contract can include:

**Commands:** `listDevices`, `selectSource`, `setOutputs`, `startRoute`, `stopRoute`, `setOutputVolume`, `setOutputDelay`, `requestDiagnostics`.

**Events:** `devicesChanged`, `routeStateChanged`, `outputStateChanged`, `latencyEstimateUpdated`, `audioWarning`, `audioError`, `diagnosticsUpdated`.

**Device model fields:** stable ID, display name, input/output capability, connection state, supported formats (when available), platform/device type, and any latency estimate plus its confidence/source.

For Windows Qt, this boundary can initially be ordinary C++ interfaces and Qt signals/slots. On Android, the Compose UI should call an app-level controller/service; platform-specific audio APIs live behind that layer. If a C++ synchronization core is shared on Android later, expose only the necessary API via JNI. Avoid adding HTTP or WebSockets between the UI and local engine unless a separate-process design later makes it necessary.

## 4. Work split for two people

Recommended ownership for the **first Windows MVP**:

### Person A — Frontend and app integration

- Own `frontend/windows/` and UI layout.
- Build the main window, device list, output-selection controls, route status, settings, and diagnostics panel.
- Define how loading/error/unsupported states are shown.
- Use a mock device provider at first so UI progress does not depend on the real audio engine.
- Integrate with the agreed engine interface after the contract is stable.
- Keep UI code out of audio callbacks and low-level endpoint management.

### Person B — Backend and audio engine

- Own `backend/core/`, `backend/platform/windows/`, and relevant tests.
- Define device, source, output, route, error, and diagnostic models.
- Implement device discovery, one source to one output, then one source to two outputs.
- Add per-output queues and a simple configurable delay before attempting automatic calibration.
- Implement start/stop/disconnect/error handling and test-tone playback.
- Provide a mock or fake backend and a clear interface for Person A.

### Shared responsibilities

- Agree on the interface contract before parallel implementation.
- Both review changes to shared models/contracts.
- Pair on integration and physical-device tests.
- Track issues and write down reproduction steps for bugs.
- Do not both edit the same files at the same time unless coordinating a deliberate pair-programming session.

Android can be a later workstream after the Windows engine contract and limitations are understood. If both people want to work on Android early, Person A can create the Android screen skeleton while Person B keeps the shared engine API portable—but the primary milestone should remain Windows.

## 5. How to start and work in parallel

### Step 1 — Set up the repository

1. Extract the ZIP.
2. Open `PhaseLink-Starter/` in your editor.
3. Initialize a Git repository if this is not already inside one:

   ```bash
   git init
   git add .
   git commit -m "chore: create PhaseLink starter structure"
   ```

4. Create a remote repository and push the initial commit if you are collaborating through GitHub or a similar host.

There is no runnable program yet, so there is **no build/run command at this stage**. The folders are placeholders by request.

### Step 2 — Agree on the first milestone

The first goal should be deliberately small:

> A Windows desktop prototype plays a generated test tone through one selected output, then through two outputs, with independent playback queues and visible output status.

Do not start with arbitrary Spotify/YouTube interception, a virtual audio driver, Android multi-output, Bluetooth relaying, or a custom Bluetooth driver. Add system-wide routing only after the audio engine works for a controlled source.

### Step 3 — Create the interface contract first

Person B drafts the device/route models and engine API in `docs/interface-contract.md`. Person A creates a mock implementation that returns simulated endpoints and route states. Both review the contract before implementing against it.

The UI can then be built against the mock. The backend can be built and tested independently of the UI. Swap the mock for the real backend during integration.

### Step 4 — Use separate branches

Example branch names:

```text
main
feat/windows-ui
feat/audio-device-model
feat/windows-endpoint-discovery
feat/multi-output-rendering
```

Keep pull requests small. Avoid committing generated build folders, IDE caches, or local device-specific settings.

### Step 5 — Integrate in increments

1. Mock device list appears in the UI.
2. Real Windows endpoint discovery replaces the mock.
3. Test tone plays through one output.
4. Two output streams play independently.
5. Add configurable per-output delay and verify behaviour using repeatable tests.
6. Handle hot-unplug, reconnection, unsupported formats, and underruns.
7. Test with wired + Bluetooth outputs and document the observed relative delay.
8. Only then investigate automatic calibration and system-wide audio capture/routing.

### Step 6 — Minimum done definition

The first milestone is done when a user can select two endpoints, start/stop the test source, see each endpoint's state, receive a useful error if one disappears, and compare whether playback is acceptably synchronized. Do not claim “perfect sync” until it is measured over multiple devices and sessions.

## 6. Scope, constraints, and milestones

### Suggested milestones

- **M0 — Repository and contracts:** folder structure, architecture notes, interface/model definitions.
- **M1 — UI mock:** window and simulated device states; no real audio yet.
- **M2 — One output:** generated tone to one Windows output.
- **M3 — Multi-output:** one source rendered to two available endpoints.
- **M4 — Manual sync controls:** per-output delay, mute/volume, diagnostics.
- **M5 — Resilience:** hotplug, reconnection, format mismatch, underruns, logs.
- **M6 — Calibration research:** estimate relative latency and adapt delays; document measurement limitations.
- **M7 — System-wide audio:** investigate a virtual endpoint/driver or supported routing approach. This is not part of M1–M5.
- **M8 — Android:** native Android prototype with supported routing; assess actual device/API limits before promising multi-output.
- **M9 — SYNCOS integration:** only after standalone PhaseLink is stable.

### Important technical limitations

- Bluetooth output latency varies by device, codec, buffer, connection, and operating system.
- Matching timestamps in the app does not guarantee matching sound at the physical transducers; end-to-end calibration is needed.
- Multiple independent Bluetooth A2DP outputs are not guaranteed to be exposed to ordinary applications as simultaneously writable endpoints.
- System-wide audio capture/routing is different from playing a file/test tone inside PhaseLink.
- Protected audio, exclusive-mode applications, calls, and OS policy may prevent capture or rerouting.
- Bluetooth LE Audio/Auracast only helps when compatible source, receiver, OS, and profile support align; it is not a universal relay for ordinary Bluetooth devices.
- Android's app-level playback APIs do not grant arbitrary control over every connected audio route.

Treat these as design constraints to test, not features to assume.

## 7. Repository structure

```text
PhaseLink-Starter/
├── README.md
├── .gitignore
├── frontend/
│   ├── README.md
│   ├── windows/                # Planned Qt desktop UI (empty)
│   └── android/                # Planned Kotlin/Compose app (empty)
├── backend/
│   ├── README.md
│   ├── core/
│   │   ├── include/phaselink/   # Planned public engine/model interfaces (empty)
│   │   └── src/                 # Planned portable C++ audio core (empty)
│   └── platform/
│       ├── windows/             # Planned WASAPI/Windows endpoint implementation (empty)
│       └── android/             # Planned Android audio adapter (empty)
├── docs/
│   ├── architecture.md
│   └── interface-contract.md
├── tests/
│   ├── unit/
│   ├── integration/
│   └── manual/
├── scripts/                     # Planned helper/build scripts (empty)
└── assets/                      # Planned icons/mockups/test assets (empty)
```

The `.gitkeep` files only preserve empty directories in Git. They are not application code.

## 8. Useful official documentation

- Microsoft WASAPI / Windows audio: <https://learn.microsoft.com/en-us/windows/win32/coreaudio/wasapi>
- Microsoft Core Audio interfaces and endpoint notifications: <https://learn.microsoft.com/en-us/windows/win32/coreaudio/core-audio-interfaces>
- Qt 6 with CMake: <https://doc.qt.io/qt-6/cmake-get-started.html>
- Android `AudioTrack`: <https://developer.android.com/reference/android/media/AudioTrack>
- Jetpack Compose setup: <https://developer.android.com/develop/ui/compose/setup>
- Android NDK overview: <https://developer.android.com/ndk/guides>

---

**Project rule:** build PhaseLink as a standalone, measurable audio-routing prototype first. Integrate it into SYNCOS only after its core behaviour is reliable.
