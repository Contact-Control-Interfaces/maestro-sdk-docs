# Maestro Unity SDK Guide

## This guide is for Maestro devices using external tracking systems. See [this guide](./Alpha_README.md) for the Alpha models

The Unity SDK [can be downloaded here](https://github.com/Contact-Control-Interfaces/maestro-sdk-unity/releases/tag/v0.2).

The SDK contains scripts for controlling the Maestro device, as well as three example scenes demonstrating some interactions.

This version is built to be tracking agnostic, but for the sake of simplicity the current SteamVR plugin is included. This aspect of the included content may be updated through the Unity Asset Store.

## Setup

First you will need to run the [Maestro Alpha installer](https://github.com/Contact-Control-Interfaces/maestro-installer), which will handle creating configuration files needed for the Maestro to work. This is described under "Initial Setup" and "Configuration" [here](https://contact-control-interfaces.github.io/maestro-sdk-docs/C/html/index.html)

Once you have configuration files installed and the Unity Maestro SDK Unity Package imported into a Unity project, an example rig prefab "[MaestroIndexRig]" will be available under `Assets/Maestro/Prefabs`. This is an example configuration set up to allow Maestro haptics while using the Valve Index's motion and finger tracking.

## Functionality

All major haptic functionality is controlled by four scripts: `MaestroHand`, `FingerTipCollider`, `MaestroInteractable`, and `HapticResponse`. In order to have the hand able to pick up an object:
- The `MaestroHand` script must be properly bound to the hand hierarchy, and
- The `MaestroInteractable` script must be placed onto the desired object.

The `FingerTipCollider` script is necessary for pickup to properly occur, however they are spawned by the `MaestroHand` script and as such require no setup. The configuration for the other scripts are as follows:

##### `MaestroInteractable`
Must be placed on any game object that requires haptics.

TODO

##### `FingerTipCollider`
Must be placed on each fingertip and in the center of the palm. Spawned and set up automatically by `MaestroHand`.

<strong>Please contact us at support@contactci.co with any questions.</strong>
