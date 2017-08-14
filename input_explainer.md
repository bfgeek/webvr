# WebVR Input Explained

## WARNING: UNSTABLE
The content in this explainer is undergoing rapid iteration and no part of it should be considered representative of a final API.

## What is WebVR?
See the [WebVR Explainer](https://github.com/w3c/webvr/blob/master/explainer.md)

## What is WebVR Input?
Whereas the WebVR explainer is primarily concerned with detecting headset movement and displaying imagery to the user, this explainer describes how developers can receive input from the various control mechanisms provided by VR headsets, whether that's as simple as a one-button gaze and click input scheme for Cardboard devices or as complex as full 6DoF, multi-button controllers.

### Goals
Define core input models for Virtual Reality applications on the web by providing the following:

* A high level, ray-based input system that works with most VR/AR systems across multiple input sources.
* Access to raw pose, button, axis and haptic data of dedicated VR/AR input sources (Tracked controllers, hands, etc.)
* Enable sites to visualize VR/AR input and input devices.

### Non-goals
* A method for providing semantic descriptions of VR/AR scenes.
* Handle every possible VR input device.
* Full hand or body tracking.
* 2D DOM input with VR input devices.

As a special note on that last item: We expect that most VR system will eventually offer a way to browse traditional 2D web pages, which effectively necessitates emulating mouse and/or touch events with the controller. This should be handled invisibly to the page.

## Use cases
VR applications may receive input from a variety of different sources. These are broken down into a few categories:

// TODO: Comment on how each category is used
* Tracked input sources
  * Purpose-built VR controller with some level of tracking capability.
  * Optical hand tracking of varying levels of detail (HoloLens, Leap Motion)
* Gaze and commit
  * Touchpad on the headset (GearVR)
  * Single button on the headset (Cardboard)
  * Simple handheld bluetooth clicker (Cardboard, HoloLens)
  * Traditonal Gamepads
  * Mouse (in exclusive sessions)
* Screen input
  * Mouse (in non-exclusive sessions)
  * Touch
  * Stylus

Obviously these all represent wildly different styles of input, which can be very difficult for individual applications to make sense of in a consistent manner. The primary goal of the input API is to provide a high level abstraction of all these input sources. As with non-VR/AR WebGL content the application is responsible for all rendering and hit testing.

To accomplish this, each of these categories of inputs are surfaced by the input API in several ways:

**Polling:** For rendering of cursors and tracked controllers, an imperative model is provided that allows the application to poll a predictive controller pose. Polling also provides per-frame controller element states.

**State Events:** The app can also register event listeners to inform it of when the user interacts with individual elements of a VR/AR input source such as tracked controllers and hands. These events are treated as user gestures by the UA.

**Gesture Events:** Finally the app can register event listeners for when the user performs common actions with an input source. These events are also treated as user gestures by the UA.

Aside from generating high level gestures, the input API does not attempt to describe the state of input sources that are already surfaced by the UA, such as mouse, keyboard, touch, stylus, or gamepad inputs.

## Basic WebVR Input usage

### Hit testing


### Gesture events
Applications that want to work on the widest variety of platforms and devices will want to rely heavily on gesture events. Gesture events communicate semantically significant actions performed in VR, such as making a selection.

The exact inputs that trigger these events are controlled by the UA and dependent on the hardware that the user has. For example, to perform a select gesture on a variety of potential hardware:
* On Daydream the user would click the controller touchpad.
* On HoloLens the user would performs a hand gesture.
* On a Vive or Rift the would pull the trigger.
* On Cardboard the user would press the headsets button.

```js
function onVrStart() {
  vrSession.addEventListener("select", onSelectGesture);
}

function onSelectGesture(event) {
  let pose = event.frame.getInputPose(event.inputSource, vrFrameOfRef);
  if (pose) {
    // Ray cast into scene with the pointer to determine if anything was hit.
    // If so, cache the object for testing later.
    let selectedObject = scene.rayPick(pose.pointerPoseMatrix);
    if (selectedObject) {
      onSelection(selectedObject)
    }
  }
}
```

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

    // gripPoseMatrix will only be present on tracked controllers.
    if (controllerPose && controllerPose.gripPoseMatrix) {

      // Draw a representation of the controller for each view
      for (let view of vrFrame.views) {
        let viewport = view.getViewport(vrSession.baseLayer);
        gl.viewport(viewport.x, viewport.y, viewport.width, viewport.height);

        drawController(view, controllerPose.gripPoseMatrix);
      }
    }
  }
}
```

If an input source can be tracked the `VRInputPose` will provide a `gripPoseMatrix` to indicate it's position and orientation. This will be `null` if the input source can't be tracked or has temporarily lost tracking. The `gripPoseMatrix` is a transform into a space where if the user was holding a straight rod in their hand it would be aligned with the Y axis and the origin rests at their palm. This enables developers to properly objects intended for the user to hold. For example, a sword would be positioned so that the blade points directly down the positive Y axis and the center of the handle is at the origin.

Even input sources with no tracking capabilities, however, must provide a `pointerPoseMatrix`. This represents a transform to be applied to a ray which points from the origin down the negative Z axis, and indicates where the input is "pointing". If the input source has no tracking capabilities the pointer ray should originate from the users head and follow their gaze. Even if the input source is tracked, the pointer ray may not align with the Y axis of the `gripPoseMatrix` space, and should follow platform guidelines if there are any. In absense of platform guidance the pointer ray should most likely point in the same direction as the user's index finger if it was outstretched while holding the controller. If a pointer ray cannot be determined because a tracked source has lost tracking or the users head has lost tracking with a non-tracked source, this will be `null`.

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

## Input visualization

### Cursor rendering
Many VR applications will want to render a cursor and possibly a ray to visualize where the user is pointing and what will be targeted when the user performs a select gesture. When rendering the application should take into consideration the types of inputs available and the mode the application is in to determine what elements of a the pointer make sense to draw.

* If tracked controllers are present, a pointer should be rendered as a ray originating from each controller, optionally with a cursor shown at the point of ray intersection with the scene.
* In exclusive sessions where no tracked controllers are present, the VRDevicePose should be used to render a cursor with no associated ray. This is used for gaze-based selection scenarios where having a ray emitting from the users head would be visually confusing.
* In non-exclusive sessions the VRDevicePose pointer should not be given a visual representation. In this case input will likely be coming from a touchscreen or mouse, and as such a gaze cursor is nonsensical.

```js
let cursorMode = vrSession.exclusive ? "gaze" : "none";

