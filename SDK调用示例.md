# SDK 调用示例（报文级）

> 本文只给"手写 `ros2 topic pub` 要用的完整 JSON"，每个接口一段，可直接复制。
>
> **流程怎么走、每步看什么、失败怎么处理** 见 [SDK 全流程操作手册](./SDK全流程操作手册.md)
> （含 §0.3 一页纸最短流程、§11.1 命令全表、§4.4 批量验证点位）。
> 字段含义与错误码见 [SDK 接口文档](./SDK接口文档.md)。
>
> 最后修改时间：2026-11-16（接口 1801~1822）
> 所有示例均假定已 `source /opt/app/slam/.bashrc`；SDK 已通过 `bash scripts/launch_sdk.sh` 启动。
> **English:** [SDK Examples](./en/SDK_EXAMPLES.md)

## /slam/init_pose（1804，启动定位与导航）

### 自动估计位姿

```bash
SCENE_ID=$(basename "$(realpath /opt/app/slam/data/maps/out)")   # 最近一次结束建图保存的场景
JSON=$(printf '{"request_id":"test_init_pose","auto_init_pose":true,"scene_id":"%s"}' "$SCENE_ID")
ros2 topic pub --once --keep-alive 1 /slam/init_pose std_msgs/msg/String "{data: '$JSON'}"
```

### 手动位姿（欧拉角）

```bash
SCENE_ID=$(basename "$(realpath /opt/app/slam/data/maps/out)")
JSON=$(printf '{"request_id":"test_init_pose","auto_init_pose":false,"manual_pose":{"use_euler":true,"position":{"x":1.5,"y":2.0,"z":0.0},"euler_angle":{"r":0.0,"p":0.0,"y":1.57}},"scene_id":"%s"}' "$SCENE_ID")
ros2 topic pub --once --keep-alive 1 /slam/init_pose std_msgs/msg/String "{data: '$JSON'}"
```

### 手动位姿（四元数）

```bash
SCENE_ID=$(basename "$(realpath /opt/app/slam/data/maps/out)")
JSON=$(printf '{"request_id":"test_init_pose","auto_init_pose":false,"manual_pose":{"use_euler":false,"position":{"x":1.5,"y":2.0,"z":0.0},"orientation":{"q_x":0.0,"q_y":0.0,"q_z":0.7071,"q_w":0.7071}},"scene_id":"%s"}' "$SCENE_ID")
ros2 topic pub --once --keep-alive 1 /slam/init_pose std_msgs/msg/String "{data: '$JSON'}"
```

## /slam/nav_to_pose（1806，导航到目标点）

### 绝对位姿（map 坐标系）

```bash
JSON='{"request_id":"test_nav_to_pose","relative":false,"target_pose":{"use_euler":true,"position":{"x":5.0,"y":0.0,"z":0.0},"euler_angle":{"r":0.0,"p":0.0,"y":1.57}}}'
ros2 topic pub --once --keep-alive 1 /slam/nav_to_pose std_msgs/msg/String "{data: '$JSON'}"
```

### 相对位姿（ego 坐标系，前进 2 米）

```bash
JSON='{"request_id":"test_nav_to_pose_relative","relative":true,"target_pose":{"use_euler":true,"position":{"x":2.0,"y":0.0,"z":0.0},"euler_angle":{"r":0.0,"p":0.0,"y":0.0}}}'
ros2 topic pub --once --keep-alive 1 /slam/nav_to_pose std_msgs/msg/String "{data: '$JSON'}"
```

## /slam/pause_nav（1807，暂停导航）

```bash
JSON='{"request_id":"test_pause_nav"}'
ros2 topic pub --once --keep-alive 1 /slam/pause_nav std_msgs/msg/String "{data: '$JSON'}"
```

## /slam/resume_nav（1808，恢复导航）

```bash
JSON='{"request_id":"test_resume_nav"}'
ros2 topic pub --once --keep-alive 1 /slam/resume_nav std_msgs/msg/String "{data: '$JSON'}"
```

## /slam/cancel_nav（1809，取消导航）

```bash
JSON='{"request_id":"test_cancel_nav"}'
ros2 topic pub --once --keep-alive 1 /slam/cancel_nav std_msgs/msg/String "{data: '$JSON'}"
```

## /slam/get_global_pose（1805，机器人当前位姿）

```bash
JSON='{"request_id":"test_get_global_pose"}'
ros2 topic pub --once --keep-alive 1 /slam/get_global_pose std_msgs/msg/String "{data: '$JSON'}"
```

## /slam/get_nav_state（1814，导航状态）

```bash
JSON='{"request_id":"test_get_nav_state"}'
ros2 topic pub --once --keep-alive 1 /slam/get_nav_state std_msgs/msg/String "{data: '$JSON'}"
```

## /slam/get_state（1815，SLAM 状态）

```bash
JSON='{"request_id":"test_get_state"}'
ros2 topic pub --once --keep-alive 1 /slam/get_state std_msgs/msg/String "{data: '$JSON'}"
```

## /slam/shutdown（1810，结束定位与导航）

```bash
JSON='{"request_id":"test_shutdown"}'
ros2 topic pub --once --keep-alive 1 /slam/shutdown std_msgs/msg/String "{data: '$JSON'}"
```

## /slam/start_mapping（1801，开始建图）

```bash
JSON='{"scene_id":"ground_floor","slam_type":"indoor","request_id":"test_start_mapping"}'
ros2 topic pub --once --keep-alive 1 /slam/start_mapping std_msgs/msg/String "{data: '$JSON'}"
```

## /slam/stop_mapping（1802，结束建图并保存地图）

```bash
JSON='{"scene_id":"ground_floor","request_id":"test_stop_mapping"}'
ros2 topic pub --once --keep-alive 1 /slam/stop_mapping std_msgs/msg/String "{data: '$JSON'}"
```

## /slam/terminate_mapping（1822，强制终止建图，不保存地图）

```bash
JSON='{"scene_id":"ground_floor","request_id":"test_terminate_mapping"}'
ros2 topic pub --once --keep-alive 1 /slam/terminate_mapping std_msgs/msg/String "{data: '$JSON'}"
```

## /slam/restart_odom（1816，里程计重启）

### relocalize=true（默认，重启后做限定范围全局重定位）

```bash
JSON='{"request_id":"test_restart_odom","scene_id":"ground_floor","relocalize":true}'
ros2 topic pub --once --keep-alive 1 /slam/restart_odom std_msgs/msg/String "{data: '$JSON'}"
```

### relocalize=false（确信机器人未移动，跳过全局重定位）

```bash
JSON='{"request_id":"test_restart_odom","scene_id":"ground_floor","relocalize":false}'
ros2 topic pub --once --keep-alive 1 /slam/restart_odom std_msgs/msg/String "{data: '$JSON'}"
```

> 响应订阅、状态查看、快捷函数表、`slam_cycle_test` 用法都已收敛到
> [SDK 全流程操作手册](./SDK全流程操作手册.md) 的 §7.1、§11.1、§4.4，本文不重复。

