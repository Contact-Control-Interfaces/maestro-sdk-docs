# Maestro Unity SDK 2.6.0 Guide

## See [this guide](../README.md) for previous versions

Unity SDK 2.6.0 [can be downloaded here](https://github.com/Contact-Control-Interfaces/maestro-sdk-unity/releases/tag/v2.6.0).

The Unity SDK provides compatibilty and smooth integration of Maestro with new and existing XR scenes.

This release includes:
 - Prefabs for Ultraleap and Meta Quest 1/2 hand-tracking compatibility
 - The ability to customize force feedback and vibrotactile haptics per object in scene
 - An example scene showcasing interactions and haptics named "Sandbox"

This Unity SDK is designed to be tracking-agnostic, though we provide prefab rigs for Ultraleap and Meta Quest 1/2 hand tracking.

## Compatibility

The Unity SDK provides the following prefab rigs for ease of integration:
 - [MaestroUltraleapRig] - Provides support for:
   - Leap Motion Controller
   - Stereo IR 170
   - Ultraleap 3Di
 - [MaestroOculusRig]
   - Meta Quest 1/2

It is also possible to roll your own configuration using the SDK components available.

## Setup

Before installing the Unity SDK package, you'll need to add a scoped registry entry for Ultraleap's Unity SDK so that the Package Manager can resolve the dependency. You can do this by following the steps listed here: https://developer.leapmotion.com/unity#setup-openupm.

You can grab a tarball of our latest Unity SDK Package here [TODO]. You can add this package using the Package Manager:

<img src="https://contact-control-interfaces.github.io/maestro-sdk-docs/Unity/2.6.0/images/package_manager_tarball.png" height="250"/>

&nbsp;

## Included Sample

Once you've installed the package, you can navigate to the "Samples" section within the Package Manager listing to import the Sandbox sample scene:

<img src="https://contact-control-interfaces.github.io/maestro-sdk-docs/Unity/2.6.0/images/package_manager_samples.png" height="250"/>

The sandbox scene has a variety of interactions including picking up objects, pushing buttons, and finger painting.

In order to run the scene, first ensure that you have enabled the rig that corresponds to your target platform and XR setup. There are currently two rigs: one for Ultraleap hand tracking support and another for Meta Quest 1/2 hand tracking support.

<img src="https://contact-control-interfaces.github.io/maestro-sdk-docs/Unity/2.6.0/images/hierarchy_rigs.png" height="250"/>

&nbsp;

## Integrating Maestro / Usage

The easiest way to add Maestro compatibility to a Unity scene is by utilizing the provided prefabs: [MaestroUltraleapRig] and [MaestroOculusRig]. These can be found under `Contact CI Unity SDK/Runtime/Core/Prefabs/` in the `Packages` folder after installing from a tarball. Simply drag the appropriate prefab into the hierarchy.

### Default Haptics

By default, any object that is collidable within the scene will provides haptics when touched. The default haptics can be configured via `MaestroManager`. `MaestroManager` can be found on the root of the provided prefabs.

 <img src="https://contact-control-interfaces.github.io/maestro-sdk-docs/Unity/2.6.0/images/maestro_manager.PNG" height="250"/>&nbsp;

Under `Default Interaction Profile`, you can configure the default haptics and configuration for the scene:
 - `Grab Type` - Currently only `Arcade` is available; more to come soon!
 - `Interactables Only` - Disables these default haptic settings. Only objects that have a `MaestroInteractable` will have haptics when touched.
 - `Amplitude` - Controls the strength of force feedback. The value ranges from `0` to `255`, with `0` being no pull and `255` being full force feedback.
 - `Vibration Effect` - The vibration effect to be applied.
 - `Vibration Options` - Options corresponding to the selected vibration effect. Typically the strength of the effect.

### MaestroInteractable

To customize the haptics for a given collidable gameobject, utilize the `MaestroInteractable` script. The fields available will look very similar to those found on `MaestroManager` ([documented here](#default-haptics)).

 <img src="https://contact-control-interfaces.github.io/maestro-sdk-docs/Unity/2.6.0/images/interactable_haptics.png" height="250"/>

You can also control how an object can be manipulated using the `Type` field under Configuration:
  - `Static` - Unable to be picked up
  - `One hand grab` - Can be picked up using either individual hand
  - `Two hand` - Requires both hands to pick up

This script also defines a few events that may be used to easily create complex interactions:
- Events
  - `On Grab` - Called when after the object is grabbed
  - `On Release` - Called when after the object is released
- Finger events
  - `On Touch` (FingerCollider) - Called when a finger touches the object
  - `Un Touch` (FingerCollider) - Called when a finger stops touching the object
  - `While Touch` (FingerCollider) - Called while a finger touches the object

&nbsp;

## Troubleshooting

#### Running my scene causes Unity to terminate!

It should be noted that the DLLs used by our SDK are 64-bit only. As such, if the Unity platform settings are set to `Any CPU` or `x86` execution will halt.

To address this, you may find all DLLs under `Contact CI Unity SDK/Runtime/Core/_DLLs`. Set each DLL's CPU setting to `x86_64`. If you are performing builds, this will also have to be done so the DLLs are included with 64-bit builds, and excluded for 32-bit builds.

You may also have to enable `Load on startup` under `Plugin load settings`. The Unity editor will have to be restarted for this to take effect.

#### My Quest `.apk` doesn't have hand tracking! Starting the app gives a "Switch to Controllers" prompt.

In the headset itself, be sure you have hand tracking enabled in your settings, under **Hands and Controllers**.

In Unity, make sure your Oculus project settings don't have **Hand Tracking Support** set to `Controllers Only`. Setting it to `Hands Only` is recommended. These settings can be found on the main `OVRManager` component in the scene or under `Assets/Oculus/OculusProjectConfig.asset`.

#### An error occurred while resolving packages: . . . com.contactci.unity has invalid dependencies or related test packages: com.ultraleap.tracking (dependency): Package [com.ultraleap.tracking@5.11.0] cannot be found

You need to add the scoped registry for Ultraleap hand tracking support, [see Setup above](#setup).

#### error CS0246: The type or namespace name 'OVRManager' could not be found (are you missing a using directive or an assembly reference?)

You need to import [the Oculus Integration](https://assetstore.unity.com/packages/tools/integration/oculus-integration-82022) for full Quest hand tracking support.

<strong>Please contact us at support@contact.ci with any questions.</strong>
