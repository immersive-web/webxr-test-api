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
    // Whether the space supports tracking in inline sessions
    boolean supportsTrackingInInline = true;
    // The bounds coordinates. If null/empty, bounded reference spaces are not supported. If not, must have at least three elements.
    sequence<FakeXRBoundsPoint> boundsCoodinates;
    // Eye level used for calculating floor-level spaces
    float eyeLevel = 1.5;
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
  // Sets eye level used for calculating floor-level spaces
  void setEyeLevel(float eyeLevel);


  Promise<FakeXRInputController>
      simulateInputSourceConnection(FakeXRInputSourceInit);
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

interface FakeXRInputSourceInit {
  required XRHandedness handedness;
  required XRTargetRayMode targetRayMode;
  required FakeXRRigidTransformInit pointerOrigin;
  // was the primary action pressed when this was connected?
  bool selectionStarted = false;
  FakeXRRigidTransformInit gripOrigin;
};

interface FakeXRInputController {
  void setOrigins(
    boolean emulatedPosition,
    FakeXRRigidTransformInit pointerOrigin,
    FakeXRRigidTransformInit? gripOrigin);

  // Temporarily disconnect the input device
  Promise<void> disconnect();

  // Reconnect a disconnected input device
  Promise<void> reconnect();

  // Start a selection for the current frame with the given button index
  void startSelection();

  // End selection for the current frame with the given button index
  void endSelection();
};
```

These initialization object and control interfaces do not represent a complete set of WebXR functionality,
and are expected to be expanded on as the WebXR spec grows.
