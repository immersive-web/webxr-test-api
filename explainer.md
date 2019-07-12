In order to allow javascript tests for WebXR there are some basic functions which are common across all tests,
such as adding a fake test device and specifying poses. Below is a Javascript IDL which attempts to capture
the necessary functions, based off what was defined in the spec. Different browser vendors can implement this
Javascript IDL in whatever way is most compatible with their browser. For example, some browsers may back the
interface with a WebDriver API while others may use HTTP or IPC mechanisms to communicate with an out of process
fake backend. Because of this, any "synchronous" methods that update the state of a device or controller are not
guaranteed to have that updated state respected until the next "requestAnimationFrame" returns.

```WebIDL
partial interface XR {
    [SameObject] readonly attribute XRTest test;
};

interface XRTest {
  // Simulates connecting a device to the system.
  // Used to instantiate a fake device for use in tests.
  Promise<FakeXRDevice> simulateDeviceConnection(FakeXRDeviceInit init);

  // Simulates a user activation (aka user gesture) for the current scope.
  // The activation is only guaranteed to be valid in the provided function and only applies to WebXR
  // Device API methods.
  void simulateUserActivation(Function);

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
    required boolean supportsImmersive;
    required sequence<FakeXRViewInit> views;

    boolean supportsUnbounded = false;
    // The bounds coordinates. If null/empty, bounded reference spaces are not supported. If not, must have at least three elements.
    sequence<FakeXRBoundsPoint> boundsCoodinates;
    // A transform used to identify the physical position of the user's floor.  If not set, indicates that the device cannot identify the physical floor.
    FakeXRRigidTransformInit floorOrigin;
    // native origin of the viewer
    // If not set, the device is currently assumed to not be tracking, and xrFrame.getViewerPose should
    // not return a pose.
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
  void clearViewerOrigin()

  // Simulates devices focusing and blurring sessions.
  void simulateVisibilityChange(XRVisibilityState);

  void setBoundsGeometry(sequence<FakeXRBoundsPoint> boundsCoodinates);
  // Sets the native origin of the physical floor
  void setFloorOrigin(FakeXRRigidTransformInit floorOrigin);

  // Indicates that the device can no longer identify the location of the physical floor.
  void clearFloorOrigin();

  // Used to simulate a major change in tracking and that a reset pose event should be fired
  // https://immersive-web.github.io/webxr/#event-types
  void simulateResetPose();

  // Used to connect and send input events
  FakeXRInputController simulateInputSourceConnection(FakeXRInputSourceInit);
};

// https://immersive-web.github.io/webxr/#xrview
dictionary FakeXRViewInit {
  required XREye eye;
  // https://immersive-web.github.io/webxr/#view-projection-matrix
  required sequence<float> projectionMatrix;
  // https://immersive-web.github.io/webxr/#dom-xrwebgllayer-getviewport
  required FakeXRDeviceResolution resolution;
  // https://immersive-web.github.io/webxr/#view-offset
  required FakeXRRigidTransformInit viewOffset;
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

// When used as a native origin, it is in the reference space
// where the viewer's native origin is identity at initialization
//
// https://immersive-web.github.io/webxr/#xrrigidtransform
dictionary FakeXRRigidTransformInit {
  // must have three elements
  required sequence<float> position;
  // must have four elements
  required sequence<float> orientation;
};
```

For many UAs input is sent on a per-frame basis, therefore input events are not guaranteed to fire and the FakeXRInputController
is not guaranteed to be present in session.inputSources until after one animation frame.

``` WebIDL
interface FakeXRInputSourceInit {
  required XRHandedness handedness;
  required XRTargetRayMode targetRayMode;
  required FakeXRRigidTransformInit pointerOrigin;
  required sequence<DOMString> profiles;
  // was the primary action pressed when this was connected?
  bool selectionStarted = false;
  // should this input source send a select immediately upon connection?
  bool selectionClicked = false;
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
  void startSelection();

  // End selection for the current frame with the primary input
  void endSelection();

  // Simulates a start/endSelection for the current frame with the primary input
  void simulateSelect();
};
```

These initialization object and control interfaces do not represent a complete set of WebXR functionality,
and are expected to be expanded on as the WebXR spec grows.
