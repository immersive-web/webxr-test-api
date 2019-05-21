In order to allow javascript tests for WebXR there are some basic functions which are common across all tests, 
such as adding a fake test device and specifying poses. Below is a Javascript IDL which attempts to capture 
the necessary functions, based off what was defined in the spec. Different browser vendors can implement this
Javascript IDL in whatever way is most compatible with their browser. For example, some browsers may back the
interface with a WebDriver API while others may use HTTP or IPC mechanisms to communicate with an out of process 
fake backend.

```WebIDL
partial interface XR {
    [SameObject] readonly attribute XRTest test;
};

interface XRTest {
  // Simulates connecting a device to the system.
  // Used to instantiate a fake device for use in tests.
  Promise<FakeXRDevice> simulateDeviceConnection(FakeXRDeviceInit);

  // Simulates a user activation (aka user gesture) for the current scope.
  // The activation is only guaranteed to be valid in the provided function and only applies to WebXR
  // Device API methods.
  void simulateUserActivation(Function);
};
```

The promise returned from simulateDeviceConnection resolves with a FakeXRDevice, which can be used 
to control the fake XRDevice that has been created in the background. The fake device may be used in a session returned by 
navigator.xr.requestSession(), depending on how many devices have been created and how the browser decides to hand 
them out.

```WebIDL
dictionary FakeXRDeviceInit {
    required boolean supportsImmersive;
    required Array<FakeXRViewInit> views;

    boolean supportsBounded = true;
    boolean supportsUnbounded = true;
    // Whether the space supports tracking in inline sessions
    boolean supportsTrackingInInline = true;
    Array<FakeXRBoundsPoint>? boundsCoodinates = null;
    // Eye level used for calculating floor-level spaces
    float eyeLevel = 1.5;
}

interface FakeXRDevice {
  // Sets the values to be used for subsequent
  // requestAnimationFrame() callbacks.
  void setViews(Array<FakeXRViewInit> views);

  // behaves as if device was disconnected
  void disconnect();

  // Sets the origin of the viewer
  void setViewerOrigin(FakeXRRigidTransformInit origin, boolean emulatedPosition = false);

  // Simulates devices focusing and blurring sessions.
  void simulateVisibilityChange(XRVisibilityState);

  void setBoundsGeometry(Array<FakeXRBoundsPoint> boundsCoodinates);
  // Sets eye level used for calculating floor-level spaces
  void setEyeLevel(float eyeLevel);

  
  Promise<FakeXRInputController>  
      simulateInputSourceConnection(FakeXRInputSourceInit);
};

// https://immersive-web.github.io/webxr/#dom-xrwebgllayer-getviewport
dictionary FakeXRViewInit {
  required XREye eye;
  // https://immersive-web.github.io/webxr/#view-projection-matrix
  required Float32Array projectionMatrix;
  // https://immersive-web.github.io/webxr/#view-offset
  required Float32Array viewOffset;
  // https://immersive-web.github.io/webxr/#dom-xrwebgllayer-getviewport
  required FakeXRViewportInit viewport;
};

// https://immersive-web.github.io/webxr/#xrviewport
dictionary FakeXRViewportInit {
    required long x;
    required long y;
    required long width;
    required long height;
};

dictionary FakeXRBoundsPoint {
  double x; double z;
};

// When used as a native origin, it is in the reference space
// where the viewer's native origin is identity at initialization
dictionary FakeXRRigidTransformInit {
  required Float32Array position;
  required Float32Array orientation;
};

interface FakeXRInputSourceInit {
  XRHandedness handedness;
  XRTargetRayMode targetRayMode;
};

interface FakeXRInputController {
  void setOrigins(
    boolean emulatedPosition, 
    FakeXRRigidTransformInit pointerOrigin, 
    FakeXRRigidTransformInit? gripOrigin);

  // Temporarily disconnect the input device
  void disconnect();

  // Reconnect a disconnected input device
  void reconnect();

  // Start a selection for the current frame with the given button index
  void startSelection(long? buttonIndex = null);

  // End selection for the current frame with the given button index
  void endSelection(long? buttonIndex = null);
};
```

These initialization object and control interfaces do not represent a complete set of WebXR functionality, 
and are expected to be expanded on as the WebXR spec grows.
