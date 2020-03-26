# WebXR Test API

In order to allow javascript tests for WebXR there are some basic functions which are common across all tests,
such as adding a fake test device and specifying poses. Below is a Javascript IDL which attempts to capture
the necessary functions, based off what was defined in the spec. Different browser vendors can implement this
Javascript IDL in whatever way is most compatible with their browser. For example, some browsers may back the
interface with a WebDriver API while others may use HTTP or IPC mechanisms to communicate with an out of process
fake backend. Because of this, any "synchronous" methods that update the state of a device or controller are not
guaranteed to have that updated state respected until the next "requestAnimationFrame" returns.

```WebIDL
partial interface XRSystem {
    [SameObject] readonly attribute XRTest test;
};

interface XRTest {
  // Simulates connecting a device to the system.
  // Used to instantiate a fake device for use in tests.
  Promise<FakeXRDevice> simulateDeviceConnection(FakeXRDeviceInit init);

  // Simulates a user activation (aka user gesture) for the current scope.
  // The activation is only guaranteed to be valid in the provided function and only applies to WebXR
  // Device API methods.
  void simulateUserActivation(Function f);

  // Disconnect all fake devices
  Promise<void> disconnectAllDevices();
};
```

The promise returned from simulateDeviceConnection resolves with a FakeXRDevice, which can be used
to control the fake XRDevice that has been created in the background. The fake device may be used in a session returned by
navigator.xr.requestSession(), depending on how many devices have been created and how the browser decides to hand
them out.

```WebIDL
dictionary FakeXRDeviceInit {
    // Deprecated - use `supportedModes` instead.
    required boolean supportsImmersive;
    // Sequence of modes that should be supported by this device.
    sequence<XRSessionMode> supportedModes;
    required sequence<FakeXRViewInit> views;

    // https://immersive-web.github.io/webxr/#feature-name
    // The list of feature names that this device supports.
    // Any requests for features not in this list should be rejected, with the exception of those
    // that are guaranteed regardless of device availability (e.g. 'viewer').
    // If not specified/empty, the device supports no features.
    // NOTE: This is meant to emulate hardware support, not whether a feature is
    // currently available (e.g. bounds not being tracked per below)
    sequence<DOMString> supportedFeatures;

    // The bounds coordinates. If empty, no bounded reference space is currently tracked.
    // If not, must have at least three elements.
    sequence<FakeXRBoundsPoint> boundsCoodinates;

    // A transform used to identify the physical position of the user's floor.
    // If not set, indicates that the device cannot identify the physical floor.
    FakeXRRigidTransformInit floorOrigin;

    // native origin of the viewer
    // If not set, the device is currently assumed to not be tracking, and xrFrame.getViewerPose should
    // not return a pose.
    //
    // This sets the viewer origin *shortly after* initialization; since the viewer origin at initialization
    // is used to provide a reference origin for all matrices.
    FakeXRRigidTransformInit viewerOrigin;
};

interface FakeXRDevice {
  // Sets the values to be used for subsequent
  // requestAnimationFrame() callbacks.
  void setViews(sequence<FakeXRViewInit> views);

  // behaves as if device was disconnected
  Promise<void> disconnect();

  // Sets the origin of the viewer
  void setViewerOrigin(FakeXRRigidTransformInit origin, optional boolean emulatedPosition = false);

  // If an origin is not specified, then the device is assumed to not be tracking, emulatedPosition should
  // be assumed for cases where the UA must always provide a pose.
  void clearViewerOrigin();

  // Simulates devices focusing and blurring sessions.
  void simulateVisibilityChange(XRVisibilityState visibilityState);

  void setBoundsGeometry(sequence<FakeXRBoundsPoint> boundsCoodinates);
  // Sets the native origin of the physical floor
  void setFloorOrigin(FakeXRRigidTransformInit floorOrigin);

  // Indicates that the device can no longer identify the location of the physical floor.
  void clearFloorOrigin();

  // Used to simulate a major change in tracking and that a reset pose event should be fired
  // https://immersive-web.github.io/webxr/#event-types
  void simulateResetPose();

  // Used to connect and send input events
  FakeXRInputController simulateInputSourceConnection(FakeXRInputSourceInit inputSource);
};

// https://immersive-web.github.io/webxr/#xrview
dictionary FakeXRViewInit {
  required XREye eye;
  // https://immersive-web.github.io/webxr/#view-projection-matrix
  required sequence<float> projectionMatrix;
  // https://immersive-web.github.io/webxr/#dom-xrwebgllayer-getviewport
  required FakeXRDeviceResolution resolution;
  // https://immersive-web.github.io/webxr/#view-offset
  // This is the origin of the view in the viewer space. In other words, this is
  // a transform from the view space to the viewer space.
  required FakeXRRigidTransformInit viewOffset;
  // This is an optional means of specifying a decomposed form of the projection
  // matrix.  If specified, the projectionMatrix should be ignored.
  // Any test that wishes to test clip planes or similar features that would require
  // decomposing/recomposing the projectionMatrix should use this instead of
  // the projection matrix.
  FakeXRFieldOfViewInit fieldOfView;
};

// A set of 4 angles which describe the view from a center point, units are degrees.
dictionary FakeXRFieldOfViewInit {
  required float upDegrees;
  required float downDegrees;
  required float leftDegrees;
  required float rightDegrees;
};

// This represents the native resolution of the device, but may not reflect the viewport exposed to the page.
// https://immersive-web.github.io/webxr/#xrviewport
dictionary FakeXRDeviceResolution {
    required long width;
    required long height;
};

dictionary FakeXRBoundsPoint {
  double x; double z;
};


// https://immersive-web.github.io/webxr/#xrrigidtransform
dictionary FakeXRRigidTransformInit {
  // must have three elements
  required sequence<float> position;
  // must have four elements
  required sequence<float> orientation;
};
```


