# WebVR Input Explained

## WARNING: UNSTABLE
The content in this explainer is undergoing rapid iteration is should no part of it should be considered representative of a final API.

## What is WebVR?
See the [WebVR Explainer](https://github.com/w3c/webvr/blob/master/explainer.md)

## What is WebVR Input?
Whereas the WebVR spec is primarily concerned with detecting headset movement and displaying imagery to the user, this explainer describes how developers can receive input from the various control mechanims provided by VR headsets, whether that's as simple as a one-button gaze and click input scheme for Cardboard devices or as complex as full 6DoF, multi-button controllers.

### Goals
Define core input models for Virtual Reality applications on the web by providing the following:

* Access to raw pose, button, and axis data of VR input devices.
* A higher-level, ray-based input system that works with most VR systems.

// Future goals? (v0.1+?)
* Default visualizations for VR input.
* A method for providing simple semantic descriptions of VR scenes.
* Haptic feedback.

### Non-goals
* Handle every possible VR input device.
* Full hand or body tracking.
* 2D DOM input with VR input devices.

As a special note on that last item: We expect that most VR system will eventually offer a way to browse traditional 2D web pages, which effectively necessitates emulating mouse and/or touch events with the controller. This should be handled invisibly to the page.

## Use cases
There's two ways to use the VR input API:

**Event Based:** The app registers event listeners to inform it of when the user interacts with the input devices. This is similar to mouse, touch, or pointer events on the 2D DOM. The primary difference is that the page is still responsible for handling scene rendering and determining what object the user is interacting with (frequently via a ray cast into the scene.)

This approach makes it easy to determine when the user is performing discreet actions across a wide range of devices, but it only provides snapshots of the input state when the user performs some action.

**Polling Based:** For experiences that need to continuously track the controller's motion over time or simply find it more useful to manually track button state a more raw, imperitive model is provided that allows the application to poll the controller states. This model offers the ability to perform more detailed tracking, though it requires more manual handling by the developer.

Both models CAN be used in tandem if needed.

## Basic WebVR Input usage

### Pose Tracking
Whether the application is using an event or polling based input system, tracking the controller's pose is handled the same way. Input events, just like `requestFrame` callbacks, will provide a `VRFrame` object that can be used to query the pose of controllers in a given coordinate system. The `frame` can also be used to query the `VRDevicePose`, but since these events fall outside the render loop no `VRViews` will be provided.

```js
function onVrStart() {
  vrSession.addEventListener("controlleradded", onControllerAdded);
}

function onControllerAdded(event) {
  // Get the initial pose for the controller that just connected.
  let controllerPose = event.frame.getControllerPose(event.controller, vrFrameOfRef);

  // Also grab the head pose.
  let headPose = event.frame.getDevicePose(vrFrameOfRef);

  // Do something with the poses.
}
```

Controller poses can also be queried in any `requestFrame` callback.

```js
function onDrawFrame(vrFrame) {
  let pose = vrFrame.getDevicePose(vrFrameOfRef);
  gl.bindFramebuffer(vrSession.baseLayer.framebuffer);

  // Probably don't want to do per-frame if you can help it?
  let controllers = vrSession.getControllers();

  for (let controller of controllers) {
    let controllerPose = vrFrame.getControllerPose(controller, vrFrameOfRef);

    if (controllerPose) {
      // Draw a representation of the controller for each view
      for (let view of vrFrame.views) {
        // This will only be present on controllers with a tracked position.
        if (controllerPose.poseFromOriginMatrix) {
          drawController(view, controllerPose.poseFromOriginMatrix);
        }

        // This should be present for any controller.
        if (controllerPose.pointerFromOriginMatrix) {
          drawCursor(view, controllerPose.pointerFromOriginMatrix);
        }
      }
    }
  }
}
```

If a controller can be tracked the `VRControllerPose` will provide a `poseFromOriginMatrix` to indicate it's position and orientation. This will be `null` if the controller can't be tracked or has temporarily lost tracking.

Even controllers with no tracking capabilities, however, must provide a `pointerFromOriginMatrix`. This represents a transform to be applied to a ray which points from the origin down the negative Z axis, and indicates where the controller is "pointing". If the controller has no tracking capabilities the pointer ray should originate from the users head and follow their gaze. If a pointer ray cannot be determined because a tracked controller has lost tracking or the users head has lost tracking with a non-tracked controller, this will be `null`.

### Input states
Controller input state (joysticks, touchpads, triggers, and buttons) can be read at any point, not just within frame or event callbacks, but may not be updated more frequently than the frame loop. The `VRControllerInput` objects are "live", meaning that the same object gets updated over time with new values.

```js
function printControllerStates() {
  let controllers = vrSession.getControllers();

  for (let controller of controllers) {
    // Can't get poses without a VRPresentationFrame, so ignore that for now.

    `string text ${expression} string text`
    let controllerString = `Controller State (hand: ${controller.hand})\n`;

    controllerString += inputStateString(input.touchpad);
    controllerString += inputStateString(input.joystick);
    controllerString += inputStateString(input.trigger);
    controllerString += inputStateString(input.grip);

    for (let button of input.buttons) {
      controllerString += inputStateString(button);
    }

    console.log(controllerString);
  }
}

function inputStateString(input) {
  if (!input)
    return null;

  let stateString = `${input.name} - `;

  if (input.xAxis)
    stateString += `X: ${input.xAxis}, `;

  if (input.yAxis)
    stateString += `Y: ${input.yAxis}, `;

  stateString += `pressed: ${input.pressed}, `;
  stateString += `touched: ${input.touched}, `;
  stateString += `value: ${input.value}\n`;

  return stateString;
}
```

### Event-based input
There are two type of input events that WebVR produces.

**State events:** These events communicate simple input state changes as they happen. Examples include controllers being connected and disconnected or buttons and triggers being pressed. No semantic meaning is attributed to the state change.

```js
function onVrStart() {
  vrSession.addEventListener("inputtouchstart", onTouchStart);
  vrSession.addEventListener("inputtouchend", onTouchEnd);
}

function onTouchStart(event) {
  if (event.input == event.controller.touchpad)
    displayMenuOverlay();
}

function onTouchEnd(event) {
  if (event.input == event.controller.touchpad)
    hideMenuOverlay();
}

```

**Gesture events:** Gesture events communicate semantically significant actions performed in VR. Examples include making a selection or grabbing and dragging. The exact inputs that trigger these events are controlled by the UA and dependent on the hardware that the user has. For example: On a Daydream controller clicking the touchpad may be considered the selection action, but on a Vive wand using the trigger fires the select event instead. Similarly, on a device like Cardboard pressing and holding the headsets button may initiate a future "grab" gesture while an Oculus Rift may use it's dedicated grip trigger. Gesture events provide the `target` object that the gesture was fired against, guaranteed to be a `VRGestureTarget`, but at this time only a single potential target is defined: the `VRSession`.

```js
function onVrStart() {
  vrSession.addEventListener("selectstart", onSelectStartGesture);
  vrSession.addEventListener("selectend", onSelectEndGesture);
}

function onSelectStartGesture(event) {
  let pose = event.frame.getControllerPose(event.controller, vrFrameOfRef);
  if (pose) {
    // Ray cast into scene with the pointer to determine if anything was hit.
    // If so, cache the object for testing later.
    selectedObject = scene.rayPick(pose.pointerFromOriginMatrix);
  }
}

function onSelectEndGesture(event) {
  let pose = event.frame.getControllerPose(event.controller, vrFrameOfRef);
  if (pose) {
    // See if selection started and ended on the same object. If so, perform
    // some action on the object.
    if (selectedObject == scene.rayPick(pose.pointerFromOriginMatrix)) {
      onSelection(selectedObject);
    }
  }
}
```

## Appendix A: I don’t understand why this is a new API. Why can’t we use…

### Pointer events
Pointer events and their various sub-parts (mouse, touch, etc.) were created with the needs of a 2D web in mind, and serve that purpose well. Trying to bolt on an understanding of 3D space would both over-complicate the APIs with functionality that most pages don't need and make VR input a second-class citizen within the larger pointer even model.

We do expect that many future VR uses (such as interacting with 2D DOM rectangles in 3D space) will need to emulate 2D input methods, producing appropriately transformed pointer events in response to VR input.

### The Gamepad API
This was actually the first thing that we tried, extending the API with the necessary pose data. It proved to be mechanically feasible, but provided a confusing, non-semantic view of VR input that left developers guessing as to what each button meant based on the device name string. Obviously that's a fairly fragile model, and not one that can reasonably extend to a robust ecosystem of hundreds of different devices. It also lacks many of the higher level concepts that we feel developers will eventually need (events, ray generation, action mapping across devices, controller visualization, etc.)

## Appendix B: Proposed IDL

```webidl
//
// Controllers
//

enum VRControllerHand {
  "",
  "left",
  "right"
};

// Ugh on the name. Need to disambiguate from "Input" though.
interface VRController {
  readonly attribute VRControllerHand hand;

  readonly attribute VRControllerInput? touchpad;
  readonly attribute VRControllerInput? joystick;
  readonly attribute VRControllerInput? trigger;
  readonly attribute VRControllerInput? grip;
  readonly attribute FrozenArray<VRControllerInput> buttons;

  VRControllerInput? queryInput(DOMString selector); // v0.1+?
};

//
// Input
//

interface VRControllerPose {
  readonly attribute Float32Array? poseFromOriginMatrix;
  readonly attribute Float32Array? pointerFromOriginMatrix;

  // velocity/acceleration here? v0.1+?
};

// Live object
interface VRControllerInput {
  readonly attribute DOMString name;

  readonly attribute boolean pressed;
  readonly attribute boolean touched;
  readonly attribute double  value;
  readonly attribute double? xAxis;
  readonly attribute double? yAxis;
}

//
// Frame
//

partial interface VRPresentationFrame {
  VRControllerPose? getControllerPose(VRController controller, VRCoordinateSystem coordinateSystem);
};

//
// Session
//

partial interface VRSession : VRGestureTarget {
  attribute EventHandler oncontrolleradded;
  attribute EventHandler oncontrollerremoved;

  attribute EventHandler oninputpressed;
  attribute EventHandler oninputreleased;
  attribute EventHandler oninputtouchstart;
  attribute EventHandler oninputtouchend;

  FrozenArray<VRController> getControllers();
};

//
// Events
//

interface VRGestureTarget : EventTarget {
  attribute EventHandler onselectstart;
  attribute EventHandler onselectend;
}

[Constructor(DOMString type, VRControllerEventInit eventInitDict)]
interface VRControllerEvent : Event {
  readonly attribute VRPresentationFrame frame;
  readonly attribute VRController controller;
};

dictionary VRControllerEventInit : EventInit {
  required VRPresentationFrame frame;
  required VRController controller;
};

[Constructor(DOMString type, VRControllerInputStateEventInit eventInitDict)]
interface VRControllerInputStateEvent : Event {
  readonly attribute VRPresentationFrame frame;
  readonly attribute VRController controller;
  readonly attribute VRControllerInput input;
};

dictionary VRControllerInputStateEventInit : EventInit {
  required VRPresentationFrame frame;
  required VRController controller;
  required VRControllerInput input;
};

[Constructor(DOMString type, VRGestureEventInit eventInitDict)]
interface VRGestureEvent : Event {
  readonly attribute VRPresentationFrame frame;
  readonly attribute VRController controller;
  readonly attribute VRControllerInput input;
  // Normal `target` attribute is always guaranteed to be a VRGestureTarget
};

dictionary VRGestureEventInit : EventInit {
  required VRPresentationFrame frame;
  required VRController controller;
  required VRControllerInput input;
  // Again, existing `target` takes on new meaning here.
};
```
