# Navigation SDK Interface Documentation (Topic-based)

> **Note:** This is the English version of [SDK接口文档.md](../SDK接口文档.md).
> **Last updated:** 2026-11-16 (interface codes 1801~1822)
> **Service note:** This document describes the SDK service deployed under `/opt/app/slam`; the interfaces, state machine, and error messages are governed by the version actually running.
> **Companion documents:** [SDK State Machine](./SDK_STATE_MACHINE.md) · [SDK Usage Examples](./SDK_EXAMPLES.md) · [SDK Full Workflow Guide](./SDK_FULL_WORKFLOW_GUIDE.md)

## Overview

This document describes the topic interfaces of the navigation-related SDK, covering mapping, localization, navigation and state query modules. Every interface is invoked by publishing to a `/slam/xxx` topic and receives its response by subscribing to the `/slam/status` topic.

**Conventions:**

- `/slam/status` and all `/slam/xxx` request topics use the message type `std_msgs/String`, where the `data` field is a JSON text string.
- Valid interface codes range from `1801` to `1822`.
- Except for the state query interfaces, all interfaces execute **asynchronously**; the SDK node handles them in a dedicated thread and does not block other requests.
- A request JSON must contain the fields required by the corresponding interface (generally `request_id`; mapping/localization interfaces additionally require `scene_id`, etc.). Invalid JSON returns a failure (`Failed to parse json request`); if a required field is missing the request is not processed.
- **Both requests and responses are written to `logs/slam_sdk.log`**, with the uniform prefix `[API <task_id> <interface name>]`, to make on-site review easier (see [Logging and On-Site Review](#logging-and-on-site-review)).

### Common Response Format

Responses for all interfaces are returned on the `/slam/status` topic, in the following format:

```json
{
    "request_id": "<task ID, used to align request and response>",
    "scene_id": "<scene ID>",
    "task_id": "<interface code, such as 1801>",
    "success": 1,       // 0 - accepted  1 - success  2 - failure
    "message": "<description>",
    "data": "<response data>"
}
```

| Field | Type | Description |
|------|------|------|
| `request_id` | string \| null | Echoes the `request_id` from the request; `null` when the request JSON fails to parse |
| `scene_id` | string \| null | For mapping interfaces, the `scene_id` from the request; for navigation interfaces, the current navigation scene ID (`null` when not initialized) |
| `task_id` | int | Interface code |
| `success` | int | `0` — accepted (intermediate state of asynchronous interfaces); `1` — success; `2` — failure |
| `message` | string | Description; carries the failure reason when the request fails |
| `data` | string \| null | Response data, see the notes below |

> **`data` field format:** Except for a few interfaces, `data` is `null` or a plain string. For pose/navigation state interfaces, `data` is a **serialized JSON string** that must be parsed a second time (`json.loads`) to obtain the structured data.

| Interface | `data` content |
|------|-------------|
| 1805 / 1814 | JSON string; once parsed it is an object (such as `{"pose_error": ..., "global_pose": {...}}`) |
| 1815 | Plain state string (such as `"NAV_READY"`) |
| All other interfaces | `null` |

> **Note:** `success = 0` only means the request has been accepted (the asynchronous task has been dispatched). The final result is returned again through the same `task_id` with `success = 1` (success) or `success = 2` (failure); clients must align responses by `request_id` and wait for a terminal state.

---

## Interface Overview

| Code | Function | Type | Request Topic | Description |
|------|----------|------|------------|------|
| 1801 | Start mapping | Mapping | `/slam/start_mapping` | Start mapping; a scene ID must be specified |
| 1802 | Stop mapping | Mapping | `/slam/stop_mapping` | Stop mapping and save the map files |
| 1822 | Force-terminate mapping | Mapping | `/slam/terminate_mapping` | Force-terminate mapping without saving the map |
| 1804 | Initialize pose | Localization | `/slam/init_pose` | Asynchronous; supports automatic/manual estimation |
| 1805 | Get pose state | Localization | `/slam/get_global_pose` | Get the current pose and its error |
| 1806 | Nav to pose | Navigation | `/slam/nav_to_pose` | Asynchronous; supports relative/absolute coordinates |
| 1807 | Pause navigation | Navigation | `/slam/pause_nav` | Pause the current navigation |
| 1808 | Resume navigation | Navigation | `/slam/resume_nav` | Resume a paused navigation |
| 1809 | Cancel navigation | Navigation | `/slam/cancel_nav` | Cancel the navigation task |
| 1810 | Shutdown SLAM | System | `/slam/shutdown` | Shut down lidar, localization and navigation |
| 1814 | Get navigation state | Navigation | `/slam/get_nav_state` | Query navigation progress and remaining distance |
| 1815 | Get SLAM state | System | `/slam/get_state` | Query the SLAM running state |
| 1816 | Restart odometry | Localization | `/slam/restart_odom` | Finely grained odometry restart followed by relocalization |

> When the odometry is abnormal (heartbeat lost / drift), the SDK node performs the 1816 restart flow **automatically**, without any external trigger; see [Automatic Monitoring and Protection](#automatic-monitoring-and-protection).

---

## Pose Data Structure

Several interfaces use pose data, always in the following structure:

```json
{
    "use_euler": false,
    "position": {
        "x": 0.0,       // unit: meters
        "y": 0.0,
        "z": 0.0
    },
    "orientation": {    // quaternion (either this or euler_angle)
        "q_x": 0.0,
        "q_y": 0.0,
        "q_z": 0.0,
        "q_w": 0.0
    },
    "euler_angle": {    // Euler angles
        "r": 0.0,       // Roll  (radians)
        "p": 0.0,       // Pitch (radians)
        "y": 0.0        // Yaw   (radians)
    }
}
```

> **Note:** When `use_euler` is `true` the `euler_angle` field is read; when it is `false` the `orientation` field (quaternion) is read.

---

## Interface Details

### 1. Start Mapping (1801)

**Description:** Call this interface to start mapping. It brings up odometry, mapping and related functions, and mapping continues until a "stop mapping" command is received.

**Request** — `/slam/start_mapping`

```json
{
    "scene_id": "<scene ID, specified by the user>",
    "slam_type": "indoor",
    "request_id": "<request ID>"
}
```

| Parameter | Type | Description |
|------|------|------|
| `scene_id` | string | Scene ID, specified by the user |
| `slam_type` | string | Mapping type, such as `"indoor"` (reserved field, currently unused) |
| `request_id` | string | Request ID |

**Response** — `/slam/status`

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

| Item | Description |
|----|------|
| Precondition | Only available in `IDLE`; any other state returns a failure: `Another task already running or in stopping` |
| Success criterion | After start-up the `/cloud_registered` topic must appear within 15 seconds; on timeout the processes are cleaned up, the state returns to `IDLE`, and the failure `mapping start failed` is returned |
| During mapping | The state is `MAPPING_RUNNING`; localization/navigation requests are not accepted while it lasts |

---

### 2. Stop Mapping (1802)

**Description:** Call this interface to stop mapping and save the map to the designated path (including `map.pcd`, `map.pgm`, `map.yaml`). The map is saved in the `data/maps/<scene_id>/` directory, and the `data/maps/out` symlink is updated at the same time to point to that directory.

**Request** — `/slam/stop_mapping`

```json
{
    "scene_id": "<scene ID>",
    "request_id": "<request ID>"
}
```

| Parameter | Type | Description |
|------|------|------|
| `scene_id` | string | Scene ID |
| `request_id` | string | Request ID |

**Response** — `/slam/status`

```json
{
    "scene_id": "<scene ID>",
    "request_id": "<request ID>",
    "task_id": 1802,
    "success": 1,
    "message": "Map saved",
    "data": null
}
```

| Item | Description |
|----|------|
| Precondition | Only available in `MAPPING_RUNNING`; any other state returns a failure: `Mapping not running` |
| Success criterion | All three map files are copied successfully with no exception; missing files or a copy exception returns a failure (`Missing map files` / `Copy map error`), and even then the mapping processes are still stopped |
| After completion | The state returns to `IDLE` and all mapping processes have exited |

---

### 3. Force-Terminate Mapping (1822)

**Description:** Call this interface to force-terminate the mapping processes without saving any map file. It is intended for mapping failures or when the user voluntarily abandons the current mapping task.

**Request** — `/slam/terminate_mapping`

```json
{
    "request_id": "<request ID>",
    "scene_id": "<scene ID>"
}
```

| Parameter | Type | Description |
|------|------|------|
| `request_id` | string | Request ID |
| `scene_id` | string | Scene ID |

**Response** — `/slam/status`

```json
{
    "request_id": "<request ID>",
    "scene_id": "<scene ID>",
    "task_id": 1822,
    "success": 1,
    "message": "Mapping terminate success",
    "data": null
}
```

| Item | Description |
|----|------|
| Precondition | Only available in `MAPPING_RUNNING`; any other state returns a failure: `Mapping not running` |
| Difference from 1802 | It does not call the map-saving service and does not copy files; it kills the processes directly. When it finishes the state returns to `IDLE` |

---

### 4. Initialize Pose (1804, asynchronous)

**Description:** Call this interface to start localization and navigation. It brings up odometry, localization and related functions, and the process continues until a "shutdown SLAM" command is received.

**Request** — `/slam/init_pose`

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

| Parameter | Type | Description |
|------|------|------|
| `request_id` | string | Request ID |
| `auto_init_pose` | bool | Whether to estimate the pose automatically; when `true`, `manual_pose` is ignored |
| `manual_pose` | object | Manually estimated pose (structure in [Pose Data Structure](#pose-data-structure)); omitted fields are treated as `x/y/z=0`, `q_w=1` |
| `scene_id` | string | Map identifier; it must be an existing scene (containing `map.pcd`/`map.pgm`/`map.yaml`) |

> **Internal behavior:** When `auto_init_pose=true`, the localization node's `/global_initialpose_estimation` (BBS global search) is called; when `false`, the given initial pose is published to `/initialpose`. The first start-up also brings up the lidar, BBS, DLL, Nav2 and pose_verifier; subsequent calls in `NAV_READY`/`NAV_PAUSED` only update the pose and do not restart the navigation stack.

**Response** — `/slam/status`

On the first start-up (when the navigation stack is not yet up), an intermediate state is returned first and the final result afterwards:

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

| Item | Description |
|----|------|
| Precondition | Only `IDLE`, `NAV_STARTING`, `NAV_READY` and `NAV_PAUSED` are available (changing the pose is not supported in `NAV_RUNNING`); otherwise `Failed to execute request in current robot state: <state>` is returned |
| Scene constraint | Once `scene_id` is bound for the first time it cannot be changed until navigation is shut down; a mismatch returns `Request mismatch with current scene id: ...` |
| Repeated calls | When the navigation stack is already running, only the pose is updated and the final result is returned directly (`Success to setup initpose automatically` / `Success to setup initpose manually`); no intermediate state is returned |
| Success criterion | Navigation becomes ready within `nav_wait_time` (default 90 seconds): the localization error converges within `pose_err_thr` (default 0.20 meters) and the pose score passes; otherwise 1804 returns `success=2` and the navigation stack is shut down. The failure reason falls into one of three cases: `Timeout to start navigation, please restart` (localization error did not converge), `Low pose score, please relocalize` (pose score below threshold), `Pose score service unavailable, please restart` (scoring service unavailable) |
| Other failure reasons | Invalid quaternion in the manual pose (`Invalid quaternion: ...`), scene does not exist or map files are missing (`Scene id <id> not exist or missing files: ...`), leftover navigation processes were not cleaned up (`Leftover nav processes detected, ...`), navigation was already shut down during odometry self-healing (`Navigation shutdown during odom restart`), the start-up was superseded by 1810 or by a new 1804 (`Nav was aborted before start, ...`), etc. |
| After success | The state is `NAV_READY` and a 1806 navigation goal can be dispatched |

---

### 5. Get Robot Pose State (1805)

**Description:** Returns the robot's current global pose and its estimation error.

**Request** — `/slam/get_global_pose`

```json
{
    "request_id": ""
}
```

| Parameter | Type | Description |
|------|------|------|
| `request_id` | string | Request ID |

**Response** — `/slam/status`

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

`data` is a JSON string; once parsed its structure is as follows:

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

| Parameter | Type | Description |
|------|------|------|
| `pose_error` | float | Pose estimation error (meters); `1000` when localization has not started |
| `global_pose` | object | Global pose (structure in [Pose Data Structure](#pose-data-structure)) |

| Item | Description |
|----|------|
| Precondition | Only `NAV_READY`, `NAV_RUNNING`, `NAV_PAUSED` and `NAV_STOPPING` are available; otherwise `Navigation invalid, no global pose prepared to response` is returned |

> `global_pose` is taken from the most recent `map -> base_link` TF; `pose_error` is `1000` when localization has not started.

---

### 6. Nav to Pose (1806, asynchronous)

**Description:** Call this interface to start navigation. The relative coordinate frame (ego) and the absolute coordinate frame (map) are both supported.

**Request** — `/slam/nav_to_pose`

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

| Parameter | Type | Description |
|------|------|------|
| `request_id` | string | Request ID, used to align request and response; kept identical across pause and resume |
| `relative` | bool | `false` — absolute coordinate frame (map); `true` — relative coordinate frame (ego, transformed against the `map -> base_link` TF at the moment the request is received) |
| `target_pose` | object | Target pose (structure in [Pose Data Structure](#pose-data-structure)); omitted fields are treated as `x/y/z=0`, `q_w=1` |

> **After the request is accepted:** the SDK first publishes a zero velocity command, clears the global/local costmaps, and then dispatches the goal to Nav2.

**Response** — `/slam/status`

When the request is accepted, an intermediate state is returned first:

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

When navigation finishes, the final result is returned with the same `request_id`:

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

| Item | Description |
|----|------|
| Precondition | Only `NAV_READY` (new goal) or `NAV_PAUSED` (resume the original goal); in `NAV_RUNNING` the current goal must be cancelled first (1809) before a new goal can be dispatched |
| Error constraint | In `NAV_READY`, if the localization error exceeds `nav_reject_err_thr` (parameter-file default 0.4 meters; built-in code default 0.3 meters) the request is rejected: `Localization error too high (...), please re-localize` |
| State-dependent rejections | `Navigation not started, invalid request` (`IDLE`), `Navigation is starting, please wait for a moment` (`NAV_STARTING`), `Navigation running, please cancel current goal before new request` (`NAV_RUNNING`), `Navigation is pausing, please wait` (`NAV_PAUSING`), `Navigation paused, please resume before request` (`NAV_PAUSED` and the `request_id` is not that of the paused original navigation), `Navigation not ready, invalid request` (any other state) |
| Goal failure reasons | `Goal was rejected`, `Goal was cancelled`, `Failed to reach the goal` (aborted / not reached), `Failed to connect nav server`, `Failed to lookup relative transform` (TF lookup failed during relative navigation), `Failed to figure out result` |
| Automatic timeout | A single navigation longer than `max_nav_time` (default 240 seconds) is cancelled automatically by the SDK node, returning `Goal was cancelled` |
| Resume after pause | Only accepted in `NAV_PAUSED` with the original `request_id` (that is, the internal resume performed by 1808); a new goal sent directly from outside is rejected |

---

### 7. Pause Navigation (1807)

**Description:** Call this interface to pause the current navigation task. Internally it is implemented by cancelling the current navigation goal; after a successful pause the goal is retained and can be resumed with 1808.

**Request** — `/slam/pause_nav`

```json
{
    "request_id": ""
}
```

| Parameter | Type | Description |
|------|------|------|
| `request_id` | string | Request ID; the pause result is reported using this request ID |

**Response** — `/slam/status`

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

| Item | Description |
|----|------|
| Precondition | Only available in `NAV_RUNNING`; otherwise `Failed to pause because navigation not running` is returned |
| After success | The state is `NAV_PAUSED` and navigation can be resumed with 1808; the goal cancellation triggered by the pause does not report a 1806 failure |
| Failure reasons | `Failed to pause navigation` (cancel failed), `No goal to cancel` |
| Edge case | If the goal happens to have already finished when the pause is requested, it is treated as a successful pause with the message `Navigation paused (goal already finished)` |

---

### 8. Resume Navigation (1808)

**Description:** Call this interface to resume a paused navigation task. Internally the original navigation goal is dispatched again (in absolute coordinates).

**Request** — `/slam/resume_nav`

```json
{
    "request_id": ""
}
```

| Parameter | Type | Description |
|------|------|------|
| `request_id` | string | Request ID (used only for the intermediate state; the final result reuses the `request_id` of the original navigation) |

**Response** — `/slam/status`

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
    "request_id": "<request_id of the original navigation>",
    "scene_id": "test",
    "task_id": 1808,
    "success": 1,
    "message": "Resume success",
    "data": null
}
```

| Item | Description |
|----|------|
| Precondition | Only available in `NAV_PAUSED` and when a saved goal exists; otherwise `Goal not paused, failed to resume` / `Goal not received, failed to resume` is returned |
| `request_id` convention | The intermediate state with `success=0` uses the `request_id` of this resume request; the final result uses the `request_id` of the **original navigation goal** (a resume re-dispatches the original goal, so it can be judged together with the final result of 1806) |
| Failure reasons | `Resume failed: goal rejected` (goal rejected), `Navigation paused, please resume before request` (a new goal dispatched while paused) |

---

### 9. Cancel Navigation (1809)

**Description:** Call this interface to cancel the current navigation task.

**Request** — `/slam/cancel_nav`

```json
{
    "request_id": ""
}
```

| Parameter | Type | Description |
|------|------|------|
| `request_id` | string | Request ID |

**Response** — `/slam/status`

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

| Item | Description |
|----|------|
| Precondition | `NAV_RUNNING` or `NAV_PAUSING`; cancelling while paused (`NAV_PAUSED`) returns `Cancel succeeded while paused` and additionally reports a 1806 failure (`Navigation cancelled while paused`) |
| After success | The state returns to `NAV_READY` |
| Failure reasons | `Navigation not running`, `No goal to cancel`, `Failed to cancel goal` |

---

### 10. Shutdown SLAM (1810)

**Description:** Call this interface to shut down the lidar driver, localization, odometry and navigation functions.

**Request** — `/slam/shutdown`

```json
{
    "request_id": ""
}
```

| Parameter | Type | Description |
|------|------|------|
| `request_id` | string | Request ID |

**Response** — `/slam/status`

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

| Item | Description |
|----|------|
| Precondition | Only `NAV_STARTING`, `NAV_READY`, `NAV_RUNNING`, `NAV_PAUSING` and `NAV_PAUSED` are available; otherwise `Navigation not ready` is returned |
| After success | The state returns to `IDLE`, `scene_id` is cleared, and 1801/1804 can be issued again |
| Failure case | If leftover navigation processes are still detected after shutdown, a failure is returned: `Navigation stopped but leftover processes exist` |

---

### 11. Get Navigation State (1814)

**Description:** Query the real-time state of the current navigation task, including the goal pose, elapsed navigation time and remaining distance.

**Request** — `/slam/get_nav_state`

```json
{
    "request_id": "000001"
}
```

| Parameter | Type | Description |
|------|------|------|
| `request_id` | string | Request ID |

**Response** — `/slam/status`

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

`data` is a JSON string; once parsed its structure is as follows:

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

| Parameter | Type | Description |
|------|------|------|
| `goal_pose` | object | Navigation goal pose (absolute coordinates, in meters/radians) |
| `navigation_time` | float | Elapsed navigation time (seconds) |
| `distance_remaining` | float | Remaining distance (meters) |

| Item | Description |
|----|------|
| Precondition | Only available in `NAV_RUNNING`; otherwise `Robot not in navigation state` is returned |
| Other failure reasons | `Never received a goal pose` (no goal received), `Empty navigation feedback, navigation finished or not started` (feedback empty; navigation has finished or has not started) |

---

### 12. SLAM State Query (1815)

**Description:** Query the current running state of the SLAM system.

**Request** — `/slam/get_state`

```json
{
    "request_id": ""
}
```

| Parameter | Type | Description |
|------|------|------|
| `request_id` | string | Request ID |

**Response** — `/slam/status`

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

| Item | Description |
|----|------|
| Precondition | No restriction; available in any state |
| `scene_id` | Mapping states (`MAPPING_*`) return the mapping scene ID; other states return the current navigation scene ID (`null` when not initialized) |

**`data` state enumeration:**

> For the transitions between states and the interfaces available in each state, see [SDK State Machine](./SDK_STATE_MACHINE.md).

| Value | Description |
|------|------|
| `IDLE` | Idle (no mapping, no localization) |
| `MAPPING_STARTING` | Starting mapping |
| `MAPPING_RUNNING` | Mapping in progress |
| `MAPPING_STOPPING` | Stopping mapping (saving the map) |
| `MAPPING_TERMINATING` | Force-terminating mapping |
| `NAV_STARTING` | Starting localization and navigation |
| `NAV_READY` | Navigation ready (localized, no goal dispatched) |
| `NAV_RUNNING` | Navigating |
| `NAV_PAUSING` | Pausing navigation (cancel request has been sent, awaiting the result) |
| `NAV_PAUSED` | Navigation paused |
| `NAV_STOPPING` | Stopping navigation |

> During an odometry restart the node internally sets `freezing` (it is not returned as a state value). In that window only 1804 (relocalize) and 1810 (shutdown) are allowed; all other commands are rejected. See [1816](#13-restart-odometry-1816) for details.

---

### 13. Restart Odometry (1816)

**Description:** Finely grained odometry restart (only the odometry is restarted; localization, navigation and the lidar are not). After success it relocalizes automatically and resumes navigation if necessary. It is used to recover from odometry anomalies (heartbeat lost, drift).

**Request** — `/slam/restart_odom`

```json
{
    "request_id": "<request ID>",
    "scene_id": "<scene ID>",
    "relocalize": true
}
```

| Parameter | Type | Description |
|------|------|------|
| `request_id` | string | Request ID, used to align request and response |
| `scene_id` | string | Scene ID (not validated in the current implementation; may be omitted) |
| `relocalize` | bool | Whether to perform a (range-limited) global relocalization; defaults to `true`. See "relocalize semantics" below |

**relocalize semantics** (it only affects whether the BBS global relocalization is called; both values re-anchor to the pose from before the restart)

| Value | Applicable scenario | Behavior |
|------|----------|------|
| `true` (default) | Triggered by SDK automatic monitoring (odometry drift / heartbeat lost); the external caller is not sure whether the robot has moved | First re-anchor `map->camera_init` with the pre-restart pose P, then perform a range-limited global relocalization (BBS) centered on P, using the current point cloud measurements to correct any translation/rotation caused by inertia or message delay during the restart; on success it announces "odometry recovered" |
| `false` | The caller is confident the robot has not moved (for example the tour feature's "same-scene map switch", called after each task ends) | It still re-anchors `map->camera_init` with P (after an odometry restart `camera_init` returns to a new origin, and without re-anchoring the robot would jump to the old odometry origin), but it skips the global relocalization search and keeps the previous localization result. This is a routine restart initiated by the caller, so on success it is **silent** (to avoid the user hearing a recovery announcement without having noticed any anomaly) |

> After the odometry process restarts, the `camera_init` origin becomes the robot's current position. Therefore, regardless of the `relocalize` value, a `/initialpose` must be published once (request_id = `odom_restart_relocalize`) to re-anchor; `relocalize` only decides whether an additional global search is performed.

**Response** — `/slam/status`

```json
{
    "request_id": "<request ID>",
    "scene_id": "test",
    "task_id": 1816,
    "success": 1,
    "message": "Odom restarted",
    "data": null
}
```

| Item | Description |
|----|------|
| Precondition | Only localization/navigation states (`NAV_STARTING`, `NAV_READY`, `NAV_RUNNING`, `NAV_PAUSING`, `NAV_PAUSED`) are available; otherwise `Odom restart ignored: abnormal state` is returned |
| Re-entrancy protection | A repeated request while a restart is in progress returns `Odom restart ignored: already restarting`; externally triggered restarts are not subject to the automatic-restart cooldown |
| Execution flow | ① If navigating (`NAV_RUNNING`), pause the current navigation first; ② restart the odometry process and wait for it to resume publishing (the script retries automatically up to `odom_restart_max_fail` times, default 3); ③ reuse 1804 manual relocalization (with the pre-restart pose P as the initial pose); ④ when `relocalize=true`, limit the planar search range around P (`odom_restart_search_range`, default 3 m, 0=disabled) and cap the BBS thread count to `odom_restart_threads` (default 1, 0=unchanged), then call the global initialization once (`/global_initialpose_estimation`), afterwards restoring the original range and thread count; when `relocalize=false` this step is skipped; ⑤ once the pose score check passes: if navigation was in progress the original goal is resumed automatically, otherwise the state returns to `NAV_READY` |
| During the restart | The node enters the `freezing` state; every command except 1804 (relocalize) and 1810 (shutdown) is rejected: `Odom recovering, only relocalize/shutdown allowed` |
| Failure reasons | `Odom restart ignored: ...` (state not eligible / re-entrant / cooling down), `Odom restart aborted: navigation shutdown` (shut down by 1810 during the flow), `Low pose score after odom restart: ...` (score below threshold after relocalization; if a navigation was in progress the original goal also reports a 1806 failure), `Pose score service call failed after odom restart...` (scoring service unavailable), `Odom restart failed, full shutdown` (script retries still failed, so a full shutdown is performed instead) |
| Automatic recovery | With `relocalize=true`, a successful restart plays the "odometry recovered" voice prompt; with `false` it is silent. On failure it plays the "navigation abnormal, please take over" voice prompt and gives up the fine-grained recovery |
| Relationship with 1804 | The relocalization after the restart reuses the 1804 flow internally and produces an extra `1804` status (request_id = `odom_restart_relocalize`), which clients may ignore |

---

## Automatic Monitoring and Protection

The following behaviors are triggered by the SDK node automatically, without any external request. Automatic triggers use fixed sentinel `request_id` values (heartbeat loss: `odom_lost_caused_restart`; drift: `odom_drift_caused_restart`; navigation timeout: `timeout_caused_cancel`; mapping drift: `drift_caused_terminate`). Their results are also reported on `/slam/status`; clients can recognize and ignore/log them by these fixed values, and must not treat them as requests they issued.

| Mechanism | Trigger condition | Action |
|------|----------|----------|
| Odometry heartbeat monitoring | After navigation becomes ready / starts running, no odometry data for longer than `odom_run_wait_time` (default 1 second) (a grace period of `odom_wait_time`, default 30 seconds, applies during start-up) | Announces "odometry lost" and performs the 1816 restart flow automatically |
| Odometry drift detection | The speed derived from two consecutive odometry positions exceeds `odom_drift_thr` (default 5.0 m/s), for `odom_drift_confirm_n` (default 5) consecutive abnormal frames | Announces "odometry drift" and performs the 1816 restart flow automatically |
| Automatic restart cooldown | The interval between non-externally-triggered restarts is shorter than `restart_cooldown` (default 10 seconds) | The restart is skipped, preventing a restart storm |
| Consecutive restart failures | The script's internal retries (`odom_restart_max_fail`, default 3) still fail | Gives up the fine-grained restart and performs a full shutdown (1801/1804 must be issued again) |
| Range-limited relocalization after restart | The odometry restart succeeds (publishing resumes after the script retries) | Sets the BBS planar search range centered on the pre-restart pose P (`odom_restart_search_range`, default 3 m, 0=disabled) and the thread count (`odom_restart_threads`, default 1, 0=unchanged), calls the global initialization once, then restores the original search range and thread count |
| Navigation timeout protection | A single navigation exceeds `max_nav_time` (default 240 seconds) | Announces "navigation timeout" and cancels the current navigation automatically (1806 returns a failure) |
| Mapping drift protection | During mapping, the odometry speed exceeds `map_drift_speed_thr` (default 5 m/s) for `map_drift_confirm_n` (default 3) consecutive frames | Announces "odometry drift" and performs 1822 to terminate mapping automatically (the map is not saved) |
| Recovery voice reminder | A new recovery event is received during navigation and more than `recovery_voice_cooldown` (default 15 seconds) has passed since the last announcement | Announces "hello, please make way" |

> Parameter file: `config/params/slam_sdk.yaml`; after modifying it, restart the SDK service for the changes to take effect.

### Collision Risk Topic

| Topic | Type | Description |
|------|------|------|
| `/slam/collision_risk` | `std_msgs/Float32` | Collision risk value, range 0~1 (higher means more dangerous), 1 Hz |

Computation: on the local costmap (`risk_costmap_topic`, default `/local_costmap/costmap`, global_frame = map), using the robot pose as the reference, sample the rectangular region **in front of** the robot with length `risk_range` (default 1.5 m) and `risk_half_width` to the left and right (default 0.4 m), skip unknown cells and take the average cost (0~100); the risk equals the average cost / `risk_cost_thr` (default 50), clamped to 1.
When the costmap or the pose is not ready, or there is no valid cell in front, the topic is not published (only a log entry is written); downstream consumers should tolerate missing messages.

### Internal Topics (for debugging; external reliance is not recommended)

| Topic | Type | Description |
|------|------|------|
| `/slam/status` | `std_msgs/String` | The unified response for all interfaces |
| `/slam/_state` | `std_msgs/String` | Current state machine state, broadcast at 10 Hz |
| `/slam/_nav_error` | `std_msgs/Float32` | Internal navigation error feedback (from DLL's `~/fitness_score`) |
| `/slam/collision_risk` | `std_msgs/Float32` | Collision risk (see the previous section) |
| `/cmd_vel` | `geometry_msgs/Twist` | The SDK's velocity output; `launch_sdk.sh` remaps it to `/cmd_vel_nav`, which is then published through `velocity_smoother` |

> **TF isolation:** The entire SLAM stack remaps `/tf` and `/tf_static` to `/slam/tf` and `/slam/tf_static`. External nodes that need to read this stack's TF must apply the same remapping (or subscribe to `/slam/tf` directly).

---

## Logging and On-Site Review

The SDK node writes **every request and every response** to its own log (`scripts/launch_sdk.sh` redirects it to `logs/slam_sdk.log`):

```bash
tail -f logs/slam_sdk.log | grep --line-buffered '\[API'
grep '\[API' logs/slam_sdk.log | tail -50
```

Format:

```
[API 1806 nav_to_pose] request_id=test_nav_to_pose scene_id=arena data={...}
[API 1806 nav_to_pose] response: {"request_id":"test_nav_to_pose","scene_id":"arena","task_id":1806,"success":0,"message":"Success to send nav request","data":null}
```

Common on-site commands:

```bash
slam_status                                  # watch /slam/status continuously
ros2 topic echo /slam/status --full-length   # do not truncate long data
slam_state                                   # current state
ls -lt logs/nav_fail | head                  # navigation failure snapshots (annotated.png + meta.json)
```

> For the complete log inventory, CPU monitoring and Foxglove debugging, see sections 7 and 8 of the [SDK Full Workflow Guide](./SDK_FULL_WORKFLOW_GUIDE.md).

---

## Navigation Workflow

> For the complete state machine (11 states, all state transitions, automatic protection mechanisms and the 1816 sub-flow), see [SDK State Machine](./SDK_STATE_MACHINE.md). The main call order is shown below:

```
Start mapping (1801)  ──→  Stop mapping (1802)
    │                           │
    └── Force terminate (1822)  │
                                ↓
                    Initialize pose (1804)
                                ↓
        Nav to pose (1806)  ←  Get pose state (1805)
                    ↓
            ┌── Pause (1807) → Resume (1808)
            ├── Cancel (1809)
            ├── Get nav state (1814)
            └── Restart odometry (1816) ──→ Relocalize (1804) ──→ Resume original goal (1808)
                    ↓
                Shutdown SLAM (1810)
```