The WebXR API never exposes native origins directly, instead exposing transforms between them, so we need to specify a base reference space for XRRigidTransformInit so that we can have consistent numerical values across implementations. When used as an origin, XRRigidTransformInits are in the base reference space where the viewer's native origin is identity at initialization, unless otherwise specified. In this space, the `local` reference space has a native origin of identity. This is an arbitrary choice: changing this reference space doesn't affect the data returned by the WebXR API, but we must make such a choice so that the tests produce the same results across different UAs. When used as an origin it is logically a transform _from_ the origin's space _to_ the underlying base reference space described above.

For many UAs input is sent on a per-frame basis, therefore input events are not guaranteed to fire and the FakeXRInputController
is not guaranteed to be present in session.inputSources until after one animation frame.

``` WebIDL
dictionary FakeXRInputSourceInit {
  required XRHandedness handedness;
  required XRTargetRayMode targetRayMode;
  required FakeXRRigidTransformInit pointerOrigin;
  required sequence<DOMString> profiles;
  // was the primary action pressed when this was connected?
  boolean selectionStarted = false;
  // should this input source send a select immediately upon connection?
  boolean selectionClicked = false;
  // Initial button state for any buttons beyond the primary that are supported.
  // If empty, only the primary button is supported.
  // Note that if any FakeXRButtonType is repeated the behavior is undefined.
  sequence<FakeXRButtonStateInit> supportedButtons;
  // If not set the controller is assumed to not be tracked.
  FakeXRRigidTransformInit gripOrigin;
};

interface FakeXRInputController {

  // Indicates that the handedness of the device has changed.
  void setHandedness(XRHandedness handedness);

  // Indicates that the target ray mode of the device has changed.
  void setTargetRayMode(XRTargetRayMode targetRayMode);

  // Indicates that the list of profiles representing the device has changed.
  void setProfiles(sequence<DOMString> profiles);

  // Sets or clears the position of the controller.  If not set, the controller is assumed to
  // not be tracked.
  void setGripOrigin(FakeXRRigidTransformInit gripOrigin, optional boolean emulatedPosition = false);
  void clearGripOrigin();

  // Sets the pointer origin for the controller.
  void setPointerOrigin(FakeXRRigidTransformInit pointerOrigin, optional boolean emulatedPosition = false);

  // Temporarily disconnect the input device
  void disconnect();

  // Reconnect a disconnected input device
  void reconnect();

  // Start a selection for the current frame with the primary input
  // If a gamepad is supported, should update the state of the primary button accordingly.
  void startSelection();

  // End selection for the current frame with the primary input
  // If a gamepad is supported, should update the state of the primary button accordingly.
  void endSelection();

  // Simulates a start/endSelection for the current frame with the primary input
  // If a gamepad is supported, should update the state of the primary button accordingly.
  void simulateSelect();

  // Updates the set of supported buttons, including any initial state.
  // Note that this method should not be generally used to update the state of the
  // buttons, as the UA may treat this as re-creating the Gamepad.
  // Note that if any FakeXRButtonType is repeated the behavior is undefined.
  void setSupportedButtons(sequence<FakeXRButtonStateInit> supportedButtons);

  // Used to update the state of a button currently supported by the input source
  // Will not add support for that button if it is not currently supported.
  void updateButtonState(FakeXRButtonStateInit buttonState);
};

// Bcause the primary button is always guaranteed to be present, and other buttons
// should fulfill the role of validating any state from FakeXRButtonStateInit
// the primary button is not present in this enum.
enum FakeXRButtonType {
  "grip",
  "touchpad",
  "thumbstick",
  // Represents a button whose position is not specified by the xr-standard mapping.
  // Should appear at one past the last reserved button index.
  "optional-button",
  // Represents a thumbstick whose position is not specified by the xr-standard mapping.
  // Should appear at two past the last reserved button index.
  "optional-thumbstick"
};

// Used to update the state of optionally supported buttons.
dictionary FakeXRButtonStateInit {
  required FakeXRButtonType buttonType;
  required boolean pressed;
  required boolean touched;
  required float pressedValue;
  // x and y value are ignored if the FakeXRButtonType is not touchpad, thumbstick, or optional-thumbstick
  float xValue = 0.0;
  float yValue = 0.0;
};
```

These initialization object and control interfaces do not represent a complete set of WebXR functionality,
and are expected to be expanded on as the WebXR spec grows.

## Hit Test Extension

In order to create deterministic and cross-browser WPT tests for the proposed WebXR [hit testing API](https://github.com/immersive-web/hit-test/), the WPT tests need to have a way to mock the data that is supposed to be returned from the API under test. This can be achieved by leveraging the test API extensions for hit test, described below.

```webidl
partial interface FakeXRDevice {
  // Sets new world state on the device.
  void setWorld(FakeXRWorldInit world);
  // Clears the entire knowledge of the world on the device.
  void clearWorld();
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
  void setOverlayPointerPosition(float x, float y);
};
```
