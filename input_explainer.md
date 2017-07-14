# WebVR Input Explained

## WARNING: UNSTABLE
The content in this explainer is undergoing rapid iteration is should no part of it should be considered representative of a final API.

## What is WebVR?
See the [WebVR Explainer](https://github.com/w3c/webvr/blob/master/explainer.md)

## What is WebVR Input?
Whereas the WebVR spec is primarily concerned with detecting headset movement and displaying imagery to the user, this explainer describes how developers can receive input from the various control mechanims provided by VR headsets, whether that's as simple as a one-button gaze and click input scheme for Cardboard devices or as complex as full 6DoF, multi-button controllers.

### Goals
Define core input models for Virtual Reality applications on the web by providing the following:

* A high level, ray-based input system that works with most VR systems across multiple input sources.
* Access to raw pose, button, and axis data of purpose-built VR controllers.

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

**Event Based:** The app registers event listeners to inform it of when the user interacts with the input devices. This is similar to mouse, touch, or pointer events on the 2D DOM. The primary difference is that the application is still responsible for handling scene rendering and determining what object the user is interacting with (frequently via a ray cast into the scene.)

This approach makes it easy to determine when the user is performing discreet actions across a wide range of devices, but it only provides snapshots of the input state when the user performs some action.

**Polling Based:** For experiences that need to continuously track the controller's motion over time or simply find it more useful to manually track button state a more raw, imperitive model is provided that allows the application to poll the controller states. This model offers the ability to perform more detailed tracking, though it requires more manual handling by the developer.

Both models CAN be used in tandem if needed.

## Types of Input

### Input Sources
VR applications may receive input from a variety of different sources. To name just a few possibilities:
 - Purpose-built VR controller with some level of tracking capability (Vive, Rift, Daydream, etc.)
 - Optical hand tracking of varying levels of detail (HoloLens, Leap Motion)
 - Touchpad on the headset (GearVR)
 - Single button on the headset (Cardboard)
 - Simple handheld bluetooth clicker (Cardboard)
 - Gamepad (Rift, GearVR)
 - Mouse/Keyboard/Touch/Stylus

Obviously these all represent wildly different styles of input, which can be very difficult for individual applications to make sense of in a consistent manner. The primary goal of the WebVR input API is to provide a high level abstraction of all these input sources. It does this by generating a pointer ray for any input source that can be used to select objects in the VR scene. It also exposes a series of semantic "gesture" events that indicate the action the user wishes to apply to the target of the pointer.

The inputs devices used to generate those events are represented by a `VRInputSource` object. The input source can be used to query pose and pointer information, as well as describing the `type` of input source it represents.

```js
function processInputSource(inputSource) {
  switch (inputSource.type) {
    case 'pointerevent': processPointerEvents();
    case 'gamepad': processGamepad(inputSource.gamepad);
    case 'controller': processController(inputSource);
  }
}
```

The `VRInputSource` also describes where the pointer ray will originate from, which can be used to determine what type of rendering is appropriate for that input.

```js
function shouldDrawCursor(inputSource) {
  switch (inputSource.pointerOrigin) {
    case 'screen': return false;
    case 'head': return true;
    case 'hand': return true;
  }
}

function shouldDrawRay(inputSource) {
  switch (inputSource.pointerOrigin) {
    case 'screen': return false;
    case 'head': return false;
    case 'hand': return true;
  }
}
```

### Controller state
The WebVR input API also exposes the state of the raw input elements of purpose built VR controllers, such as the tracked controllers for the Vive or Daydream systems, but does not attempt to describe the state of input sources that are already surfaced by the UA, such as mouse, keyboard, touch, stylus, or gamepad inputs.

These controllers can be queried from the `VRSession` with the `getControllers` method, which returns an array of the currently connected controllers.

### Primary input source
Every `VRPresentationFrame` reports a `primaryInputSource`. This can change from frame to frame, and may jump between different source types as the user interacts with various devices. In most cases, though, the frame's `primaryInputSource` is the one that has most recently generated a gesture event. For example: If the user is actively using a motion controller it will be reported as the `primaryInputSource` as well as still appearing in the `VRSession.getControllers` array. If the user begins pressing buttons on a gamepad, however, the `primaryInputSource` would start reporting a `VRGamepadInputSource`.

## Basic WebVR Input usage

### Pose Tracking
Whether the application is using an event or polling based input system, tracking the pose of any input source is handled the same way. Input events, just like `requestFrame` callbacks, will provide a `VRPresentationFrame` object that can be used to query the pose of a `VRInputSource` in a given coordinate system. The `frame` can also be used to query the `VRDevicePose`, but since these events fall outside the render loop no `VRViews` will be provided.

```js
function onVrStart() {
  vrSession.addEventListener("controlleradded", onControllerAdded);
}

function onControllerAdded(event) {
  // Get the initial pose for the controller that just connected.
  let inputPose = event.frame.getInputPose(event.inputSource, vrFrameOfRef);

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
    let controllerPose = vrFrame.getInputPose(controller, vrFrameOfRef);

    // poseFromOriginMatrix will only be present on tracked controllers.
    if (controllerPose && controllerPose.poseFromOriginMatrix) {

      // Draw a representation of the controller for each view
      for (let view of vrFrame.views) {
        let viewport = view.getViewport(vrSession.baseLayer);
        gl.viewport(viewport.x, viewport.y, viewport.width, viewport.height);

        drawController(view, controllerPose.poseFromOriginMatrix);
      }
    }
  }
}
```

If an input source can be tracked the `VRInputPose` will provide a `poseFromOriginMatrix` to indicate it's position and orientation. This will be `null` if the input source can't be tracked or has temporarily lost tracking.

Even input sources with no tracking capabilities, however, must provide a `pointerFromOriginMatrix`. This represents a transform to be applied to a ray which points from the origin down the negative Z axis, and indicates where the input is "pointing". If the input source has no tracking capabilities the pointer ray should originate from the users head and follow their gaze. If a pointer ray cannot be determined because a tracked source has lost tracking or the users head has lost tracking with a non-tracked source, this will be `null`.

### Primary input visualization
In almost all cases if a pick-ray visualization is going to be rendered it should be done using the `primaryInputSource` pose. When rendering the application should take into consideration the `pointerVisualStyle`, which indicates what elements of a the pointer make sense to draw for that input source.

* "ray": Pointer should be rendered as a ray, optionally with a cursor shown at the point of ray intersection with the scene.
* "cursor": Pointer should be rendered as a cursor with no associated ray. This is used for gaze-based selection scenarios where having a ray emitting from the users head would be visually confusing.
* "none": The pointer's should not be given a visual representation. This is used in scenarios where rendering a pointer doesn't make sense or a pointer visualization is provided by the system. An example would be a Magic Window session on a touchscreen or desktop.

```js
function drawPrimaryPointer(vrFrame) {
  // Determine if there is a primary input source and if it should be drawn
  if (vrFrame.primaryInputSource &&
      vrFrame.primaryInputSource.visualStyle != "none") {
    let primaryInputPose = vrFrame.getInputPose(
        vrFrame.primaryInputSource, vrFrameOfRef);

    // This should be present for most input sources. Main exception is
    // Magic Window w/ touchscreens.
    if (primaryInputPose && primaryInputPose.pointerFromOriginMatrix) {
      // Draw a representation of the controller for each view
      for (let view of vrFrame.views) {
        let viewport = view.getViewport(vrSession.baseLayer);
        gl.viewport(viewport.x, viewport.y, viewport.width, viewport.height);

        if (vrFrame.primaryInputSource.visualStyle == "ray") {
          drawRay(view, primaryInputPose.pointerFromOriginMatrix);
        }
        if (vrFrame.primaryInputSource.visualStyle == "ray" ||
            vrFrame.primaryInputSource.visualStyle == "cursor") {
          drawCursor(view, primaryInputPose.pointerFromOriginMatrix);
        }
      }
    }
  }
}
```

### Controller element states
Controller elements (joysticks, touchpads, triggers, and buttons) can be read at any point, not just within frame or event callbacks, but may not be updated more frequently than the frame loop. The `VRControllerElement` objects are "live", meaning that the same object gets updated over time with new values.

The elements are provided in a map, with each element's name serving as the key. This allows them to be easily iterated over or accessed directly by name:

```js
function printControllerStates() {
  let controllers = vrSession.getControllers();

  for (let controller of controllers) {
    // Can't get poses without a VRPresentationFrame, so ignore that for now.
    let controllerString = `Controller State (hand: ${controller.hand})\n`;

    for (let [name, element] of controller.elements) {
      controllerString += controllerElementString(name, element);
    }

    console.log(controllerString);
  }
}

function controllerElementString(name, element) {
  if (!input)
    return null;

  let stateString = `  ${name} - `;

  if (input.xAxis)
    stateString += `X: ${element.xAxis}, `;

  if (input.yAxis)
    stateString += `Y: ${element.yAxis}, `;

  stateString += `pressed: ${element.pressed}, `;
  stateString += `touched: ${element.touched}, `;
  stateString += `value: ${element.value}\n`;

  return stateString;
}
```

Elements should be given lowercase names that match the description given to them by the hardware manufacturer when possible. For example, if there are buttons on the controller labeled "A" and "B", they should be named "a" and "b". If the elements did not have any obvious name, they should be given numbered names according to their percieved order of importance, such as "button1", "trigger2", etc. In addition there are several "common" element names that must be used if the controller element fits the usage they describe, even if the manufacturer provides a slightly different name. These common elements are:

 - "touchpad"
 - "joystick"
 - "trigger"
 - "grip"

This enables applications to easily check for the presence of commonly used controller elements and access their values reliably.

```js
// Updates the position of an object in the scene based on the user's thumb
// motion on whatever controller element is available.
function updateMovement(controller) {
  if (controller.elements.joystick) {
    object.x += controller.elements.joystick.axisX;
    object.y += controller.elements.joystick.axisY;
  } else if (controller.elements.touchpad) {
    if (controller.elements.touchpad.touched) {
      object.x += controller.elements.touchpad.axisX;
      object.y += controller.elements.touchpad.axisY;
    }
  }
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
  if (event.element == event.inputSource.touchpad)
    displayMenuOverlay();
}

function onTouchEnd(event) {
  if (event.element == event.inputSource.touchpad)
    hideMenuOverlay();
}

```

**Gesture events:** Gesture events communicate semantically significant actions performed in VR. Examples include making a selection or grabbing and dragging. The exact inputs that trigger these events are controlled by the UA and dependent on the hardware that the user has. For example: On a Daydream controller clicking the touchpad may be considered the selection action, but on a Vive wand using the trigger fires the select event instead. Similarly, on a device like Cardboard pressing and holding the headsets button may initiate a future "grab" gesture while an Oculus Rift may use it's dedicated grip trigger. Gesture events provide the `target` object that the gesture was fired against, guaranteed to be a `VRGestureTarget`, but at this time only a single potential target is defined: the `VRSession`.

```js
function onVrStart() {
  vrSession.addEventListener("select", onSelectGesture);
}

function onSelectGesture(event) {
  let pose = event.frame.getInputPose(event.inputSource, vrFrameOfRef);
  if (pose) {
    // Ray cast into scene with the pointer to determine if anything was hit.
    // If so, cache the object for testing later.
    let selectedObject = scene.rayPick(pose.pointerFromOriginMatrix);
    if (selectedObject) {
      onSelection(selectedObject)
    }
  }
}
```

## Appendix A: I don’t understand why this is a new API. Why can’t we use…

### Pointer events
Pointer events and their various sub-parts (mouse, touch, etc.) were created with the needs of a 2D web in mind, and serve that purpose well. Trying to bolt on an understanding of 3D space would both over-complicate the APIs with functionality that most applications don't need and make VR input a second-class citizen within the larger pointer even model.

We do expect that many future VR uses (such as interacting with 2D DOM rectangles in 3D space) will need to emulate 2D input methods, producing appropriately transformed pointer events in response to VR input.

### The Gamepad API
This was actually the first thing that we tried, extending the API with the necessary pose data. It proved to be mechanically feasible, but provided a confusing, non-semantic view of VR input that left developers guessing as to what each button meant based on the device name string. Obviously that's a fairly fragile model, and not one that can reasonably extend to a robust ecosystem of hundreds of different devices. It also lacks many of the higher level concepts that we feel developers will eventually need (events, ray generation, action mapping across devices, controller visualization, etc.)

## Appendix B: Proposed IDL

```webidl
//
// Controllers
//

enum VRInputSourceType {
  "pointerevent",
  "gamepad",
  "controller"
};

enum VRPointerVisualStyle {
  "none",
  "cursor",
  "ray"
};

interface VRInputSource {
  readonly attribute VRInputSourceType type;
  readonly attribute VRPointerVisualStyle pointerVisualStyle;
};

interface VRGamepadInputSource : VRInputSource {
  readonly attribute Gamepad gamepad;
};

enum VRHand {
  "",
  "left",
  "right"
};

interface VRController : VRInputSource {
  readonly attribute VRHand hand;

  readonly attribute VRControllerElementMap elements; // :P
};

interface VRControllerInputMap {
  readonly maplike<DOMString, VRControllerElement>;
};

interface VRControllerElement {
  readonly attribute boolean pressed;
  readonly attribute boolean touched;
  readonly attribute double  value;
  readonly attribute double? xAxis;
  readonly attribute double? yAxis;
};

//
// Frame
//

interface VRInputPose {
  readonly attribute Float32Array? poseFromOriginMatrix;
  readonly attribute Float32Array? pointerFromOriginMatrix;

  // velocity/acceleration here? v0.1+?
};

partial interface VRPresentationFrame {
  VRInputSource primaryInputSource;

  VRInputPose? getInputPose(VRInputSource inputSource, VRCoordinateSystem coordinateSystem);
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
  attribute EventHandler onselect;
}

[Constructor(DOMString type, VRControllerEventInit eventInitDict)]
interface VRInputSourceEvent : Event {
  readonly attribute VRPresentationFrame frame;
  readonly attribute VRInputSource inputSource;
};

dictionary VRInputSourceEventInit : EventInit {
  required VRPresentationFrame frame;
  required VRInputSource inputSource;
};

[Constructor(DOMString type, VRControllerInputStateEventInit eventInitDict)]
interface VRControllerElementEvent : Event {
  readonly attribute VRPresentationFrame frame;
  readonly attribute VRController inputSource;
  readonly attribute VRControllerElement element;
};

dictionary VRControllerElementEventInit : EventInit {
  required VRPresentationFrame frame;
  required VRController inputSource;
  required VRControllerElement element;
};

[Constructor(DOMString type, VRControllerInputStateEventInit eventInitDict)]
interface VRGestureEvent : Event {
  readonly attribute VRPresentationFrame frame;
  readonly attribute VRInputSource inputSource;
  readonly attribute DOMString element;
};

dictionary VRGestureEventInit : EventInit {
  required VRPresentationFrame frame;
  required VRInputSource inputSource;
  required DOMString element;
};
```
