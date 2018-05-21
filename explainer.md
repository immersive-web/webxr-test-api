In order to allow javascript tests for WebXR there are some basic functions which are common across all tests, 
such as adding a fake test device and specifying poses. Below is a Javascript IDL which attempts to capture 
the necessary functions, based off what was defined in the spec. Different browser vendors can implement this
Javascript IDL in whatever way is most compatible with their browser. For example, some browsers may back the
interface with a WebDriver API while others may use HTTP or IPC mechanisms to communicate with an out of process 
fake backend.

```WebIDL
interface XRTest {
  // Simulates connecting a device to the system.
  // Used to instantiate a fake device for use in tests.
  Promise<FakeXRDeviceController> simulateDeviceConnection(FakeXRDeviceInit);

  // Simulates a device being disconnected from the system.
  Promise<void> simulateDeviceDisconnection(XRDevice);

  // Simulates a user activation (aka user gesture) for the current scope.
  // The activation is only guaranteed to be valid in the provided function and only applies to WebXR
  // Device API methods.
  void simulateUserActivation(Function);
};
```

The promise returned from simulateDeviceConnection resolves with a FakeXRDeviceController, which can be used 
to control the fake XRDevice that has been created in the background. The fake device may be returned by 
navigator.xr.requestDevice, depending on how many devices have been created and how the browser decides to hand 
them out.

```WebIDL
dictionary FakeXRDeviceInit {
	// TODO: Subject to change to match spec changes.
	required boolean supportsExclusive;
}

interface FakeXRDeviceController {
	// Creates and attaches a XRFrameOfReference of the type specified to the device. 
  void setFrameOfReference(XRFrameOfReferenceType,  FakeXRFrameOfReferenceInit);

  // Sets the values to be used for subsequent
  // requestAnimationFrame() callbacks.
  void setXRPresentationFrameData(Float32Array poseModelMatrix, Array<FakeXRViewInit> views);

  // Simulates the user activating the reset pose on a device.
  void simulateResetPose();

  // Simulates the platform ending the sessions.
  void simulateForcedEndSessions();

  // Simulates devices focusing and blurring sessions.
  void simulateBlurSession(XRSession);
  void simulateFocusSession(XRSession);
  
  Promise<FakeXRInputSourceController>  
      simulateInputSourceConnection(FakeXRInputSourceInit);
}

interface FakeXRViewInit {
  required XREye eye;
  required Float32Array projectionMatrix;
  required Float32Array viewMatrix;
};

dictionary FakeXRFrameOfReferenceInit {
  // The transform is an offset from the default world coordinates. 
  // TODO: As the value can change on a per frame basis, this should be setable in 
  // setXRPresentationFrameData.
  Float32Array transform;
  //TODO: Need to have a way to set this to trigger an onBoundsChangedEvent.
  Array<FakeXRBoundsPoint> boundsCoodinates;
}

dictionary FakeXRBoundsPoint {
  double x; double z;
}

interface FakeXRInputSourceInit {
  XRHandedness handedness;
  XRPointerOrigin pointerOrigin;
};

interface FakeXRInputSourceController {
  void setPose(
    boolean emulatedPosition, 
    Float32Array pointerMatrix, 
    Float32Array? gripMatrix);
};
```

These initialization object and control interfaces do not represent a complete set of WebXR functionality, 
and are expected to be expanded on as the WebXR spec grows.
