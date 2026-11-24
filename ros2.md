# Maestro ROS 2 Guide

Contact CI provides two ROS 2 packages for driving a Maestro glove from your own ROS 2 system:

- `contactci_msgs`: the message definitions, `FingerHaptic`, `HapticCommand` and `HapticApplied`. It is kept separate so your packages can depend on the messages without the glove driver.
- `contactci_ros`: one node, `glove_node`, that drives one glove. Your robot-side process publishes haptic commands on a ROS topic; the node validates each one, writes it to the glove, and releases the glove when commands stop arriving.

The node only carries haptic commands to the glove. Read [What the node does not provide](#what-the-node-does-not-provide) before you design around it.

**Status:** the packages build and pass their tests on both supported distros, but they have not yet been run against a real glove. Every test so far uses a fake driver.

## Contents

- [Supported platforms](#supported-platforms)
- [Installing](#installing)
- [What a real glove needs](#what-a-real-glove-needs)
- [Trying the node without a glove](#trying-the-node-without-a-glove)
- [Running the glove node](#running-the-glove-node)
- [A minimal publisher](#a-minimal-publisher)
- [Topics and services](#topics-and-services)
- [Messages](#messages)
- [Parameters](#parameters)
- [Safety behaviour](#safety-behaviour)
- [Clocks](#clocks)
- [What the node does not provide](#what-the-node-does-not-provide)
- [Diagnostics](#diagnostics)
- [Licence](#licence)

## Supported platforms

| ROS 2 | Ubuntu | Status |
|---|---|---|
| Jazzy Jalisco | 24.04 | Primary target |
| Humble Hawksbill | 22.04 | Supported until its end of life, May 2027 |

Architecture: x86-64 only.

The glove node and the processes that publish to it must run the same ROS 2 distribution. Humble and Jazzy processes often see each other, but ROS does not support mixing them, and Jazzy advertises type hashes (a fingerprint of each message definition) that Humble does not. If your robot stack runs Jazzy, run the Jazzy build.

The node does not depend on a particular RMW (the middleware ROS 2 uses to carry messages between processes). It is tested under `rmw_fastrtps_cpp`, the default on both distros, and `rmw_cyclonedds_cpp`. Both are DDS implementations; DDS is the publish and subscribe standard those RMWs are built on.

## Installing

Contact CI provides each release as a source archive, `contactci-ros2_<version>.tar.gz`, together with a checksum file, `SHA256SUMS.txt`. There are no `.deb` packages of the ROS packages: you build the archive on the machine that runs the node, with two standard ROS 2 tools:

- `rosdep` installs the system packages that ROS packages depend on.
- `colcon` builds ROS packages. It builds in a workspace: a directory whose `src` subdirectory holds the package sources, and into which `colcon` writes `build`, `install` and `log` directories.

1. **Install ROS 2.** Install Jazzy on Ubuntu 24.04, or Humble on Ubuntu 22.04, from the [official Debian packages](https://docs.ros.org/en/jazzy/Installation/Ubuntu-Install-Debs.html), plus `ros-dev-tools`, which provides `colcon` and `rosdep`. Use the distro your robot stack runs.

2. **Check the download and unpack it.** With the archive and `SHA256SUMS.txt` in the current directory, check the checksum, then unpack the archive into the `src` directory of a workspace:

   ```bash
   sha256sum --check --ignore-missing SHA256SUMS.txt
   mkdir -p ~/contactci_ws/src
   tar -xzf contactci-ros2_*.tar.gz -C ~/contactci_ws/src
   ```

3. **Install the dependencies and build:**

   ```bash
   cd ~/contactci_ws
   source /opt/ros/jazzy/setup.bash    # or /opt/ros/humble/setup.bash
   sudo rosdep init                    # once per machine; skip if already done
   rosdep update
   rosdep install --from-paths src --ignore-src -y
   colcon build
   ```

   Every dependency of the two packages is a rosdep key (a name `rosdep` turns into an apt package), and all of them exist for both distros, so the one `rosdep install` line installs everything the packages need from ROS.

4. **Install the glove software.** It is not ROS software, so `rosdep` does not install it. See [What a real glove needs](#what-a-real-glove-needs). Skip this step to try the node without a glove.

5. **In every new terminal**, before any `ros2` command, source the workspace (load its setup script into the shell, so ROS finds the packages you built):

   ```bash
   source ~/contactci_ws/install/setup.bash
   ```

The archive also contains `BUILD_INFO.txt`, which gives the version and the commit the archive was built from. Quote both when you report a problem; the node reports the same two when it runs (see [Diagnostics](#diagnostics)).

**Upgrading.** Delete `~/contactci_ws/src/contactci-ros2-*` and the `build`, `install` and `log` directories of the workspace, then repeat steps 2 and 3 with the new archive.

## What a real glove needs

The node's default driver writes to the glove through the `contactci` Python binding, which loads the Contact CI Core library, `libccic.so`, and talks to the Contact CI Linux service. None of these is a ROS package. On the machine the node runs on:

- The Linux service is installed and running, with the glove connected to it. It is the `contactci-service` package in a Contact CI for Linux release; take the files for the machine's Ubuntu release.
- `libccic.so.3` is where the dynamic loader finds it, or the environment variable `CONTACTCI_LIBRARY` holds its path. It is in the `libcontactci-core3` package of the same release. Install the service and Core from one release: a mismatched pair refuses to connect.
- The user that runs the node is in the `contactci` group, which the service package creates. A user outside the group is refused access to the service:

  ```bash
  sudo usermod -aG contactci $USER    # then log out and back in
  ```

- The `contactci` Python package is importable by the system Python that ROS uses. Contact CI builds it as the `python3-contactci` package, which is not in a release yet. To check:

  ```bash
  python3 -c "import contactci"
  ```

Without these the node still starts, but `configure` fails and the node logs the reason.

## Trying the node without a glove

Set `driver: fake` in the parameter file (see [Parameters](#parameters)). The node then opens no glove and needs neither the Python binding nor the Linux service: commands are accepted and echoed on the `applied` topic, and nothing moves. The node logs a warning when the fake driver is in use.

## Running the glove node

`glove_node` drives one glove; run one node per glove. It is a lifecycle node: a ROS 2 node with managed states that you move it through with explicit transitions. It starts unconfigured, opens the glove at `configure`, and accepts commands only while active.

**1. Write a parameter file.** Copy `config/glove.yaml` from the `contactci_ros` package (after the build it is at `~/contactci_ws/install/contactci_ros/share/contactci_ros/config/glove.yaml`) and set the two values it leaves unset, `max_hold_ms` and `max_command_age_ms`. They have no default on purpose, and the node refuses to configure without them. See [Parameters](#parameters).

**2. Launch the node.** From the workspace:

```bash
source install/setup.bash
ros2 launch contactci_ros glove.launch.py side:=right params_file:=/path/to/glove.yaml
```

| Launch argument | Default | Meaning |
|---|---|---|
| `side` | none; required | `left` or `right`: which glove this node drives |
| `namespace` | `/maestro/<side>` | Namespace the node runs under (a prefix for its topic and service names) |
| `params_file` | the installed `config/glove.yaml` | Parameter file; the installed one will not configure as it is |

**3. Configure and activate it.** In a second terminal:

```bash
source install/setup.bash
ros2 lifecycle set /maestro/right/glove configure
ros2 lifecycle set /maestro/right/glove activate
ros2 topic echo /maestro/right/glove/applied
```

The node now accepts commands on `/maestro/right/glove/command`.

To turn every finger off at any time:

```bash
ros2 service call /maestro/right/glove/release std_srvs/srv/Trigger
```

Deactivating the node, shutting it down or stopping it with Ctrl-C does the same before the glove is closed.

Parameters are read at `configure`. To change one, take the node through `cleanup` first, then `configure` again.

## A minimal publisher

This script publishes a `HapticCommand` for the right glove 10 times a second, turning on one vibration effect on the index finger. It uses only the Python standard library, `rclpy` and `contactci_msgs`, and runs unchanged on Jazzy and Humble.

```python
"""Turn on one vibration effect on the index finger of the right glove."""
import time

import rclpy
from rclpy.executors import ExternalShutdownException
from rclpy.node import Node

from contactci_msgs.msg import FingerHaptic, HapticCommand

# Must be an effect code that your glove supports (1 to 255).
VIBRATION_EFFECT_CODE = 1

TOPIC = '/maestro/right/glove/command'


class MinimalPublisher(Node):

    def __init__(self):
        super().__init__('minimal_haptic_publisher')
        # A new source_id on every start: the node name plus the start time.
        self.source_id = f'{self.get_name()}-{time.time_ns()}'
        self.sequence = 0
        self.publisher = self.create_publisher(HapticCommand, TOPIC, 10)
        self.timer = self.create_timer(0.1, self.publish_command)

    def publish_command(self):
        self.sequence += 1

        finger = FingerHaptic()
        finger.finger = FingerHaptic.INDEX
        finger.vibration_mode = FingerHaptic.VIBRATION_EFFECT
        finger.vibration_effect = VIBRATION_EFFECT_CODE

        command = HapticCommand()
        command.schema_version = HapticCommand.SCHEMA_VERSION
        command.header.stamp = self.get_clock().now().to_msg()
        command.source_id = self.source_id
        command.sequence = self.sequence
        command.side = HapticCommand.SIDE_RIGHT
        command.fingers = [finger]
        self.publisher.publish(command)


def main():
    rclpy.init()
    node = MinimalPublisher()
    try:
        rclpy.spin(node)
    except (KeyboardInterrupt, ExternalShutdownException):
        pass
    finally:
        node.destroy_node()
        if rclpy.ok():
            rclpy.shutdown()


if __name__ == '__main__':
    main()
```

Save it as `minimal_publisher.py` and run it with `python3 minimal_publisher.py` in a terminal where the workspace is sourced, while the node is active. Every field the script does not set is left at zero, so `hold` is 0 and each command is held for `max_hold_ms`.

While it runs, each accepted command is echoed on `applied` with `reason` `REASON_COMMAND` (0), the script's `source_id`, the index finger set to `VIBRATION_EFFECT` with your effect code, and the other four fingers off. Stop it with Ctrl-C: about `max_hold_ms` later the node releases the glove and echoes one message with `reason` `REASON_EXPIRED` (1), every finger off and `sequence` 0.

This example has been run against the node with the fake driver only, on both Jazzy and Humble, not against a glove. A real publisher should also set the Lifespan QoS of its command publisher, as described under [Clocks](#clocks).

## Topics and services

The node is named `glove`, and its topics and service are private names (names relative to the node), so they appear under `<namespace>/glove`. With the default namespace and `side:=right` they are:

| Name | Kind | Type | QoS |
|---|---|---|---|
| `/maestro/right/glove/command` | subscription | `contactci_msgs/msg/HapticCommand` | Best effort, keep last 1, volatile |
| `/maestro/right/glove/applied` | publisher | `contactci_msgs/msg/HapticApplied` | Reliable, keep last 50, volatile |
| `/maestro/right/glove/release` | service | `std_srvs/srv/Trigger` | |
| `/diagnostics` | publisher | `diagnostic_msgs/msg/DiagnosticArray` | Reliable, keep last 1, volatile; once a second |

QoS (quality of service) is the set of delivery settings a ROS 2 publisher and subscription each declare, such as reliability and queue depth. A best-effort subscription matches both reliable and best-effort publishers, so your command publisher connects whichever reliability it uses.

After a call to `release`, commands are accepted again from the next message.

## Messages

All command values are dimensionless numbers in [0, 1]. There are no newtons anywhere in this interface. Haptic commands carry no spatial data: `header.frame_id` is unused and the node publishes no TF (ROS coordinate frames).

### FingerHaptic

The desired output for one finger. A command, not a measurement.

| Field | Type | Values |
|---|---|---|
| `finger` | `uint8` | `THUMB`=0, `INDEX`=1, `MIDDLE`=2, `RING`=3, `LITTLE`=4 |
| `force_mode` | `uint8` | `FORCE_OFF`=0; `FORCE_AMPLITUDE`=1, normalized tendon resistance, open loop; `FORCE_POSITION`=2, reserved and rejected |
| `force_value` | `float32` | [0, 1] for `FORCE_AMPLITUDE`; must be 0 for `FORCE_OFF` |
| `vibration_mode` | `uint8` | `VIBRATION_OFF`=0; `VIBRATION_EFFECT`=1, one of the driver's built-in effects; `VIBRATION_AMPLITUDE`=2, reserved and rejected |
| `vibration_effect` | `uint8` | Driver effect code, 1 to 255; 0 is not an effect |
| `vibration_modifier` | `uint8` | Driver effect modifier byte |
| `vibration_amplitude` | `float32` | [0, 1]; ignored unless `VIBRATION_AMPLITUDE`, which is reserved |

Every field of a force or vibration that is off must be zero.

### HapticCommand

The complete desired haptic state for one glove. You publish this on `command`.

| Field | Type | Values |
|---|---|---|
| `schema_version` | `uint32` | Must be `SCHEMA_VERSION`, which is 1 |
| `header` | `std_msgs/Header` | `stamp`: the publisher's ROS time when it produced the command. `frame_id`: unused |
| `session_id` | `string` | Recording session |
| `source_id` | `string` | Unique commanding process; must not be empty |
| `sequence` | `uint64` | Strictly increasing per `source_id` |
| `side` | `uint8` | `SIDE_LEFT`=0, `SIDE_RIGHT`=1; must match the node's glove |
| `hold` | `builtin_interfaces/Duration` | How long to keep applying this command after it arrives. 0, or anything longer than `max_hold_ms`, means `max_hold_ms`. Must not be negative |
| `fingers` | `FingerHaptic[]` | 0 to 5 entries, each finger at most once. A finger not listed is off |
| `config_id` | `string` | Bounded configuration identifier |
| `origin_ids` | `string[]` | IDs of the robot contact samples that caused this command |

**`source_id` and `sequence`.** Use a new `source_id` every time your publisher starts, for example its node name followed by its start time. The node remembers the highest `sequence` it has seen from each `source_id` from `configure` until `cleanup`, so a publisher that restarts under its old `source_id` and counts from 0 again has every command rejected until its count passes the old one. A publisher that cannot change its `source_id` must carry its `sequence` on across the restart.

### HapticApplied

What the node wrote to the glove driver, and why. The node publishes it on `applied`. It is a command echo, not glove telemetry: it is not a measurement of what the glove did.

| Field | Type | Values |
|---|---|---|
| `header` | `std_msgs/Header` | `stamp`: the node's ROS time of the write |
| `command_stamp` | `builtin_interfaces/Time` | `header.stamp` of the originating command |
| `receive_stamp` | `builtin_interfaces/Time` | The node's ROS time when that command arrived |
| `source_id` | `string` | Of the originating command |
| `sequence` | `uint64` | Of the originating command; 0 for releases the node started itself |
| `reason` | `uint8` | `REASON_COMMAND`=0, `REASON_EXPIRED`=1, `REASON_RELEASE_SERVICE`=2, `REASON_DEACTIVATED`=3, `REASON_SHUTDOWN`=4, `REASON_DEVICE_LOST`=5 |
| `fingers` | `FingerHaptic[5]` | The state written, one entry per finger in `THUMB` to `LITTLE` order |

`fingers` is read back from the state written, not copied from the command, so a `FORCE_AMPLITUDE` command with value 0 is echoed as `FORCE_OFF`. The all-off write the node makes at `activate` is not echoed. Rejected commands are not echoed, because nothing was written.

To see a definition on your machine:

```bash
ros2 interface show contactci_msgs/msg/HapticCommand
```

## Parameters

Set these in the parameter file. The node reads them at `configure`; they cannot be changed while it is active.

| Parameter | Type | Default | Notes |
|---|---|---|---|
| `side` | string | required | `left` or `right`. The launch file sets it from its `side` argument |
| `device_serial` | string | `""` | Empty selects the one connected glove on `side`; the node refuses to guess between two |
| `max_hold_ms` | int | required; no default | Milliseconds. Longest time a command is applied without a newer one |
| `max_command_age_ms` | int | required; no default | Milliseconds. Oldest command, by `header.stamp`, the node accepts. Must exceed the measured publisher-to-node latency plus the clock offset bound (see [Clocks](#clocks)) |
| `watchdog_period_ms` | int | `10` | Milliseconds. Period of the check for an expired command and for a lost glove |
| `force_enabled` | bool | `false` | Force commands are rejected unless this is true. See [Force output](#force-output) |
| `max_force_amplitude` | double | `0.0` | Largest force value accepted, in [0, 1]. A command that asks for more is rejected, not reduced to this |
| `allowed_vibration_effects` | int[] | unset | Vibration effect codes that are accepted; any other is rejected. Unset allows any effect code 1 to 255 |
| `driver` | string | `contactci` | `contactci`: the glove, through the `contactci` Python binding and the Linux service. `fake`: no glove; commands are accepted and echoed, nothing moves |

## Safety behaviour

Design your publisher around these rules.

### Every command is the full state

Every `HapticCommand` is the complete state for the glove and replaces the previous one entirely. A finger it does not list is turned off, and a command with an empty `fingers` list releases the glove. So a lost message is corrected by the next one, and you must send every finger you want on in every command.

### Hold and expiry

An accepted command is applied for its `hold`, measured from when it arrived. A `hold` of 0, or one longer than `max_hold_ms`, means `max_hold_ms`. If no newer command is accepted in that time, the node releases the glove and echoes `REASON_EXPIRED`. To keep feedback on, keep publishing.

Expiry runs on the node's steady clock (a clock that only moves forward and is not affected by ROS time), even with `use_sim_time`, so simulated time cannot stretch a hold. It is checked every `watchdog_period_ms`, so the release comes at most `max_hold_ms` plus `watchdog_period_ms` after the command arrived, plus the driver write.

### Command age

The node rejects a command that is stale (older than `max_command_age_ms`), future-dated (stamped more than `max_command_age_ms` ahead of the node's clock) or unstamped. Without this check, a command delayed in the network would turn feedback back on after the contact that caused it had ended. See [Clocks](#clocks).

### Commands the node rejects

A rejected command changes nothing. It is not written to the glove and not echoed on `applied`; the command before it stays applied and expires when it would have. The node counts the rejection and gives the reason on `/diagnostics`.

A command is rejected when:

- `schema_version` is not `1`, `side` is not this node's glove, or `source_id` is empty;
- a finger is listed twice, or a `finger`, `force_mode` or `vibration_mode` is out of range;
- it uses a reserved mode, `FORCE_POSITION` or `VIBRATION_AMPLITUDE`;
- `force_value` or `vibration_amplitude` is NaN, infinite or outside [0, 1], or any field of a force or vibration that is off is not zero;
- it asks for force while `force_enabled` is false, or for more than `max_force_amplitude`. A force above the limit is rejected, not reduced to it;
- it asks for vibration effect 0, or for an effect that is not in `allowed_vibration_effects` when that list is set;
- its `sequence` is not higher than the highest the node has seen from that `source_id` since `configure`;
- `hold` is negative;
- it is stale or future-dated.

### When the glove is released

The glove is released, every finger off, on each of these. All but the last are echoed on `applied` with the reason shown:

| Release | `reason` on `applied` |
|---|---|
| The command expired | `REASON_EXPIRED` |
| The `release` service | `REASON_RELEASE_SERVICE` |
| `deactivate` | `REASON_DEACTIVATED` |
| `cleanup` or `shutdown` | `REASON_SHUTDOWN` |
| The glove was lost | `REASON_DEVICE_LOST` |
| SIGINT or SIGTERM | none; ROS is shut down by then, so nothing can be published. The node logs `the glove is released and the session is closed` |

**When the glove is lost.** The glove counts as lost when the driver reports it disconnected, or when a write or a release fails. The node releases it, deactivates itself and reports an error on `/diagnostics`. It does not go back to active when the glove returns, so feedback cannot resume unattended: activate it again, which it refuses while the glove is away. If the release fails as well, the node closes its session, which makes the Linux service clear the glove, and goes back to unconfigured.

**If the node is killed outright** (SIGKILL, a crash), none of the above runs. Release then depends on the Linux service clearing what the node last wrote when its connection closes, and on the glove firmware. Some glove firmware currently holds the last state when the host disappears, so a crash of the node or the host can leave feedback on.

### Force output

Force commands are rejected unless `force_enabled` is true, and it is false by default; `max_force_amplitude` defaults to `0.0`. Do not set `force_enabled` to true until Contact CI has signed off force output: force opposition is the hazardous mode. Until then, use vibration only.

`max_hold_ms`, `max_command_age_ms` and `max_force_amplitude` are bench-test values. This page gives no values for them.

## Clocks

The node compares `header.stamp` of every command with its own ROS clock when the command arrives. So the publisher and the node must stamp from the same clock:

- On one machine they do.
- Across machines, both hosts must run chrony or PTP (two standard ways to synchronize a computer's clock to a time source) against the same source. Record the bound on the offset between the two clocks, and set `max_command_age_ms` above the measured publisher-to-node latency plus that bound.
- With `use_sim_time`, both read `/clock`, so a bag (a ROS recording) replays as it was recorded.

Accepted commands therefore act for at most `max_command_age_ms` plus `max_hold_ms` after the publisher stamped them.

Also set the Lifespan QoS of your command publisher to `max_command_age_ms`, so that DDS drops a sample that is already too old on its way. That is a second line only: it is the publisher's setting and relies on the same clocks; the check in the node is the one that is enforced.

`/diagnostics` gives the number of stale and of future-dated rejections, the age of the last command and the largest age seen. A largest age that creeps towards `max_command_age_ms` is a clock drifting, before anything is rejected.

## What the node does not provide

The node carries haptic commands to the glove and nothing else. It does not provide:

| Not provided | Why |
|---|---|
| Operator hand pose or joint angles | The glove has no pose sensing. Hand tracking comes from the camera tracking stack on a Windows host (Meta XR, Ultraleap), which does not run on Linux |
| Glove telemetry (actuator position, current, measured force) | No host software on any operating system requests or reads it yet. `HapticApplied` is an echo of the command written, not a measurement |
| Contact or tactile data | The glove has no fingertip sensor. Robot contact comes from your robot's own stack |
| Force in newtons | Force commands are normalized and open loop. A haptic command is not a force on the operator |
| Retargeting or robot control | Stays on the robot side, in your robot's existing controller |

## Diagnostics

`glove_node` publishes one status on `/diagnostics` every second, named after the node. Its level is ERROR when a transition failed, the glove is not connected or the glove was lost, and WARN when more than one `source_id` has sent commands for this glove in the last second.

Its values: lifecycle state, connected, side, serial number, product line, service version, driver, active `source_id`s, commands accepted, commands rejected, last rejection reason, stale rejections, future-dated rejections, largest and last command age, expiries, driver write errors, seconds since the last accepted command, and the five build values below. The counts start again at `configure`.

### Which build is running

The node logs one line when it starts, for example:

```text
contactci_ros version 0.1.0-rc.1, commit 0123456789abcdef0123456789abcdef01234567, dirty false, build 1234567890; ROS jazzy, Python 3.12.3
```

and gives the same on `/diagnostics`, so a bag that includes `/diagnostics` keeps it:

| Value | Meaning |
|---|---|
| `node version` | The full version of the build, for example `0.1.0-rc.1` |
| `node commit` | The git commit the build was made from, all 40 characters |
| `node build dirty` | `true` when tracked files had uncommitted changes, so the commit alone does not give the code back. Use such a build on the bench only, never to record data |
| `node build id` | The CI (automated build) run that made a release archive; `local` for a build that CI did not make |
| `toolchain` | The ROS distro and the Python version the node runs on |

A release archive carries this record in `BUILD_INFO.txt`. A build that has no record still works; the node then reports `unknown` for what it cannot know.

## Licence

The packages are distributed under the [Contact CI SDK EULA](https://contactci.co/pages/sdk-eula.html), which each release archive and each package also carries as `EULA.md`.
