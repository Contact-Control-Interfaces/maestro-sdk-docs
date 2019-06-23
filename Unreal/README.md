# Maestro Unreal SDK Guide

## This guide is for Maestro devices using external tracking systems. See [this guide](./Alpha_README.md) for the Alpha models

In order to use the Maestro Glove in Unreal Engine 4, the Maestro Glove Unreal plugin must be used. 

[The plugin can be downloaded from here](https://github.com/Contact-Control-Interfaces/maestro-sdk-unreal/releases/tag/v0.2.1). 

Additionally, this plugin is set up to support the official LeapMotion plugin as the current Maestro version has no internal tracking system. Note that the plugin is built to support any tracking system, and merely hooks onto a driven hand skeleton, and as such other hand tracking systems may be used.

This plugin was compiled for version 4.20.3, so [the recommended LeapMotion plugin version is here.](https://github.com/leapmotion/LeapUnreal/releases/tag/v3.0.1)

[Full Unreal plugin documentation can be found here](https://docs.unrealengine.com/latest/INT/Programming/Plugins/).

## Plugin Set-up
Before the plugin can be installed, verify that the following prerequisites are met:

+ **Plugin engine version matches project engine version.**
> This can be fixed in two ways: changing the version of the project or changing the version of the plugin. 
> 
> Changing the version of the project can be done by right-clicking that project’s `.uproject` file and selecting “Switch Unreal Engine version…” followed by selecting the desired engine version.
> 
> Changing the engine version of the plugin requires it to be recompiled. This can be done a variety of ways. If the project happens to be a C++ project, the plugin may be automatically recompiled upon loading, so extra steps are not necessary. Otherwise, recompilation will have to be done manually. To recompile the plugin for a different engine version, first create a new C++ project of the desired engine version. Follow the [Plugin Installation instructions below](#plugin-installation) for installing the plugin in that project. Generate the new project’s project files by right-clicking on its `.uproject` file and selecting “Generate Visual Studio project files.” The newly generated `.sln` file with be able to recompile the plugin. By opening the solution in Visual Studio and performing a build, the plugin/project source will be recompiled into their respective binaries. Once the build succeeds, that project’s version of the Maestro Glove Unreal plugin will be compatible with the project’s engine version, and can be moved/copied to compatible projects as desired.

+ **Main plugin directory is unchanged.**
> The plugin will fail to start up if the main plugin folder “MaestroGlove” is renamed. Renaming any of the subfolders therein may also cause the plugin to fail to start, and is discouraged for this reason.

+ **Plugin contains Maestro Glove library files.**
> The Binaries folder should contain `libMaestro.dll` and `libMaestro.dll.a`, both of which are necessary at runtime for the plugin. Renaming these files or removing them will cause the plugin to fail to start.

## Plugin Installation
To install the plugin for a given project, place the Maestro Glove plugin in the `Plugins` folder in your project’s main directory. If that folder does not exist, create it first. The next time the project’s editor is opened the plugin will be loaded and ready to use. This can be verified by opening the Plugins Browser in editor by clicking on Plugins under the Edit dropdown on the main toolbar. If the plugin was loaded properly, it should be listed under Installed. Verify that the enabled checkbox has been clicked for the project here before continuing.

It is also possible to install the plugin on the specific version of UE4 itself if desired. If the plugin is placed into the `Plugins` folder for the Engine itself, usually found under `Program Files\Epic Games\UE_[VERSION]\Engine`, the plugin will startup and be available for use in any Unreal project using that version.

## Getting Started with the Plugin
The main objective of the plugin is to make it easier for developers and designers alike to interface with the Maestro Glove. For this reason, the plugin includes a Blueprint Function Library that exposes all of our C API functions to the Blueprint Editor under the Maestro Glove category. It also includes several Blueprints that should be useful when attempting to interface with the Maestro. In order to be able to see the content included with the plugin the “Show Plugin Content” checkbox under the View Options dropdown in the Content Browser must be selected. The included content is detailed in the [Useful Blueprints section below](#useful-blueprints).

If you are developing an application using C++ rather than Blueprint scripting, it may be more useful to simply include both of the C headers, `maestro.h` and `maestro_types.h`, and use those exposed functions directly. These headers are stored in the `Source\Maestro\Includes` folder of the plugin. As the included Blueprint Function Library is simply a wrapper for the C API, using the headers directly can eliminate some unnecessary overhead.

It should be noted that the plugin calls `start_maestro_detection_service` on start-up, so it is unnecessary to call this function to get the glove to connect provided the plugin is enabled and starts up properly.

## Example Map
A sample map is provided in the `Examples` directory of the plugin, and features a variety of different custom interactions to illustrate how to use the Maestro Unreal SDK.

## Useful Blueprints
The list below is not meant to be a complete and comprehensive list of all included content with the plugin, but instead is meant to detail the purpose and member variables of a few of the ones that are most likely to be used. The blueprints below are also meant as an example for how to use the plugin.

#### MaestroInteractable
Actor component used by anything that should have Maestro haptics. May be extended for more complex haptic effects.
> **Use Default Values** - Should I ignore all these values and just use the [MaestroComponent](#maestrocomponent) defaults?
>
> **Pull Amplitude** - Pull amplitude used on overlap by default
>
> **Vibration Effect** - Vibration effect used on overlap by default
>
> **Pull Curve** - Curve defining amplitudes to play on overlap. Overrides Pull Amplitude.
>
> **Vibration Curve** - Curve defining vibration effects to play on overlap. Overrides Vibration Effect.
>
> **Loop Pull Curve** - Should the pull amplitude curve be played continuously?
>
> **Loop Vibration Curve** - Should the vibration effect curve be played continuously?

This blueprint contains two functions that may be overridden to provide more complex effects. See `Blueprints\Examples` for examples.
> **Get Pull Amplitude** - Called by a [FingertipCollider](#fingertipcollider) to decide what pull amplitude to play.
>
> **Get Vibration Effect** - Called by a [FingertipCollider](#fingertipcollider) to decide what vibration effect to play.

#### MaestroLeapPawn
Leap VR pawn set up to use Maestro components for each hand. An example of how to use the Maestro with a driven skeletal mesh.
> **Left Component** - [MaestroComponent](#maestrocomponent) spawned on the left Leap hand.
>
> **Right Component** - [MaestroComponent](#maestrocomponent) spawned on the right Leap hand.

#### MaestroComponent
The Blueprint enables haptic responses on overlap with a skeletal mesh's fingertips/palm. For this reason, all actors in the scene need to have overlap events enabled in order for the blueprint to react to them. All instance variables under the Maestro category are necessary for the blueprint to function properly, and their purposes are as follows:
> **Which Hand** - Whether this hand is a left hand or a right hand. Blueprint uses this to decide which glove to retrieve data from.
> 
> **Haptic Mode** - Haptic mode for object interaction. Use OnAll to activate haptics on all overlaps, use OnInteractables to activate haptics only on Maestro Interactables.
>
> **Vibration Effect** - Byte representing the default strength of the vibration played in the fingertips during an overlap, `0` being no vibration and `128` being full vibration.
> 
> **Pull Amplitude** - Byte representing the default amplitude of the force feedback motors during an overlap, `0` being no pull and `255` being full pull. It is recommended to always set this to somewhere in the approximate range `40`-`200`. Values above about `200` are clamped down to `200`. This maximum value may change at our discretion.
>
> **Socket Names** - Ordered list of socket names on the target skeleton (Thumb, Index, ..., Palm). Used to spawn and attach fingertip colliders.
>
> **Finger Size** - Diameter of colliders used on the fingertips.
>
> **Palm Size** - Size of the collider used on the palm.
>
> **Show Colliders** - Whether or not to show the colliders used for haptics. Change color on overlap.

#### FingertipCollider
Sphere collider that is attached to the end of a finger or to the palm to enable haptics/pickup. Spawned automatically by [MaestroComponent](#maestrocomponent).
> **Index** - Integer index defining which finger this collider is attached to. `0` is the thumb, `1`-`4` are the fingers, `5` is the palm. Assigned automatically by [MaestroComponent](#maestrocomponent).
>
> **Parent** - Reference to the [MaestroComponent](#maestrocomponent) actor that owns this collider. Used to decide where to get the glove pointer for haptic responses.
>
> **Velocity** - The velocity of this fingertip this frame.
