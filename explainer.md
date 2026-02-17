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
4. **Advance one frame**, then assert on data returned by WebXR (poses, events, hit test results etc.).
5. Optionally **connect fake input sources** and simulate input sequences (select lifecycle, button state changes etc.).

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

## Hit Test Extension

In order to create deterministic and cross-browser WPT tests for the proposed WebXR [hit testing API](https://github.com/immersive-web/hit-test/), the WPT tests need to have a way to mock the data that is supposed to be returned from the API under test. This can be achieved by leveraging the test API extensions for hit test, described below.

```webidl
partial interface FakeXRDevice {
  // Sets new world state on the device.
  undefined setWorld(FakeXRWorldInit world);
  // Clears the entire knowledge of the world on the device.
  undefined clearWorld();
};

partial dictionary FakeXRDeviceInit {
  // Initial state of the world known to the device.
  FakeXRWorldInit worldInit;
};

dictionary FakeXRWorldInit {
  // World consists of a collection of hit testing regions.
  // The regions are listed in no particular order.
  required sequence<FakeXRRegionInit> hitTestRegions;
};

dictionary FakeXRRegionInit {
  // Collection of faces that comprise this region.
  required sequence<FakeXRTriangleInit> faces;
  // Type of the region. This will be considered when computing hit test results
  // for the purpose of filtering out the ones that the applicaton is not interested in.
  // More details can be found in Hit Testing Explainer, Limiting results to specific entities section.
  required FakeXRRegionType type;
};

dictionary FakeXRTriangleInit {
  // Sequence of vertices that comprise this triangle.
  // The triangle is considered to be a solid surface for the purposes of hit test computations.
  required sequence<DOMPointReadOnly> vertices;  // size = 3
};

enum FakeXRRegionType {
  "point",
  "plane",
  "mesh"
};

```

## DOM Overlay Extension

In order to create deterministic and cross-browser WPT tests for the proposed WebXR [DOM Overlay API](https://immersive-web.github.io/dom-overlays/), the WPT tests need to have a way to supply data for API interactions. This can be achieved by leveraging the test API extensions for DOM Overlay support, described below.

```webidl
partial interface FakeXRInputController {
  // Sets the position within the DOM Overlay in DOM coordinates for the next controller action.
  undefined setOverlayPointerPosition(float x, float y);
};
```