function onSelectGesture(event) {
  if (!vrSession.exclusive) { // Magic Window
    cursorMode = "none"
  } else if (event.inputSource.supportsPointing) { // Pointing devices
    cursorMode = "pointing";
  } else { // Clicker devices
    cursorMode = "gaze";
  }
}

function drawPrimaryPointer(vrFrame) {
  let controllers = vrFrame.session.getControllers();

  if (cursorMode == "pointing") {
    // When tracked controllers are present, draw a pointer for each of them.
    for (let controller of controllers) {
      let inputPose = vrFrame.getInputPose(controller, vrFrameOfRef);

      // If the controller is capable of supporting pointing and currently has a
      // valid pointer ray we should draw the ray and cursor.
      if (inputPose && inputPose.pointerPoseMatrix) {
        // Draw a representation of the controller's pick ray for each view.
        drawRay(vrFrame.views, inputPose.pointerPoseMatrix);
        drawCursor(vrFrame.views, inputPose.pointerPoseMatrix);
      }
    }
  } else if (cursorMode == "gaze") {
    // When in gaze mode, draw a gaze cursor.
    let devicePose = vrFrame.getDevicePose(vrFrameOfRef);
    if (devicePose) {
      // Draw the gaze cursor for each view.
      drawCursor(vrFrame.views, devicePose.poseModelMatrix);
    }
  }
}
```

### Controller rendering

When using tracked controllers users expect to be able to see a visual representation of the controller in VR applications. The representation typically breaks down into one of two categories: In some cases the VR app has a contextually relevant visualization that it prefers to use, such as a sword, paint brush, hands, etc. that is well suited to the mechanics and artistic direction of the app. In all other cases it's typically preferable to show something that matches what the user is physically holding as closely as possible.

To achive the latter effect, applications can query a mesh that represents a given controller using `VRController`'s `mesh` attribute. Calling the `mesh`'s `requestBuffer()` method returns a promise which resolves to an `ArrayBuffer` containing a [Binary glTF 2.0](https://github.com/KhronosGroup/glTF/tree/master/specification/2.0#binary-gltf-layout) asset that visually represents the controller. The glTF asset must not reference any external resources, but may contain embedded textures. The mesh data it provides must be properly sized such that it matches the phyical controller dimensions as closely as possible using a scale of 1 unit equaling 1 real-world meter.

Multiple `VRController`s may share the same `VRControllerMesh`, which allows them to be rendered as instances of a single resource.

If the UA cannot provide a mesh that accurately reflects the user's physical hardware it is allowed to provide a generic mesh instead to ensure the user has some sense of where their controllers are. This could be something like a hand or generic remote.

Some VR systems allow the user to customize the virtual appearance of their controller. The UA is allowed and encouraged to provide the customized mesh when possible to provide stronger visual continuity between web content and the native system. It should be noted, however, that this is a potential source of high fidelity fingerprinting data and as such UAs should strongly consider gaining user permission before doing exposing custom meshes.

The glTF asset should represent the various movable elements (buttons, triggers, joysticks, etc) of the controller with different nodes in the scene hierarchy so that they can be transformed independently. The UA will then need to provide a way to map `VRController` `element` states to node transforms so that the rendered controller can visually reflect the user's interactions with it's elements. This document does not have a recommendation for how to accomplish that mapping at this time. (Suggestions welcome!)

```js
// Loads and renders controller meshes using a fictional 3D rendering library.
function loadControllerMeshes() {
  let controllers = vrSession.getControllers();

  for (let controller of controllers) {
    controller.mesh.requestBuffer().then((gltfArrayBuffer) => {
      controller._meshNode = GLTF2Loader.parseBinary(gltfArrayBuffer);
      scene.addNode(controller._meshNode);
    });
  }
}

