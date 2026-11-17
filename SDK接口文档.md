# 导航相关 SDK 接口文档（Topic 方式）

> **最后修改时间：** 2026-11-16（对应接口编码 1801~1822）
> **服务说明：** 本文描述 `/opt/app/slam` 下部署的 SDK 服务；接口、状态机与错误消息以实际运行版本为准。
> **配套文档：** [SDK 状态机](./SDK状态机.md) · [SDK 调用示例](./SDK调用示例.md) · [SDK 全流程操作手册](./SDK全流程操作手册.md)
> **English:** [SDK Interface Reference](./en/SDK_INTERFACE.md)

## 概述

本文档梳理了导航相关 SDK 的 Topic 接口，涵盖建图、定位、导航及状态查询等功能模块。所有接口均通过发布 `/slam/xxx` 话题发起请求，通过订阅 `/slam/status` 话题接收响应。

**约定：**

- `/slam/status` 及所有 `/slam/xxx` 请求话题的消息类型均为 `std_msgs/String`，`data` 字段为 JSON 文本字符串。
- 接口编码有效范围为 `1801` ~ `1822`。
- 除状态查询类接口外，其余接口均为**异步**执行，SDK 节点内部在独立线程中处理，不会阻塞其它请求。
- 请求 JSON 需包含对应接口的必要字段（通用为 `request_id`，建图/定位类还需 `scene_id` 等）；JSON 非法时返回失败（`Failed to parse json request`），必要字段缺失时请求不会被处理。
- **请求与响应都会写入 `logs/slam_sdk.log`**，统一前缀为 `[API <task_id> <接口名>]`，便于现场复盘（见[日志与现场复核](#日志与现场复核)）。

### 通用响应格式

所有接口的响应均通过 `/slam/status` 话题返回，格式如下：

```json
{
    "request_id": "<任务ID，用于请求-响应对齐>",
    "scene_id": "<场景ID>",
    "task_id": "<接口编码，如 1801>",
    "success": 1,       // 0-已接收  1-成功  2-失败
    "message": "<描述信息>",
    "data": "<响应数据>"
}
```

| 字段 | 类型 | 说明 |
|------|------|------|
| `request_id` | string \| null | 回显请求中的 `request_id`；请求 JSON 解析失败时为 `null` |
| `scene_id` | string \| null | 建图类接口为请求中的 `scene_id`；导航类接口为当前导航场景ID（未初始化时为 `null`） |
| `task_id` | int | 接口编码 |
| `success` | int | `0` — 已接收（异步接口的中间态）；`1` — 成功；`2` — 失败 |
| `message` | string | 描述信息，失败时给出失败原因 |
| `data` | string \| null | 响应数据，见下方说明 |

> **`data` 字段格式：** 除个别接口外，`data` 为 `null` 或纯字符串。位姿/导航状态类接口的 `data` 是**被序列化后的 JSON 字符串**，需要二次解析（`json.loads`）后才能取到结构化数据。

| 接口 | `data` 内容 |
|------|-------------|
| 1805 / 1814 | JSON 字符串，解析后为对象（如 `{"pose_error": ..., "global_pose": {...}}`） |
| 1815 | 纯字符串状态值（如 `"NAV_READY"`） |
| 其余接口 | `null` |

> **注意：** `success = 0` 仅代表请求已被接收（异步任务已下发），最终结果会以 `success = 1`（成功）或 `success = 2`（失败）再次通过同一 `task_id` 返回，客户端需按 `request_id` 对齐并等待终态。

---

## 接口总览

| 编码 | 功能名称 | 类型 | Topic 请求 | 说明 |
|------|----------|------|------------|------|
| 1801 | 开始建图 | 建图 | `/slam/start_mapping` | 启动建图，需指定场景ID |
| 1802 | 结束建图 | 建图 | `/slam/stop_mapping` | 结束建图，保存地图文件 |
| 1822 | 强制终止建图 | 建图 | `/slam/terminate_mapping` | 强制终止建图，不保存地图 |
| 1804 | 初始化位姿 | 定位 | `/slam/init_pose` | 异步，支持自动/手动估计 |
| 1805 | 位姿状态获取 | 定位 | `/slam/get_global_pose` | 获取当前位姿及误差 |
| 1806 | 位姿导航 | 导航 | `/slam/nav_to_pose` | 异步，支持相对/绝对坐标 |
| 1807 | 暂停导航 | 导航 | `/slam/pause_nav` | 暂停当前导航 |
| 1808 | 恢复导航 | 导航 | `/slam/resume_nav` | 恢复已暂停导航 |
| 1809 | 取消导航 | 导航 | `/slam/cancel_nav` | 取消导航任务 |
| 1810 | 关闭 SLAM | 系统 | `/slam/shutdown` | 关闭雷达、定位、导航 |
| 1814 | 导航状态查询 | 导航 | `/slam/get_nav_state` | 查询导航进度及剩余距离 |
| 1815 | SLAM 状态查询 | 系统 | `/slam/get_state` | 查询 SLAM 运行状态 |
| 1816 | 里程计重启 | 定位 | `/slam/restart_odom` | 细粒度重启里程计并重新定位 |

> 里程计异常（心跳丢失 / 漂移）时 SDK 节点会**自动**执行 1816 的重启流程，无需外部触发，见[自动监测与保护机制](#自动监测与保护机制)。

---

## 位姿数据结构

多个接口中使用位姿数据，统一结构如下：

```json
{
    "use_euler": false,
    "position": {
        "x": 0.0,       // 单位：米
        "y": 0.0,
        "z": 0.0
    },
    "orientation": {    // 四元数（与 euler_angle 二选一）
        "q_x": 0.0,
        "q_y": 0.0,
        "q_z": 0.0,
        "q_w": 0.0
    },
    "euler_angle": {    // 欧拉角
        "r": 0.0,       // Roll  (弧度)
        "p": 0.0,       // Pitch (弧度)
        "y": 0.0        // Yaw   (弧度)
    }
}
```

> **注意：** `use_euler` 为 `true` 时读取 `euler_angle`，为 `false` 时读取 `orientation`（四元数）。

---

## 接口详情

### 1. 开始建图（1801）

**功能描述：** 调用此接口启动建图，开启里程计、建图等功能，建图过程持续到接收"结束建图"指令为止。

**请求** — `/slam/start_mapping`

```json
{
    "scene_id": "<场景ID，用户指定>",
    "slam_type": "indoor",
    "request_id": "<请求ID>"
}
```

| 参数 | 类型 | 说明 |
|------|------|------|
| `scene_id` | string | 场景ID，用户指定 |
| `slam_type` | string | 建图类型，如 `"indoor"`（保留字段，当前未使用） |
| `request_id` | string | 请求ID |

**响应** — `/slam/status`

```json
{
    "scene_id": "ground_floor",
    "request_id": "2312023102",
    "task_id": 1801,
    "success": 1,
    "message": "mapping started",
    "data": null
}
```

| 项 | 说明 |
|----|------|
| 前置状态 | 仅 `IDLE` 可用；其它状态返回失败：`Another task already running or in stopping` |
| 成功判据 | 启动后 `/cloud_registered` 话题须在 15 秒内出现；超时则清理进程、回到 `IDLE` 并返回失败 `mapping start failed` |
| 建图期间 | 状态为 `MAPPING_RUNNING`，此期间不接受定位/导航类请求 |

---

### 2. 结束建图（1802）

**功能描述：** 调用此接口结束建图，保存地图到指定路径（包含 `map.pcd`、`map.pgm`、`map.yaml`）。地图保存在 `data/maps/<scene_id>/` 目录，并同步更新 `data/maps/out` 软链接指向该目录。

**请求** — `/slam/stop_mapping`

```json
{
    "scene_id": "<场景ID>",
    "request_id": "<请求ID>"
}
```

| 参数 | 类型 | 说明 |
|------|------|------|
| `scene_id` | string | 场景ID |
| `request_id` | string | 请求ID |

**响应** — `/slam/status`

```json
{
    "scene_id": "<场景ID>",
    "request_id": "<请求ID>",
    "task_id": 1802,
    "success": 1,
    "message": "Map saved",
    "data": null
}
```

| 项 | 说明 |
|----|------|
| 前置状态 | 仅 `MAPPING_RUNNING` 可用；其它状态返回失败：`Mapping not running` |
| 成功判据 | 三个地图文件均成功拷贝且无异常；文件缺失或拷贝异常返回失败（`Missing map files` / `Copy map error`），此时仍会停止建图进程 |
| 结束后 | 状态回到 `IDLE`，建图进程全部退出 |

---

### 3. 强制终止建图（1822）

**功能描述：** 调用此接口强制终止建图进程，不保存地图文件。适用于建图异常或用户主动放弃当前建图任务。

**请求** — `/slam/terminate_mapping`

```json
{
    "request_id": "<请求ID>",
    "scene_id": "<场景ID>"
}
```

| 参数 | 类型 | 说明 |
|------|------|------|
| `request_id` | string | 请求ID |
| `scene_id` | string | 场景ID |

**响应** — `/slam/status`

```json
{
    "request_id": "<请求ID>",
    "scene_id": "<场景ID>",
    "task_id": 1822,
    "success": 1,
    "message": "Mapping terminate success",
    "data": null
}
```

| 项 | 说明 |
|----|------|
| 前置状态 | 仅 `MAPPING_RUNNING` 可用；其它状态返回失败：`Mapping not running` |
| 与 1802 区别 | 不执行保存地图服务、不拷贝文件，直接杀进程；结束后状态回到 `IDLE` |

---

### 4. 初始化位姿（1804，异步）

**功能描述：** 调用此接口启动定位与导航，开启里程计、定位等功能，过程持续到接收"结束 SLAM"为止。

**请求** — `/slam/init_pose`

```json
{
    "request_id": "000001",
    "auto_init_pose": true,
    "manual_pose": {
        "use_euler": false,
        "position": { "x": 0.0, "y": 0.0, "z": 0.0 },
        "orientation": { "q_x": 0.0, "q_y": 0.0, "q_z": 0.0, "q_w": 0.0 },
        "euler_angle": { "r": 0.0, "p": 0.0, "y": 0.0 }
    },
    "scene_id": "test"
}
```

| 参数 | 类型 | 说明 |
|------|------|------|
| `request_id` | string | 请求ID |
| `auto_init_pose` | bool | 是否自动估计位姿；`true` 时忽略 `manual_pose` |
| `manual_pose` | object | 手动估计位姿（结构见[位姿数据结构](#位姿数据结构)）；省略的字段按 `x/y/z=0`、`q_w=1` 处理 |
| `scene_id` | string | 地图编号，须为已存在的场景（含 `map.pcd`/`map.pgm`/`map.yaml`） |

> **内部行为：** `auto_init_pose=true` 时调用定位节点的 `/global_initialpose_estimation`（BBS 全局搜索）；`false` 时向 `/initialpose` 发布给定初始位姿。首次启动还会拉起雷达、BBS、DLL、Nav2 与 pose_verifier；后续在 `NAV_READY`/`NAV_PAUSED` 下重复调用只更新位姿，不重启导航栈。

**响应** — `/slam/status`

首次启动（未拉起导航栈）时，先返回中间态，再返回最终结果：

```json
{
    "request_id": "000001",
    "scene_id": "test",
    "task_id": 1804,
    "success": 0,
    "message": "Success to send init request",
    "data": null
}
```

```json
{
    "request_id": "000001",
    "scene_id": "test",
    "task_id": 1804,
    "success": 1,
    "message": "Ready for navigation",
    "data": null
}
```

| 项 | 说明 |
|----|------|
| 前置状态 | 仅 `IDLE`、`NAV_STARTING`、`NAV_READY`、`NAV_PAUSED` 可用（`NAV_RUNNING` 下不支持修改位姿），否则返回 `Failed to execute request in current robot state: <state>` |
| 场景约束 | `scene_id` 首次绑定后，导航关停前不可更改；不一致返回 `Request mismatch with current scene id: ...` |
| 重复调用 | 导航栈已启动时，仅更新位姿，直接返回最终结果（`Success to setup initpose automatically` / `Success to setup initpose manually`），不再返回中间态 |
| 成功判据 | `nav_wait_time`（默认 90 秒）内导航就绪：定位误差收敛到 `pose_err_thr`（默认 0.20 米）以内且位姿评分通过；否则返回 1804 `success=2` 并关停导航栈，失败原因分三种：`Timeout to start navigation, please restart`（定位误差未收敛）、`Low pose score, please relocalize`（位姿评分不达标）、`Pose score service unavailable, please restart`（评分服务不可用） |
| 其它失败原因 | 手动位姿四元数非法（`Invalid quaternion: ...`）、场景不存在或地图文件缺失（`Scene id <id> not exist or missing files: ...`）、导航残留进程未清理干净（`Leftover nav processes detected, ...`）、里程计自愈期间导航已被关停（`Navigation shutdown during odom restart`）、启动过程被 1810 或新 1804 取代（`Nav was aborted before start, ...`）等 |
| 成功后 | 状态为 `NAV_READY`，可下发 1806 导航目标 |

---

### 5. 机器人位姿状态获取（1805）

**功能描述：** 返回机器人当前的全局位姿及估计误差。

**请求** — `/slam/get_global_pose`

```json
{
    "request_id": ""
}
```

| 参数 | 类型 | 说明 |
|------|------|------|
| `request_id` | string | 请求ID |

**响应** — `/slam/status`

```json
{
    "request_id": "",
    "scene_id": "test",
    "task_id": 1805,
    "success": 1,
    "message": "Success to get global pose",
    "data": "{\"pose_error\": 0.5, \"global_pose\": {\"position\": {...}, \"orientation\": {...}, \"euler_angle\": {...}}}"
}
```

`data` 为 JSON 字符串，解析后结构如下：

```json
{
    "pose_error": 0.5,
    "global_pose": {
        "position": { "x": 0.0, "y": 0.0, "z": 0.0 },
        "orientation": { "q_x": 0.0, "q_y": 0.0, "q_z": 0.0, "q_w": 0.0 },
        "euler_angle": { "r": 0.0, "p": 0.0, "y": 0.0 }
    }
}
```

| 参数 | 类型 | 说明 |
|------|------|------|
| `pose_error` | float | 位姿估计误差（米）；未开始定位时为 `1000` |
| `global_pose` | object | 全局位姿（结构见[位姿数据结构](#位姿数据结构)） |

| 项 | 说明 |
|----|------|
| 前置状态 | 仅 `NAV_READY`、`NAV_RUNNING`、`NAV_PAUSED`、`NAV_STOPPING` 可用，否则返回 `Navigation invalid, no global pose prepared to response` |

> `global_pose` 取自最近一次 `map -> base_link` TF；未开始定位时 `pose_error` 为 `1000`。

---

### 6. 位姿导航（1806，异步）

**功能描述：** 调用此接口开始导航，支持相对坐标系（ego）和绝对坐标系（map）。

**请求** — `/slam/nav_to_pose`

```json
{
    "request_id": "000001",
    "relative": false,
    "target_pose": {
        "use_euler": false,
        "position": { "x": 0.0, "y": 0.0, "z": 0.0 },
        "orientation": { "q_x": 0.0, "q_y": 0.0, "q_z": 0.0, "q_w": 0.0 },
        "euler_angle": { "r": 0.0, "p": 0.0, "y": 0.0 }
    }
}
```

| 参数 | 类型 | 说明 |
|------|------|------|
| `request_id` | string | 请求ID，用于请求-响应对齐；暂停-恢复时保持一致 |
| `relative` | bool | `false` — 绝对坐标系（map）；`true` — 相对坐标系（ego，以收到请求时的 `map -> base_link` TF 为基准转换） |
| `target_pose` | object | 目标位姿（结构见[位姿数据结构](#位姿数据结构)）；省略的字段按 `x/y/z=0`、`q_w=1` 处理 |

> **请求被接受后：** SDK 先发布零速度指令，清理全局/局部代价地图，再向 Nav2 下发目标。

**响应** — `/slam/status`

请求被接收时先返回中间态：

```json
{
    "request_id": "000001",
    "scene_id": "test",
    "task_id": 1806,
    "success": 0,
    "message": "Success to send nav request",
    "data": null
}
```

导航结束时以同一 `request_id` 返回最终结果：

```json
{
    "request_id": "000001",
    "scene_id": "test",
    "task_id": 1806,
    "success": 1,
    "message": "Success to reach the goal",
    "data": null
}
```

| 项 | 说明 |
|----|------|
| 前置状态 | 仅 `NAV_READY`（新目标）或 `NAV_PAUSED`（恢复原目标）；`NAV_RUNNING` 下需先取消（1809）才能下发新目标 |
| 误差约束 | `NAV_READY` 下若定位误差超过 `nav_reject_err_thr`（参数文件默认 0.4 米；代码内建缺省 0.3 米）则拒绝：`Localization error too high (...), please re-localize` |
| 状态相关拒绝 | `Navigation not started, invalid request`（`IDLE`）、`Navigation is starting, please wait for a moment`（`NAV_STARTING`）、`Navigation running, please cancel current goal before new request`（`NAV_RUNNING`）、`Navigation is pausing, please wait`（`NAV_PAUSING`）、`Navigation paused, please resume before request`（`NAV_PAUSED` 且 `request_id` 不是被暂停的原导航）、`Navigation not ready, invalid request`（其它状态） |
| 目标失败原因 | `Goal was rejected`、`Goal was cancelled`、`Failed to reach the goal`（中止/未到达）、`Failed to connect nav server`、`Failed to lookup relative transform`（相对导航时 TF 查询失败）、`Failed to figure out result` |
| 自动超时 | 单次导航超过 `max_nav_time`（默认 240 秒）由 SDK 节点自动取消，返回 `Goal was cancelled` |
| 暂停后恢复 | 只有 `NAV_PAUSED` 下用原 `request_id`（即 1808 的内部恢复）才接受；外部直接发新目标会被拒绝 |

---

### 7. 暂停导航（1807）

**功能描述：** 调用此接口暂停当前导航任务。内部通过取消当前导航目标实现，暂停成功后目标被保留，可通过 1808 恢复。

**请求** — `/slam/pause_nav`

```json
{
    "request_id": ""
}
```

| 参数 | 类型 | 说明 |
|------|------|------|
| `request_id` | string | 请求ID；暂停结果回报使用该请求ID |

**响应** — `/slam/status`

```json
{
    "request_id": "",
    "scene_id": "test",
    "task_id": 1807,
    "success": 1,
    "message": "Navigation paused",
    "data": null
}
```

| 项 | 说明 |
|----|------|
| 前置状态 | 仅 `NAV_RUNNING` 可用，否则返回 `Failed to pause because navigation not running` |
| 成功后 | 状态为 `NAV_PAUSED`，可通过 1808 恢复；暂停引发的目标取消不会再回报 1806 失败 |
| 失败原因 | `Failed to pause navigation`（取消失败）、`No goal to cancel` |
| 边界情况 | 暂停时目标恰好已结束，按暂停成功处理，消息为 `Navigation paused (goal already finished)` |

---

### 8. 恢复导航（1808）

**功能描述：** 调用此接口恢复已暂停的导航任务。内部以原导航目标（absolute 坐标）重新下发。

**请求** — `/slam/resume_nav`

```json
{
    "request_id": ""
}
```

| 参数 | 类型 | 说明 |
|------|------|------|
| `request_id` | string | 请求ID（仅中间态使用，最终结果沿用原导航的 `request_id`） |

**响应** — `/slam/status`

```json
{
    "request_id": "",
    "scene_id": "test",
    "task_id": 1808,
    "success": 0,
    "message": "Resume accepted",
    "data": null
}
```

```json
{
    "request_id": "<原导航的 request_id>",
    "scene_id": "test",
    "task_id": 1808,
    "success": 1,
    "message": "Resume success",
    "data": null
}
```

| 项 | 说明 |
|----|------|
| 前置状态 | 仅 `NAV_PAUSED` 且存在已保存目标时可用，否则返回 `Goal not paused, failed to resume` / `Goal not received, failed to resume` |
| request_id 约定 | 中间态 `success=0` 使用本次恢复请求的 `request_id`；最终结果使用**原导航目标**的 `request_id`（恢复以原目标重发，可通过 1806 的最终结果一并判断） |
| 失败原因 | `Resume failed: goal rejected`（目标被拒绝）、`Navigation paused, please resume before request`（暂停态下发新目标） |

---

### 9. 取消导航（1809）

**功能描述：** 调用此接口取消当前导航任务。

**请求** — `/slam/cancel_nav`

```json
{
    "request_id": ""
}
```

| 参数 | 类型 | 说明 |
|------|------|------|
| `request_id` | string | 请求ID |

**响应** — `/slam/status`

```json
{
    "request_id": "",
    "scene_id": "test",
    "task_id": 1809,
    "success": 1,
    "message": "Goal cancelled",
    "data": null
}
```

| 项 | 说明 |
|----|------|
| 前置状态 | `NAV_RUNNING` 或 `NAV_PAUSING`；暂停态（`NAV_PAUSED`）下取消会返回 `Cancel succeeded while paused`，并额外回报一条 1806 失败（`Navigation cancelled while paused`） |
| 成功后 | 状态回到 `NAV_READY` |
| 失败原因 | `Navigation not running`、`No goal to cancel`、`Failed to cancel goal` |

---

### 10. 关闭 SLAM（1810）

**功能描述：** 调用此接口关闭雷达驱动、定位、里程计和导航功能。

**请求** — `/slam/shutdown`

```json
{
    "request_id": ""
}
```

| 参数 | 类型 | 说明 |
|------|------|------|
| `request_id` | string | 请求ID |

**响应** — `/slam/status`

```json
{
    "request_id": "",
    "scene_id": "test",
    "task_id": 1810,
    "success": 1,
    "message": "Navigation stopped",
    "data": null
}
```

| 项 | 说明 |
|----|------|
| 前置状态 | 仅 `NAV_STARTING`、`NAV_READY`、`NAV_RUNNING`、`NAV_PAUSING`、`NAV_PAUSED` 可用，否则返回 `Navigation not ready` |
| 成功后 | 状态回到 `IDLE`，`scene_id` 清空，可重新执行 1801/1804 |
| 失败情况 | 关停后仍检测到残留导航进程时返回失败：`Navigation stopped but leftover processes exist` |

---

### 11. 导航状态查询（1814）

**功能描述：** 查询当前导航任务的实时状态，包括目标位姿、已导航时间及剩余距离。

**请求** — `/slam/get_nav_state`

```json
{
    "request_id": "000001"
}
```

| 参数 | 类型 | 说明 |
|------|------|------|
| `request_id` | string | 请求ID |

**响应** — `/slam/status`

```json
{
    "request_id": "000001",
    "scene_id": "test",
    "task_id": 1814,
    "success": 1,
    "message": "Success to get nav state",
    "data": "{\"goal_pose\": {...}, \"navigation_time\": 11.4, \"distance_remaining\": 5.1}"
}
```

`data` 为 JSON 字符串，解析后结构如下：

```json
{
    "goal_pose": {
        "position": { "x": 0.0, "y": 0.0, "z": 0.0 },
        "orientation": { "q_x": 0.0, "q_y": 0.0, "q_z": 0.0, "q_w": 0.0 },
        "euler_angle": { "r": 0.0, "p": 0.0, "y": 0.0 }
    },
    "navigation_time": 11.4,
    "distance_remaining": 5.1
}
```

| 参数 | 类型 | 说明 |
|------|------|------|
| `goal_pose` | object | 导航目标位姿（绝对坐标，单位为米/弧度） |
| `navigation_time` | float | 已导航时间（秒） |
| `distance_remaining` | float | 剩余距离（米） |

| 项 | 说明 |
|----|------|
| 前置状态 | 仅 `NAV_RUNNING` 可用，否则返回 `Robot not in navigation state` |
| 其它失败原因 | `Never received a goal pose`（未收到目标）、`Empty navigation feedback, navigation finished or not started`（反馈为空，导航已结束或未开始） |

---

### 12. SLAM 状态查询（1815）

**功能描述：** 查询 SLAM 系统的当前运行状态。

**请求** — `/slam/get_state`

```json
{
    "request_id": ""
}
```

| 参数 | 类型 | 说明 |
|------|------|------|
| `request_id` | string | 请求ID |

**响应** — `/slam/status`

```json
{
    "request_id": "",
    "scene_id": "test",
    "task_id": 1815,
    "success": 1,
    "message": "Update state",
    "data": "IDLE"
}
```

| 项 | 说明 |
|----|------|
| 前置状态 | 无限制，任意状态可用 |
| `scene_id` | 建图类状态（`MAPPING_*`）返回建图场景ID，其余返回当前导航场景ID（未初始化时为 `null`） |

**`data` 状态枚举：**

> 状态之间的迁移关系、各状态下可用的接口见 [SDK 状态机](./SDK状态机.md)。

| 值 | 说明 |
|------|------|
| `IDLE` | 空闲（未建图、未定位） |
| `MAPPING_STARTING` | 正在启动建图 |
| `MAPPING_RUNNING` | 正在建图 |
| `MAPPING_STOPPING` | 正在结束建图（保存地图） |
| `MAPPING_TERMINATING` | 正在强制终止建图 |
| `NAV_STARTING` | 正在启动定位与导航 |
| `NAV_READY` | 导航就绪（已定位，未下发目标） |
| `NAV_RUNNING` | 正在导航 |
| `NAV_PAUSING` | 正在暂停导航（取消请求已发出，等待结果） |
| `NAV_PAUSED` | 导航已暂停 |
| `NAV_STOPPING` | 正在结束导航 |

> 里程计重启期间，节点内部会置位 `freezing`（不会作为状态值返回），此时仅放行 1804（重定位）与 1810（关停），其它指令一律拒绝，详见 [1816](#13-里程计重启1816)。

---

### 13. 里程计重启（1816）

**功能描述：** 细粒度重启里程计（仅重启里程计，不重启定位、导航与雷达），成功后自动重新定位并在必要时恢复导航。用于里程计异常（心跳丢失、漂移）后的恢复。

**请求** — `/slam/restart_odom`

```json
{
    "request_id": "<请求ID>",
    "scene_id": "<场景ID>",
    "relocalize": true
}
```

| 参数 | 类型 | 说明 |
|------|------|------|
| `request_id` | string | 请求ID，用于请求-响应对齐 |
| `scene_id` | string | 场景ID（当前实现未校验，可省略） |
| `relocalize` | bool | 是否执行（限定范围）全局重定位，缺省 `true`。见下表“relocalize 语义” |

**relocalize 语义**（仅影响是否调用 BBS 全局重定位，两种取值都会按重启前位姿重新锚定）

| 取值 | 适用场景 | 行为 |
|------|----------|------|
| `true`（默认） | SDK 自动监测触发（里程计漂移/心跳丢失）；外部调用方不确定机器人是否移动 | 先按重启前位姿 P 重新锚定 `map->camera_init`，再以 P 为中心做一次限定范围全局重定位（BBS），用当前点云实测纠正重启期间因惯性/消息延迟产生的位移/旋转；成功后播报“里程计恢复” |
| `false` | 调用方确信机器人未移动（如导览“同场景换图”，每次任务结束后调用） | 仍按 P 重新锚定 `map->camera_init`（里程计重启后 `camera_init` 回到新原点，不重新锚定会使机器人跳到旧里程计起点），但跳过全局重定位搜索，直接沿用上次定位；属调用方主动发起的例行重启，成功时**不播报**（避免用户未感知异常却听到恢复播报） |

> 里程计进程重启后 `camera_init` 原点变为机器人当前位置，因此无论 `relocalize` 为何值，都必须发布一次 `/initialpose`（request_id = `odom_restart_relocalize`）重新锚定；`relocalize` 只决定是否再做一次全局搜索。

**响应** — `/slam/status`

```json
{
    "request_id": "<请求ID>",
    "scene_id": "test",
    "task_id": 1816,
    "success": 1,
    "message": "Odom restarted",
    "data": null
}
```

| 项 | 说明 |
|----|------|
| 前置状态 | 仅定位/导航相关状态（`NAV_STARTING`、`NAV_READY`、`NAV_RUNNING`、`NAV_PAUSING`、`NAV_PAUSED`）可用，否则返回 `Odom restart ignored: abnormal state` |
| 重入保护 | 已在重启中重复请求返回 `Odom restart ignored: already restarting`；外部显式触发不受自动重启冷却时间限制 |
| 执行流程 | ① 导航中（`NAV_RUNNING`）先暂停当前导航；② 重启里程计进程并等待其恢复发布（脚本内部失败自动重试 `odom_restart_max_fail` 次，默认 3 次）；③ 复用 1804 手动重定位（以重启前位姿 P 为初始位姿）；④ `relocalize=true` 时以 P 为中心限定平面搜索范围（`odom_restart_search_range`，默认 3 m，0=关闭）并将 BBS 线程数限制为 `odom_restart_threads`（默认 1，0=不修改）后调用一次全局初始化（`/global_initialpose_estimation`），再恢复原范围与线程数；`relocalize=false` 时跳过本步；⑤ 位姿评分校验通过后：导航中则自动恢复原目标，否则状态回到 `NAV_READY` |
| 重启期间 | 节点进入 `freezing` 状态，除 1804（重定位）与 1810（关停）外的所有指令均被拒绝：`Odom recovering, only relocalize/shutdown allowed` |
| 失败原因 | `Odom restart ignored: ...`（状态不满足/重入/冷却）、`Odom restart aborted: navigation shutdown`（流程中被 1810 关停）、`Low pose score after odom restart: ...`（重定位后评分不达标，同时若导航中原目标也会回报 1806 失败）、`Pose score service call failed after odom restart...`（评分服务不可用）、`Odom restart failed, full shutdown`（脚本重试仍失败，转而执行完整关停） |
| 自动恢复 | `relocalize=true` 时重启成功会播放"里程计恢复"语音，`false` 时静默；失败会播放"导航异常 请接管"语音并放弃细粒度恢复 |
| 与 1804 的关系 | 重启后的重新定位内部复用 1804 流程，会额外产生一条 `1804` 状态（`request_id = odom_restart_relocalize`），客户端可忽略 |

---

## 自动监测与保护机制

以下行为由 SDK 节点自动触发，无需外部请求。自动触发时使用固定哨兵 `request_id`（心跳丢失为 `odom_lost_caused_restart`，漂移为 `odom_drift_caused_restart`，导航超时为 `timeout_caused_cancel`，建图漂移为 `drift_caused_terminate`），这些结果也会通过 `/slam/status` 回报；客户端可按这些固定值识别并忽略/记录，不应把它们当作自己发起的请求。

| 机制 | 触发条件 | 处理动作 |
|------|----------|----------|
| 里程计心跳监测 | 导航就绪/运行后超过 `odom_run_wait_time`（默认 1 秒）未收到里程计数据（启动阶段宽限 `odom_wait_time`，默认 30 秒） | 播报"里程计丢失"，自动执行 1816 重启流程 |
| 里程计漂移检测 | 相邻两帧里程计位置推算速度超过 `odom_drift_thr`（默认 5.0 m/s），且连续 `odom_drift_confirm_n`（默认 5）帧异常 | 播报"里程计漂移"，自动执行 1816 重启流程 |
| 自动重启冷却 | 非外部触发的重启间隔小于 `restart_cooldown`（默认 10 秒） | 忽略本次重启，避免重启风暴 |
| 重启连续失败 | 脚本内部重试 `odom_restart_max_fail`（默认 3 次）仍失败 | 放弃细粒度重启，执行完整关停（1801/1804 需重新发起） |
| 重启后限定范围重定位 | 里程计重启成功（脚本重试后恢复发布） | 以重启前位姿 P 为中心设置 BBS 平面搜索范围（`odom_restart_search_range`，默认 3 m，0=关闭）与线程数（`odom_restart_threads`，默认 1，0=不修改）并调用一次全局初始化，随后恢复原搜索范围与线程数 |
| 导航超时保护 | 单次导航时间超过 `max_nav_time`（默认 240 秒） | 播报"导航超时"，自动取消当前导航（1806 返回失败） |
| 建图漂移保护 | 建图中连续 `map_drift_confirm_n`（默认 3）帧里程计速度超过 `map_drift_speed_thr`（默认 5 m/s） | 播报"里程计漂移"，自动执行 1822 终止建图（不保存地图） |
| recovery 语音提醒 | 导航过程中收到新的 recovery 事件，且距上次播报超过 `recovery_voice_cooldown`（默认 15 秒） | 播报"你好 请让一让" |

> 参数文件：`config/params/slam_sdk.yaml`，修改后需重启 SDK 服务生效。

### 碰撞风险话题

| 话题 | 类型 | 说明 |
|------|------|------|
| `/slam/collision_risk` | `std_msgs/Float32` | 碰撞风险值，范围 0~1（越大越危险），1 Hz |

计算方式：在局部代价地图（`risk_costmap_topic`，默认 `/local_costmap/costmap`，global_frame = map）上，
以机器人位姿为基准，采样其**前方**长 `risk_range`（默认 1.5 m）、左右各 `risk_half_width`（默认 0.4 m）的矩形区域，
跳过未知栅格后取平均代价（0~100），风险 = 平均代价 / `risk_cost_thr`（默认 50）并截断到 1。
代价地图或位姿未就绪、以及前方无有效栅格时不发布该话题（仅打日志），下游应容忍消息缺失。

### 内部话题（供调试，不建议外部依赖）

| 话题 | 类型 | 说明 |
|------|------|------|
| `/slam/status` | `std_msgs/String` | 所有接口的统一响应 |
| `/slam/_state` | `std_msgs/String` | 当前状态机状态，10 Hz 广播 |
| `/slam/_nav_error` | `std_msgs/Float32` | 导航内部误差回传（来自 DLL 的 `~/fitness_score`） |
| `/slam/collision_risk` | `std_msgs/Float32` | 碰撞风险（见上节） |
| `/cmd_vel` | `geometry_msgs/Twist` | SDK 的速度输出；`launch_sdk.sh` 会 remap 到 `/cmd_vel_nav`，经 `velocity_smoother` 下发 |

> **TF 隔离：** 整条 SLAM 栈把 `/tf`、`/tf_static` remap 到 `/slam/tf`、`/slam/tf_static`。外部节点若要读取本栈 TF，必须做同样的 remap（或直接订阅 `/slam/tf`）。

---

## 日志与现场复核

SDK 节点会把**每条请求和每条响应**写入自身日志（`scripts/launch_sdk.sh` 重定向到 `logs/slam_sdk.log`）：

```bash
tail -f logs/slam_sdk.log | grep --line-buffered '\[API'
grep '\[API' logs/slam_sdk.log | tail -50
```

格式：

```
[API 1806 nav_to_pose] request_id=test_nav_to_pose scene_id=arena data={...}
[API 1806 nav_to_pose] response: {"request_id":"test_nav_to_pose","scene_id":"arena","task_id":1806,"success":0,"message":"Success to send nav request","data":null}
```

现场常用命令：

```bash
slam_status                                  # 持续看 /slam/status
ros2 topic echo /slam/status --full-length   # 长 data 不截断
slam_state                                   # 当前状态
ls -lt logs/nav_fail | head                  # 导航失败现场（annotated.png + meta.json）
```

> 完整日志清单、CPU 监控和 Foxglove 调试见 [SDK 全流程操作手册](./SDK全流程操作手册.md) 第 7、8 节。

---

## 导航流程

> 完整状态机（11 个状态、全部状态迁移、自动保护机制与 1816 子流程）见 [SDK 状态机](./SDK状态机.md)。下面是主要调用顺序：

```
开始建图 (1801)  ──→  结束建图 (1802)
    │                      │
    └── 强制终止 (1822)    │
                           ↓
                    初始化位姿 (1804)
                           ↓
            位姿导航 (1806)  ←  位姿状态获取 (1805)
                    ↓
            ┌── 暂停 (1807) → 恢复 (1808)
            ├── 取消 (1809)
            ├── 导航状态查询 (1814)
            └── 里程计重启 (1816) ──→ 重定位 (1804) ──→ 恢复原目标 (1808)
                    ↓
                关闭 SLAM (1810)
```
