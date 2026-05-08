# WebXR Test API

## Introduction

In order to provide end-users with robust and consistent immersive experiences, features of the spec need to be implemented consistently across devices and platforms. Additionally, content authors need to be able to depend on up-to-date feature support data in order to make implementation decisions.

WebXR features are tightly coupled to hardware, platform runtimes and real-time sensor input which makes automated testing difficult. Without a predictable XR device, it is hard to write [Web Platform Tests (WPTs)](https://web-platform-tests.org/) that are deterministic, cross-browser, and runnable on CI. The WebXR Test API addresses these challenges by providing a testing-only surface that allows tests to simulate XR devices, poses and input sources in a controlled way.

This API is intended solely for use in browser test environments (e.g. WPT harnesses and internal test runners) and is not designed to be exposed to web content. Different user agents may implement the same test surface using whatever mechanisms best fit their architecture (e.g. internal mocks, WebDriver, IPC) to communicate with a fake backend.

## Challenges

WebXR behaviour depends on physical devices, real-time tracking, platform runtimes and per-frame input delivery. This makes it difficult to write WPTs that are reliable across browsers and runnable in continuous integration environments where XR hardware may not be present. Today, WebXR tests frequently need to:
- Run without access to real/physical XR hardware (e.g. on bots/CI).
- Control device characteristics (views, supported session modes, supported features).
- Drive deterministic tracking states (viewer pose, bounded-floor availability, tracking loss).
- Simulate input sources and user actions (controllers, select sequences, visibility changes).
- Assert outcomes consistently across different browsers.

In order to allow JavaScript tests for WebXR, there are some basic functions which are common across all tests such as adding a fake test device and specifying poses. This API attempts to capture the necessary functions, based off what is defined in the [specification](https://immersive-web.github.io/webxr-test-api/).

## Goals

- Enable deterministic, cross-browser automated testing of WebXR Device API behaviour.
- Allow tests to simulate XR device availability and capabilities (e.g. session modes, views, supported features).
- Allow tests to control per-frame tracking state and poses in a reproducible way.
- Allow tests to simulate input sources and input-driven events (e.g. select lifecycle) in a controlled manner.
- Support extension testing patterns (e.g. hit-test, anchors, light estimation, DOM overlay) by providing test hooks that let tests supply deterministic data.

## Non-goals

- This is not a production web API and is not intended to be exposed to normal web content.
- This is not a fully-featured XR emulator for application developers; it exists to mainly support automated testing of the WebXR Device API.
- This does not aim to perfectly model real-world sensor noise, tracking quality or runtime-specific behaviour beyond what is needed for conformance tests.
- This does not require or endorse a specific implementation strategy (e.g. WebDriver vs in-process mocking); UAs may differ in how tests are implemented.
- This does not guarantee synchronous visibility of state updates; tests should assume state may only be reflected on the next WebXR frame.

## Proposed Approach

The WebXR Test API is designed around the way the WebXR Device API itself operates: **state is consumed on a per-frame basis**. Tests **set up** fake devices and inputs, **state is updated** and then **results are observed on the next WebXR frame**. This keeps tests deterministic while allowing user agents to implement the backing mechanism using whatever is most compatible with their architecture.

This explainer focuses on *how tests use the API*. The normative definition of interfaces and behaviour lives in the [specification](https://immersive-web.github.io/webxr-test-api/).

### The basic testing flow

Most WebXR tests follow the same pattern:
1. **Connect a fake device** with the desired capabilities (session types, views, supported features, etc.).
2. **Use WebXR entry points** to obtain a session (e.g. `navigator.xr.requestSession(. . .)`).
3. **Drive device state** (viewer pose, tracking loss, bounds/floor origin, visibility state etc.).
4. **Advance one frame**, then assert on data returned by WebXR (poses, events, hit test results etc.). _Note: While most updates are reflected in the next animation frame, some state changes (especially those made outside of an active XR animation frame) may require waiting for up to two frames to be guaranteed as [highlighted in the spec](https://immersive-web.github.io/webxr-test-api/#xrsession-next-animation-frame). If assertions fail unexpectedly, try waiting an additional frame._
6. Optionally **connect fake input sources** and simulate input sequences (select lifecycle, button state changes etc.).

### Example: connect a fake device and assert the viewer's pose

```js
// 1. Create a fake device (values shown are illustrative).
const device = await navigator.xr.test.simulateDeviceConnection({
  supportedModes: ["immersive-vr"],
  views: [{
    eye: "none",
    projectionMatrix: [/* tests should provide a valid 4x4 matrix transformation */],
    resolution: { width: 2000, height: 2000 },
    viewOffset: { position: [0, 0, 0], orientation: [0, 0, 0, 1] },
  }],
  supportedFeatures: ["local-floor"],
});

// 2. Request a WebXR session using WebXR APIs.
const session = await navigator.xr.requestSession("immersive-vr");
const refSpace = await session.requestReferenceSpace("local");

// 3. Update the simulated tracking state (viewer origin in this case).
device.setViewerOrigin({
  position: [0, 1.5, 0],
  orientation: [0, 0, 0, 1],
});

// 4. Observe the effect on the next frame.
await new Promise(resolve => {
  session.requestAnimationFrame((_t, frame) => {
    const pose = frame.getViewerPose(refSpace);
    // assert expected pose properties here (exact assertions depend on the test harness used).
    resolve();
  });
});
```

***Note:** user agents are not required to apply state updates synchronously. Tests should assume that updates are reliably visible by the next XR animation frame.*

### Example: simulate tracking loss and recovery

Tests may need to validate behaviour when tracking is lost (i.e. `getViewerPose` returns `null`) and later restored.

```js
/* Create a fake device and request a WebXR session as above. */

// Simulate tracking loss.
device.clearViewerOrigin();

await new Promise(resolve => {
  session.requestAnimationFrame((_t, frame) => {
    const pose = frame.getViewerPose(refSpace);
    // expect pose to be null while not tracking.
    resolve();
  });
});

// Restore tracking.
device.setViewerOrigin({
  position: [0, 1.6, 0],
  orientation: [0, 0, 0, 1],
});
```

### Example: simulate an input source and a select action

Input is typically delivered per-frame, so tests should wait at least one frame after connecting an input source before expecting it to appear in `session.inputSources` or for events to fire.

```js
/* Create a fake device and request a WebXR session as above. */

// Simulate an input source (values shown are illustrative).
const controller = device.simulateInputSourceConnection({
  handedness: "right",
  targetRayMode: "tracked-pointer",
  profiles: ["generic-trigger"],
  pointerOrigin: { position: [0.2, 1.3, -0.4], orientation: [0, 0, 0, 1] },
  gripOrigin: { position: [0.2, 1.3, -0.4], orientation: [0, 0, 0, 1] },
});

// Wait a frame for the input source to become visible to the session.
await new Promise(resolve => session.requestAnimationFrame(() => resolve()));

// Listen for select events via normal WebXR events.
let sawSelect = false;
session.addEventListener("select", () => { sawSelect = true; });

// Drive input for the next frame.
controller.simulateSelect();

// Observe results on the next frame via events.
await new Promise(resolve => session.requestAnimationFrame(() => resolve()));
// Assert: sawSelect === true
```

## Extension Test Hooks (Overview)

Many WebXR modules rely on *real-world* data (i.e. geometry, environment understanding, interaction) which must be controllable and repeatable for conformance testing. This Test API takes into account this requirement and provides extension-specific hooks that let tests provide **deterministic inputs** for these modules while still exercising entry points to the **real WebXR-facing APIs** under test. This approach supports WPT-style testing and ensures tests are reliable and effective. For example:
- Hit Test Extension: tests can define a deterministic “world” (planes/meshes/points) for hit testing.
- DOM Overlay Extension: tests can define pointer positions within the overlay prior to simulating input.

The extension-specific usage patterns follow the same testing flow: configure deterministic inputs, then advance a frame and finally assert on the WebXR API results.

***Note:** The WebXR Test API is intended to enable testing of WebXR Device API and related modules, including features that may still be unstable or/and in development. As a result, the test API surface is expected to grow alongside WebXR modules that require deterministic test control.*

## Extension Test Hooks (Examples)

The examples below follow the same pattern as the core API:
1. Configure deterministic test data via the test hook.
2. Use WebXR entry points to call the WebXR module API under test.
3. Advance a frame (or await relevant promise).
4. Assert on results.

### Hit Test Extension

The WebXR Hit Test API computes intersections with real-world geometry. For testing, the device’s “real world knowledge” can be supplied explicitly so that hit test results are predictable across user agents.

Typical test flow:
- Connect a fake device with `hit-test` listed as a supported feature.
- Define a synthetic world (regions and faces).
- Request a hit test source via the real API.
- Run hit tests and assert on returned results.

```js
const device = await navigator.xr.test.simulateDeviceConnection({
  supportedModes: ["immersive-ar"],
  views: [/* Setup desired device properties as above */],
  supportedFeatures: ["hit-test"],
});

const session = await navigator.xr.requestSession("immersive-ar", {
  requiredFeatures: ["hit-test"],
});

// Provide deterministic world geometry.
device.setWorld({
  hitTestRegions: [{
    type: "plane",
    faces: [{
      vertices: [
        new DOMPointReadOnly(-1, 0, -1, 1),
        new DOMPointReadOnly( 1, 0, -1, 1),
        new DOMPointReadOnly(-1, 0,  1, 1),
      ],
    }],
  }],
});

// Use WebXR entry points to request a hit test source.
const viewerSpace = await session.requestReferenceSpace("viewer");
const hitTestSource = await session.requestHitTestSource({ space: viewerSpace });

// Advance a frame and perform assertions.
await new Promise(resolve => {
  session.requestAnimationFrame((_t, frame) => {
    const results = frame.getHitTestResults(hitTestSource);
    // assert that results are present and have expected transforms.
    resolve();
  });
});
```
Notes for test authors:
- Tests should construct geometry that is simple and unambiguous to keep assertions stable (e.g. a single plane).
- World updates should be assumed to take effect by the next XR animation frame.

### DOM Overlay Extension
The DOM Overlay API enables user interaction with DOM elements while in an immersive session. Tests may need to deterministically control the overlay pointer position before simulating input actions.

Typical test flow:
- Connect a fake device with `dom-overlay` listed as a supported feature.
- Request an immersive session with DOM overlay enabled.
- Set the overlay pointer coordinates on the fake input controller.
- Simulate an input action and assert on target/event behaviour.

```js
const device = await navigator.xr.test.simulateDeviceConnection({
  supportedModes: ["immersive-ar"],
  views: [/* Setup desired device properties as above */],
  supportedFeatures: ["dom-overlay"],
});

const overlayRoot = document.createElement("div");
overlayRoot.id = "overlay";
document.body.appendChild(overlayRoot);

const session = await navigator.xr.requestSession("immersive-ar", {
  requiredFeatures: ["dom-overlay"],
  domOverlay: { root: overlayRoot },
});

const controller = device.simulateInputSourceConnection({
  handedness: "right",
  targetRayMode: "tracked-pointer",
  profiles: ["generic-trigger"],
  pointerOrigin: { position: [0, 1.5, -0.5], orientation: [0, 0, 0, 1] },
  gripOrigin: { position: [0, 1.5, -0.5], orientation: [0, 0, 0, 1] },
});

// Example overlay target (a button inside the overlay).
const button = document.createElement("button");
button.textContent = "Test";
overlayRoot.appendChild(button);

let sawClick = false;
button.addEventListener("click", () => { sawClick = true; });

// Set deterministic overlay pointer position *before* simulating input.
controller.setOverlayPointerPosition(10, 10); // DOM overlay coordinates

// Wait a frame to ensure the input source is surfaced.
await new Promise(resolve => session.requestAnimationFrame(() => resolve()));

// Simulate input (selection) and assert expected DOM interaction.
controller.simulateSelect();
await new Promise(resolve => session.requestAnimationFrame(() => resolve()));
// Assert: sawClick === true
```

Notes for test authors:
- Use stable, layout-independent overlay geometry where possible (e.g. fixed-size root, explicit coordinates).
- Overlay pointer position is applied for the next controller action; tests should set it immediately before triggering input.

## Considered alternatives

### Using WebDriver or WebDriver BiDi for XR test control

One option would be to use WebDriver or WebDriver BiDi as the main mechanism for controlling XR state during tests. This has some appeal, particularly because WebDriver is already used for browser automation and cross-browser testing in many other contexts. A WebDriver-based solution could potentially reduce the need for dedicated XR testing hooks exposed through this API. However, WebXR testing often requires precise and deterministic control over XR-specific state in ways that map more directly to a purpose-built testing model. For example, tests may need to control simulated device capabilities or provide deterministic behaviour for extension-specific features such as hit-test, plane detection or DOM overlay. 

A WebDriver-based approach may still be useful in some contexts, and user agents may choose to use WebDriver internally as part of their implementation. However, this does not remove the need for a clear and interoperable abstraction for XR-specific testing behaviour. The WebXR Test API is intended to provide that shared testing surface while still allowing user agents to implement the underlying mechanism in whatever way best fits their architecture.