function onVRFrame(vrFrame) {
  // Update the controller state.
  let controllers = vrSession.getControllers();

  for (let controller of controllers) {
    if (controller._meshNode) {
      let controllerPose = vrFrame.getInputPose(controller, vrFrameOfRef);
      controller._meshNode.setTransform(controllerPose.gripPoseMatrix);
      // TODO: Update element transforms to match controller state.
    }
  }

  // Draw the scene, which presumably includes drawing the controller meshes at
  // their newly updated transforms.
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

interface VRInputSource {
  readonly attribute VRInputSourceType type;
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
  readonly attribute boolean supportsPointing; 
  readonly attribute boolean supportsGrabbing; // TODO: Something better.
  readonly attribute VRControllerElementMap elements;
  readonly attribute VRControllerMesh mesh;
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

interface VRControllerMesh {
  Promise<ArrayBuffer> requestBuffer();
};

//
// Frame
//

interface VRInputPose {
  readonly attribute Float32Array? gripPoseMatrix;
  readonly attribute Float32Array? pointerPoseMatrix;

  // readonly attribute VRHandPose? handPose; // v0.1+?
  // velocity/acceleration here? v0.1+?
};

partial interface VRPresentationFrame {
  VRInputPose? getInputPose(VRInputSource inputSource, VRCoordinateSystem coordinateSystem);
};

//
// Session
//

partial interface VRSession {
  attribute EventHandler onselect;

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
