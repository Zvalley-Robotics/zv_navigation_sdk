# SDK Call Examples

> This document only provides the complete JSON payloads for hand-written `ros2 topic pub` calls — one block per interface, ready to copy.
>
> **How to run the flow, what to watch at each step, and how to handle failures** are in the
> [SDK Full Workflow Guide](./SDK_FULL_WORKFLOW_GUIDE.md) (§0.3 one-page shortest workflow, §11.1 full command list, §4.4 batch waypoint verification).
> Field meanings and error codes are in the [SDK Interface](./SDK_INTERFACE.md).
>
> This is the English version of [SDK调用示例.md](../SDK调用示例.md).
>
> **Last updated:** 2026-11-16 (interfaces 1801~1822)
> All examples assume `source /opt/app/slam/.bashrc` has already been run and the SDK has been started via `bash scripts/launch_sdk.sh`.

## /slam/init_pose (1804, start localization and navigation)

### Automatic pose estimation

```bash
SCENE_ID=$(basename "$(realpath /opt/app/slam/data/maps/out)")   # the scene saved by the most recent finished mapping run
JSON=$(printf '{"request_id":"test_init_pose","auto_init_pose":true,"scene_id":"%s"}' "$SCENE_ID")
ros2 topic pub --once --keep-alive 1 /slam/init_pose std_msgs/msg/String "{data: '$JSON'}"
```

### Manual pose (Euler angles)

```bash
SCENE_ID=$(basename "$(realpath /opt/app/slam/data/maps/out)")
JSON=$(printf '{"request_id":"test_init_pose","auto_init_pose":false,"manual_pose":{"use_euler":true,"position":{"x":1.5,"y":2.0,"z":0.0},"euler_angle":{"r":0.0,"p":0.0,"y":1.57}},"scene_id":"%s"}' "$SCENE_ID")
ros2 topic pub --once --keep-alive 1 /slam/init_pose std_msgs/msg/String "{data: '$JSON'}"
```

### Manual pose (quaternion)

```bash
SCENE_ID=$(basename "$(realpath /opt/app/slam/data/maps/out)")
JSON=$(printf '{"request_id":"test_init_pose","auto_init_pose":false,"manual_pose":{"use_euler":false,"position":{"x":1.5,"y":2.0,"z":0.0},"orientation":{"q_x":0.0,"q_y":0.0,"q_z":0.7071,"q_w":0.7071}},"scene_id":"%s"}' "$SCENE_ID")
ros2 topic pub --once --keep-alive 1 /slam/init_pose std_msgs/msg/String "{data: '$JSON'}"
```

## /slam/nav_to_pose (1806, navigate to a goal point)

### Absolute pose (map frame)

```bash
JSON='{"request_id":"test_nav_to_pose","relative":false,"target_pose":{"use_euler":true,"position":{"x":5.0,"y":0.0,"z":0.0},"euler_angle":{"r":0.0,"p":0.0,"y":1.57}}}'
ros2 topic pub --once --keep-alive 1 /slam/nav_to_pose std_msgs/msg/String "{data: '$JSON'}"
```

### Relative pose (ego frame, move forward 2 meters)

```bash
JSON='{"request_id":"test_nav_to_pose_relative","relative":true,"target_pose":{"use_euler":true,"position":{"x":2.0,"y":0.0,"z":0.0},"euler_angle":{"r":0.0,"p":0.0,"y":0.0}}}'
ros2 topic pub --once --keep-alive 1 /slam/nav_to_pose std_msgs/msg/String "{data: '$JSON'}"
```

## /slam/pause_nav (1807, pause navigation)

```bash
JSON='{"request_id":"test_pause_nav"}'
ros2 topic pub --once --keep-alive 1 /slam/pause_nav std_msgs/msg/String "{data: '$JSON'}"
```

## /slam/resume_nav (1808, resume navigation)

```bash
JSON='{"request_id":"test_resume_nav"}'
ros2 topic pub --once --keep-alive 1 /slam/resume_nav std_msgs/msg/String "{data: '$JSON'}"
```

## /slam/cancel_nav (1809, cancel navigation)

```bash
JSON='{"request_id":"test_cancel_nav"}'
ros2 topic pub --once --keep-alive 1 /slam/cancel_nav std_msgs/msg/String "{data: '$JSON'}"
```

## /slam/get_global_pose (1805, robot's current pose)

```bash
JSON='{"request_id":"test_get_global_pose"}'
ros2 topic pub --once --keep-alive 1 /slam/get_global_pose std_msgs/msg/String "{data: '$JSON'}"
```

## /slam/get_nav_state (1814, navigation state)

```bash
JSON='{"request_id":"test_get_nav_state"}'
ros2 topic pub --once --keep-alive 1 /slam/get_nav_state std_msgs/msg/String "{data: '$JSON'}"
```

## /slam/get_state (1815, SLAM state)

```bash
JSON='{"request_id":"test_get_state"}'
ros2 topic pub --once --keep-alive 1 /slam/get_state std_msgs/msg/String "{data: '$JSON'}"
```

## /slam/shutdown (1810, end localization and navigation)

```bash
JSON='{"request_id":"test_shutdown"}'
ros2 topic pub --once --keep-alive 1 /slam/shutdown std_msgs/msg/String "{data: '$JSON'}"
```

## /slam/start_mapping (1801, start mapping)

```bash
JSON='{"scene_id":"ground_floor","slam_type":"indoor","request_id":"test_start_mapping"}'
ros2 topic pub --once --keep-alive 1 /slam/start_mapping std_msgs/msg/String "{data: '$JSON'}"
```

## /slam/stop_mapping (1802, finish mapping and save the map)

```bash
JSON='{"scene_id":"ground_floor","request_id":"test_stop_mapping"}'
ros2 topic pub --once --keep-alive 1 /slam/stop_mapping std_msgs/msg/String "{data: '$JSON'}"
```

## /slam/terminate_mapping (1822, force-terminate mapping, do not save the map)

```bash
JSON='{"scene_id":"ground_floor","request_id":"test_terminate_mapping"}'
ros2 topic pub --once --keep-alive 1 /slam/terminate_mapping std_msgs/msg/String "{data: '$JSON'}"
```

## /slam/restart_odom (1816, odometry restart)

### relocalize=true (default; perform a range-limited global relocalization after the restart)

```bash
JSON='{"request_id":"test_restart_odom","scene_id":"ground_floor","relocalize":true}'
ros2 topic pub --once --keep-alive 1 /slam/restart_odom std_msgs/msg/String "{data: '$JSON'}"
```

### relocalize=false (you are sure the robot has not moved, so skip the global relocalization)

```bash
JSON='{"request_id":"test_restart_odom","scene_id":"ground_floor","relocalize":false}'
ros2 topic pub --once --keep-alive 1 /slam/restart_odom std_msgs/msg/String "{data: '$JSON'}"
```

> Response subscriptions, state inspection, the shortcut-function table, and `slam_cycle_test` usage
> now live in the [SDK Full Workflow Guide](./SDK_FULL_WORKFLOW_GUIDE.md) (§7.1, §11.1, §4.4) and are not repeated here.
