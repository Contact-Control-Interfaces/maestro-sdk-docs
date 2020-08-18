# Maestro Unity SDK Guide

## This guide is for Maestro devices using external tracking systems. See [this guide](./Alpha_README.md) for the Alpha models

The Unity SDK [can be downloaded here](https://github.com/Contact-Control-Interfaces/maestro-sdk-unity/releases/tag/v2.3).

The SDK contains scripts for controlling the Maestro device, as well as three example scenes demonstrating some interactions.

This version is built to be tracking agnostic, but for the sake of simplicity the current SteamVR plugin is included. This aspect of the included content may be updated through the Unity Asset Store.

## Setup

First you will need to run the [Maestro Alpha installer](https://github.com/Contact-Control-Interfaces/maestro-installer), which will handle creating configuration files needed for the Maestro to work. This is described under "Initial Setup" and "Configuration" [here](https://contact-control-interfaces.github.io/maestro-sdk-docs/C/html/index.html)

Once you have configuration files installed and the Unity Maestro SDK Unity Package imported into a Unity project, an example rig prefab "[MaestroIndexRig]" will be available under `Assets/Maestro/Prefabs`. This is an example configuration set up to allow Maestro haptics while using the Valve Index's motion and finger tracking.

The Maestro SDK encourages but does not require two layers definitions used for interactions, which by default are named "GrippedObject" and "IgnoreWhileGripped". These names can be changed per `MaestroHand` under Advanced Configuration.

Additional setup is required for the Sandbox scene, which is as follows:

## Included Scenes

### Sandbox
A haptic sandbox with a variety of interactions, including picking up objects, pushing buttons, using a hammer, and finger painting.

By default, the user starts in a void where they are prompted to scan both hands to spawn the rest of the scene. This functionality can be disabled simply by disabling the "HandScanner" object in the main hierarchy.

Additionally, this scene requires two new input axes to be defined "Clear" and "Lock", which clear the paint canvas and regenerates the hand scanner, respectively.

### Hoops
A scene demonstrating two-hand pickups in the form of a basketball shootaround.

The user is given a silo that can generate endless basketballs, and a small lever to control the distance from the hoop. Additionally, pressing on the touchpad of the Index controllers will generate a basketball directly in front of the user.

### Escape
A low-visibility scene where the user must escape a room with doors at both ends. By default the first door approached breaks, forcing the player to navigate to the other door.

## Functionality

There are two major portions of the Maestro's haptics: 
- Pull Amplitude - Can be from `0` to `255`, with `0` being no pull and `255` being full force feedback.
- Vibration Effect - Can be from `0` to `128`, referencing a haptic effect in [this table](https://contact-control-interfaces.github.io/maestro-sdk-docs/C/html/group__vibration_control.html). Undefined effects produce no effect.

Maestro device connectivity is controlled by either a `RightMaestroGloveBehaviour` or `LeftMaestroGloveBehaviour`. These scripts have a boolean that shows whether the system detects a connected device for the corresponding hand.

All major haptic functionality is controlled by four scripts: `MaestroHand`, `FingerTipCollider`, `MaestroInteractable`, and `HapticResponse`. In order to have the hand able to pick up an object:
- The `MaestroHand` script must be properly bound to the hand hierarchy, and
- The `MaestroInteractable` script must be placed onto the desired object.

The `FingerTipCollider` script is necessary for interactions to properly occur, however they are spawned by the `MaestroHand` script and as such require no setup. The configuration for the other scripts are as follows:

### `MaestroHand`
Should be placed on the main hand skeleton, beneath either a `RightMaestroGloveBehaviour` or `LeftMaestroGloveBehaviour`. Requires an `AudioSource` that will play a noise on successful pickup.
- Which Hand - Is this hand a right hand or left hand?
- Other Hand - Reference to the other hand corresponding to the user. Must be defined for two-hand pickup to occur
- One-hand Grabbing - Shows whether this hand is currently holding something
- Two-hand Grabbing - Show whether this pair of hand is currently holding something
- Default Pull - Ambient pull amplitude for this hand
- Default Vibration - Ambient vibration effect for this hand

#### Hand Configuration
Defines the key hand locations and size.
- Fingertip Radius - The radius of colliders spawned at finger tips and knuckles
- Palm Size - The size of the box for the collider spawn at the center of the palm
- Palm Offset - How much the palm box is offset from the given Palm transform

#### Advanced Configuration
More complex interaction variables that should only be adjusted when necessary
- Reset Distance/Reset Speed - Defines when `FingerTipCollider`s will attempt to teleport to their target location
- Release Ratio - Defines ease of release, < 1.0 should be hard to keep a hold of, >> 1.0 will stick to the hand.
- Tool Release Ratio - Same as Release Ratio, but specifically for objects classified as Tools.
- Grabbed Object Layer - Layer that grabbed objects will be set to during interaction, by default "GrippedObject"
- Ignore While Gripped Layer - Layer that will ignore physics with grabbed objects, by default "IgnoreWhileGripped"
- Anchor Movement Scalar - Scales the movement speed of the grab anchor
- Physics Material - Physics material applied to all `FingerTipCollider`s

#### Debug
Outputs various information useful for debug. Also exposes one boolean:
- Show colliders - Should we render the `FingerTipCollider`s on contact?

### `MaestroInteractable`
Must be placed on any game object that requires haptics. The main three fields are:
- Amplitude - Force feedback applied when touching this object
- Vibration Effect - Vibration effect applied when touching this object
- Type - Either "Static" (unable to be picked up), "One hand grab", or "Two hand". Two-hand objects are unable to be picked up with only one hand, and vice-versa.

This script also defines a few events that may be used to easily create complex interactions
- On Grab - Called when after the object is grabbed
- On Release - Called when after the object is released
- On Touch(FingerTipCollider) - Called when a specific finger touches the object
- Un Touch(FingerTipCollider) - Called when a specific finger stops touching the object

Additionally, these other options may be used for differing behavior
- Ignore Taps - For "Static" objects, vibration is applied based on how hard an object is tapped. This option disables this feature.
- Use Render Center - Anchor the grab from the render center instead on the object's transform
- Maintain Orientation - Do not rotate the object while it is held
- Maintain Position - Do not change the object's position within the hand while it is held
- Stay In Hand - Attempts to keep the object within the hand using translation rather than physics
- Grip Transform - Defines where this object is optimally held. Classifies this object as a Tool.
- Grip Collider - Defines the collider for a handle for an object.

### `HapticResponse`
Abstract class that extends the functionality of `MaestroInteractable`. Automatically detected by `MaestroInteractable` when placed beneath it in the hierarchy. Provides a single abstract function `GetResponse(MaestroInteractable)` that is called on FixedUpdate. An example `HammerResponse` is provided under `Assets/Maestro/Responses`.

### `FingerTipCollider`
Must be placed on each fingertip and in the center of the palm. Spawned and set up automatically by `MaestroHand`.

## Troubleshooting

#### Running my scene causes Unity to terminate!

It should be noted that the DLLs used by our SDK are 64-bit only. As such, if the Unity platform settings are set to `Any CPU` or `x86` execution will halt.

To address this, you may find all DLLs under `Maestro/Core/_DLLs`. Set each DLL's CPU setting to `x86_64`. If you are performing builds, this will also have to be done so the DLLs are included with 64-bit builds, and excluded for 32-bit builds.

You may also have to enable `Load on startup` under `Plugin load settings`. The Unity editor will have to be restarted for this to take effect.

<strong>Please contact us at support@contactci.co with any questions.</strong>
