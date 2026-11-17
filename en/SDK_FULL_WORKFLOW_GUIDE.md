# SLAM SDK Full Workflow Guide (Mapping → Localization → Marking → Navigation)

> This is the English version of [SDK全流程操作手册.md](../SDK全流程操作手册.md).
>
> **Intended audience:** robot deployment, integration, and on-site operators
> **Scope:** the SLAM SDK provided by this repository (interface codes 1801~1822), deployment directory `/opt/app/slam`
> **Companion docs:** [SDK Interface](./SDK_INTERFACE.md) · [SDK State Machine](./SDK_STATE_MACHINE.md) · [SDK Call Examples](./SDK_EXAMPLES.md) · [Key Parameters](./SDK_KEY_PARAMETERS.md)
> **Last updated:** 2026-11-16

---

## 0. Read this page first

### 0.1 Safety first

1. **The robot really moves.** `1806 nav_to_pose`, `1808 resume_nav`, and the automatic recovery after odometry self-healing all make the chassis move autonomously. Before operating, confirm that there is nobody and no obstacle around the robot and that the remote controller is at hand for immediate takeover.
2. **When not navigating, switch the chassis to remote-control mode and keep the remote controller at hand.** Do not leave the robot in a "navigation mode + SDK idle" state for a long time: a mistakenly sent 1806, a retry from the upstream, or the automatic recovery after 1816 self-healing may all make the robot start moving suddenly.
3. **Never move joints when CPU usage is high.** On the robot, the SLAM algorithms and the reinforcement-learning locomotion controller usually share the same compute unit. When the CPU is saturated, the policy inference rate drops and the control period lengthens, so the robot can no longer keep its balance and falls. Therefore:
   - before standing up, stepping, or switching to navigation mode, confirm that the CPU has headroom (see 7.3);
   - **when CPU usage is high, do not move joints and do not send walking commands, including from the remote controller**;
   - first cancel navigation and close visualization/debugging tools to reduce the load, and move only after the CPU drops and the control period recovers.
4. **Emergency takeover depends on CPU state.** When there is a collision or a person at risk:
   - CPU normal: switch the remote controller to manual/remote mode immediately to take over; this is faster than sending 1809 from a terminal;
   - CPU already high: **do not drive it with the remote controller**; send 1809 to cancel navigation and close visualization to reduce the load, and keep people away; handle it after control recovers;
   - after takeover the SDK may detect an odometry/localization anomaly and trigger 1816 or return `Goal was cancelled`; this is expected. Once the robot has come to a safe, stable stop, send 1809 to clean up the navigation state and 1804 to relocalize as needed, and continue only after confirming `NAV_READY`.
5. **Keep the robot still when mapping or localization starts.** After 1801 you must wait for `/cloud_registered` to appear (15 s), and after 1804 you must wait for localization to converge (90 s maximum by default); moving the robot during these periods causes a startup failure or a wrong localization.
6. **Do not start the SDK twice.** Only one set of SDK nodes should run on one machine; to switch or restart, use the provided launch script or service command, do not start it by hand again.

### 0.2 Four key understandings

| Understanding | Description |
|------|------|
| **The SDK is only topic + JSON** | Requests are published to `/slam/xxx` (`std_msgs/String`, where `data` is a JSON string) and responses always come back through `/slam/status`. There is no REST API or SDK library. |
| **`success=0` is not a terminal state** | `0` = the request has been accepted; you must wait, matched by `request_id`, for `success=1` (success) or `2` (failure). |
| **The state machine decides whether you may send** | Mapping is only allowed in `IDLE`; navigation is only allowed in `NAV_READY`; `MAPPING_*` and `NAV_*` are mutually exclusive. Check `slam_state` before operating. |
| **The map is a contract, and parameter changes need a restart** | The scene directory must contain `map.pcd`, `map.yaml`, and `map.pgm`; `data/maps/out` points at the current scene. After changing parameters, the SDK (or the corresponding service) must be restarted for the change to take effect. |

> All states, transitions, and automatic protections are documented in the [SDK State Machine](./SDK_STATE_MACHINE.md); when you need to adjust thresholds, read [Key Parameters](./SDK_KEY_PARAMETERS.md) first.

### 0.3 One-page shortest workflow (copy and paste)

```bash
source /opt/app/slam/.bashrc          # once per new terminal
bash scripts/launch_sdk.sh            # start lidar + SDK (on a deployed machine: systemctl restart slam_sdk.service)
slam_state                            # expect IDLE

slam_start_mapping site_a             # 1801; wait for success=1, then push the robot around the site
slam_stop_mapping site_a              # 1802; wait for success=1

slam_init_pose site_a                 # 1804; wait until the state becomes NAV_READY
slam_mark entrance                    # mark a waypoint (optional)
slam_nav_to_pose 5.0 3.0 1.57         # 1806
slam_shutdown                         # 1810
```

How the state progresses:

```
IDLE ──1801──> MAPPING_* ──1802──> IDLE
  │
  └──1804──> NAV_READY ──1806──> NAV_RUNNING ──(reached/failed)──> NAV_READY
                                    └──1807 / 1808 / 1809──> NAV_PAUSED / NAV_READY
```

> Criteria for each step, failure reasons, and walking rules are in chapters 2~5; the full command list is in 11.1;
> for hand-written `ros2 topic pub` payloads see [SDK Call Examples](./SDK_EXAMPLES.md).

---

## 1. Environment and startup

### 1.1 Login and environment

On the deployment machine the project lives at `/opt/app/slam`. After logging in, run this once:

```bash
source /opt/app/slam/.bashrc
```

It sets `SLAM_ROOT`, the ROS environment variables, and loads the `slam_*` shortcut commands.

> You need to run `source` once in every newly opened terminal.

### 1.2 Self-check list

```bash
echo "$SLAM_ROOT"                 # should be /opt/app/slam
ros2 pkg prefix slam_control_c    # can locate the install directory
nproc                             # number of CPU cores (used later to judge load)
free -h                           # memory
df -h "$SLAM_ROOT/data/maps"      # map disk space
ls -l "$SLAM_ROOT/data/maps/out"  # the current scene symlink
```

### 1.3 Start the SDK

Deployment machines are usually configured to start on boot; check first:

```bash
systemctl status slam_sdk.service
```

When you need to start or restart manually:

```bash
# Script
bash /opt/app/slam/scripts/launch_sdk.sh

# Or service (deployed machine)
systemctl restart slam_sdk.service
```

What it starts: the lidar driver + the SDK node. Logs: `logs/livox.log`, `logs/slam_sdk.log`.

After about 10~30 s, confirm:

```bash
slam_state          # equivalent to ros2 topic echo /slam/_state --once
# expected output: IDLE
```

> **Localization has not happened yet, so you cannot navigate.** You must run 1804 to enter `NAV_READY`.

### 1.4 Open three terminals (recommended on-site working style)

| Terminal | Purpose | Command |
|------|------|------|
| A | Start/restart (one-off) | `bash scripts/launch_sdk.sh` or `systemctl restart slam_sdk.service` |
| B | **Keep watching interface responses** | `slam_status` (or `ros2 topic echo /slam/status --full-length`) |
| C | Send commands / check state / watch CPU | `slam_state`, `slam_global_pose`, `top` |

> All requests and responses are also written to `logs/slam_sdk.log`, so when you are too busy on site to watch a terminal you can grep them afterwards (see section 7).

---

## 2. Mapping (1801 → 1802)

### 2.1 Standard flow

```bash
# 1) Confirm it is idle
slam_state                      # must be IDLE

# 2) Start mapping; scene_id is up to you (English letters + digits recommended, e.g. site_a_20261116)
slam_start_mapping site_a

# 3) Wait for the 1801 terminal state
#    /slam/status shows task_id=1801, success=1, message="mapping started"
slam_state                      # expected MAPPING_RUNNING

# 4) Push the robot around the whole site (or drive it slowly by remote control)

# 5) Finish mapping and save
slam_stop_mapping site_a
#    expected task_id=1802, success=1, message="Map saved"
slam_state                      # back to IDLE
```

### 2.2 Site-walking rules (they directly determine the localization success rate)

1. **Speed ≤ 0.5 m/s**, slower on turns; no sudden stops or turns, no fast spinning in place.
2. **Close the loop**: come back near the start point at the end so the point cloud revisits the area, then check the map for "ghosting".
3. **Full coverage**: walk to the ends of corridors and to every doorway; do not walk only the main aisle.
4. **Avoid degenerate scenes**: long straight corridors, large glass/white walls, and empty halls easily cause mapping drift; try to keep corners, door frames, furniture, and other features in the lidar's view.
5. **Do not go up or down stairs** (the 2D occupancy grid does not apply); do not map while many people are walking around.
6. **Keep the network and power stable during mapping**; an occluded lidar or a data-stream dropout directly causes a mapping failure.

### 2.3 Monitoring during mapping

```bash
# State
slam_state
# Whether the point cloud / odometry keeps flowing
ros2 topic hz /cloud_registered
ros2 topic hz /aft_mapped_to_init
# CPU (detailed in section 7)
top -b -n1 -o %CPU | head -15
```

### 2.4 Artifacts of finishing mapping

```
data/maps/<scene_id>/
├── map.pcd      # 3D point-cloud map: used by localization (BBS/DLL)
├── map.pgm      # 2D occupancy grid: used for localization validation and as the navigation base map
└── map.yaml     # metadata such as resolution/origin
data/maps/out -> <scene_id>     # symlink: points at the most recently saved scene
```

Optional files (platform post-processing, not required by the SDK):

- `map_nav.yaml` / `map_nav.pgm`: navigation-only P map; when present, navigation uses it, while **localization and localization validation still use the original map**.
- `keepout.yaml` / `keepout.pgm`: keep-out zones; only enabled with `use_keepout_zone:=True`.

Check:

```bash
ls -l data/maps/out
ls -l data/maps/$(basename "$(readlink -f data/maps/out)")
slam_latest_scene               # prints the most recently saved scene_id
```

### 2.5 Mapping failure / abort

| Situation | Symptom | Handling |
|------|------|------|
| Startup failure | 1801 `success=2 "mapping start failed"` | Check `logs/livox.log`, `logs/odom.log`; confirm the lidar is online and that `/cloud_registered` appears; clean up and retry |
| Mapping aborted | Voice prompt "里程计漂移" (odometry drift), the state returns to `IDLE` automatically, and 1802 is no longer available | This map is not trustworthy; start 1801 again; check whether you pushed too fast, slipped, or blocked the lidar |
| Giving up deliberately | Send 1822 | `slam_terminate_mapping site_a`, **the map is not saved**, the state returns to `IDLE` |
| Save failure | 1802 `success=2 "Missing map files"` / `"Copy map error"` | Check `logs/slam_sdk.log`; check the permissions and free space of `data/maps/<scene>`; `out` will point at `_ERROR` |

---

## 3. Localization (1804)

> Localization = bringing up BBS global relocalization + DLL tracking + Nav2 + pose_verifier. On success the state becomes `NAV_READY`, and only then can you navigate.

### 3.1 Automatic pose estimation (recommended)

```bash
slam_state                                  # must be IDLE (or update the pose in NAV_STARTING/READY/PAUSED)
slam_init_pose site_a                       # auto_init_pose=true
# wait for /slam/status: task_id=1804, success=1, message="Ready for navigation"
slam_state                                  # expected NAV_READY
```

### 3.2 Manually specifying the pose (when the start point is known)

```bash
slam_init_pose site_a 1.5 2.0 1.57
# argument meaning: scene_id x y yaw(radians)
```

Manual mode suits:

- the robot always departs from a fixed charging position;
- automatic relocalization fails (corridor/empty/similar-looking scenes) but you know where the robot is.

### 3.3 Waiting and criteria

- The wait is capped by `nav_wait_time` (default 90 s).
- The localization error `pose_error` converges below `pose_err_thr` (default 0.20 m).
- Wait another 3 s for the error to settle, then call `/slam/get_pose_score` for pose scoring; when it passes, the state becomes `NAV_READY`.

### 3.4 Failure causes and handling

| Returned message | Meaning | Handling |
|--------------|------|------|
| `Scene id <id> not exist or missing files: [map.pcd, map.yaml, map.pgm]` | The scene does not exist or files are missing | Check the three files under `data/maps/<id>/`; first save once properly with 1802 |
| `Request mismatch with current scene id: ...` | The scene is bound to navigation and cannot be switched | Run 1810 to shut down first, then send 1804 with the new `scene_id` |
| `Leftover nav processes detected, clean and retry: ...` | The previous navigation processes were not cleaned up completely | Run 1804 again (it cleans up internally first), or `bash scripts/kill_nav.sh` |
| `Timeout to start navigation, please restart` | The localization error did not converge within 90 s | Check `logs/dll.log`, `logs/nav2.log`; confirm the lidar point cloud is normal and the CPU is not overloaded; retry or give the pose manually |
| `Low pose score, please relocalize` | The pose score does not meet the threshold (localization may have drifted) | Move the robot to a feature-rich position and run `slam_init_pose` again; or give the pose manually |
| `Pose score service unavailable, please restart` | The pose_verifier service is unavailable | Check in `logs/nav2.log` whether pose_verifier started; restart the SDK |
| `Navigation shutdown during odom restart` | Navigation was shut down during odometry self-healing | Wait until self-healing finishes, then run 1804 again |

### 3.5 Self-check after localization

```bash
slam_global_pose
# in the response data, check pose_error (should be < 0.3) and global_pose

slam_nav_state                  # you are not navigating now, so it returns a failure; this is normal
```

In Foxglove, checking whether `/cloud_registered` and `/map` coincide (see section 8) is the most intuitive check that localization is correct.

---

## 4. Marking (recording waypoints)

The SDK itself has no "mark" interface; marking = **reading the current pose and recording it**. There are two ways:

### 4.1 Option 1: `slam_mark` (recommended, pure SDK)

```bash
slam_mark entrance              # recorded to logs/sdk_marks.csv
```

Example output:

```
entrance,1.2345,2.3456,0.7854,0.1200
→ 导航命令: slam_nav_to_pose 1.2345 2.3456 0.7854
已追加到 /opt/app/slam/logs/sdk_marks.csv
```

> The script prints these lines in Chinese: the second line is the navigation command it suggests, and the third confirms the append target.

- The state must be `NAV_READY` / `NAV_RUNNING` / `NAV_PAUSED` (the states in which 1805 is available).
- By default it appends to `logs/sdk_marks.csv` (columns: `name,x,y,yaw,pose_error`).
- If `pose_error > 0.3 m` it prints a warning: the waypoint is unreliable, relocalize first.
- Print only, do not write to disk: `slam_mark p1 --no-save`.
- Use a specific file: `slam_mark p1 --file /tmp/p1.csv`.

### 4.2 Option 2: the guide service `/guide_tour/mark` (requires the guide node)

```bash
bash scripts/launch_guide_services.sh
ros2 service call /guide_tour/mark reception_demo/srv/GuideTourMark \
  "{id: 'p1', annotations: 'entrance', is_build_map: false}"
```

The waypoint is written to `logs/marks.yaml`:

```yaml
waypoints:
  p1: {x: 1.2345, y: 2.3456, a: 0.7854}
```

> Note: the guide node **clears `logs/marks.yaml` on every startup**, so back it up yourself if you need to keep waypoints long-term; you can also mark during mapping (`is_build_map: true`), whereas the SDK's 1805 is only available in `NAV_*` states.

### 4.3 Marking rules

1. **Localize to `NAV_READY` first and confirm that `pose_error` is small before marking.**
2. Put waypoints in the middle of the aisle, at least 0.5 m from walls, and avoid positions right next to doorways/glass.
3. yaw is the robot's **facing direction** (radians, counter-clockwise positive); navigating to the point also aligns to that heading.
4. Name waypoints with English letters/digits/underscores, e.g. `lobby_01`, `room_a_door`.
5. After marking, spot-check 2~3 points by actually running `slam_nav_to_pose`, then collect the rest in bulk.

### 4.4 Batch-verifying waypoints

```bash
# Navigate point by point in order, staying 5 s at each arrived point; stop immediately on any anomaly
slam_cycle_test 1.2 2.3 0.78 4.5 2.3 0.0 -1.0 3.0 1.57
```

- `-d 10`: stay 10 s at each point; `-n 3`: run the whole set 3 rounds; `-n 0`: loop forever, interrupt with Ctrl-C; `-t 120`: per-point timeout 120 s (default 300).
- The next point is only sent after the previous one reports success; Ctrl-C/timeout automatically sends 1809 to cancel, so the robot does not keep running.
- The logic is in `scripts/slam_cycle_test.py`; exit codes `0` all reached, `1` anomaly, `2` bad arguments, `130` interrupted.

---

## 5. Navigation (1806 + 1807/1808/1809/1814)

### 5.1 Navigation to absolute coordinates

```bash
slam_state                      # must be NAV_READY
slam_nav_to_pose 5.0 3.0 1.57   # x(m) y(m) yaw(rad)
```

- Immediately returns `1806 success=0 "Success to send nav request"`.
- On arrival it returns `1806 success=1 "Success to reach the goal"`.
- On failure it returns `success=2`; see 5.4.

### 5.2 Navigation relative to the current pose

```bash
slam_nav_relative 2.0 0 0       # move 2 m forward along the current heading, keeping the heading
```

With `relative=true` the SDK converts the relative goal into absolute coordinates using the `map -> base_link` TF before sending it, so the TF must be healthy.

### 5.3 Control during a run

| Operation | Command | Description |
|------|------|------|
| Check progress | `slam_nav_state` | Returns the goal pose, elapsed navigation time, and remaining distance |
| Pause | `slam_pause_nav` | Cancels the current goal but keeps it; state `NAV_PAUSED` |
| Resume | `slam_resume_nav` | Resends the original goal; state back to `NAV_RUNNING` |
| Cancel | `slam_cancel_nav` | Back to `NAV_READY`, goal cleared |
| Change goal | `slam_cancel_nav` first, then `slam_nav_to_pose ...` | The goal cannot be changed directly in `NAV_RUNNING` |

> **Do not send a new goal after pausing**: in `NAV_PAUSED` only "resume" can enter navigation; other 1806 requests are rejected (`Navigation paused, please resume before request`).

### 5.4 Navigation failure causes

| message | Meaning | Handling |
|---------|------|------|
| `Navigation not started, invalid request` | Not localized (`IDLE`) | Run 1804 first |
| `Navigation is starting, please wait for a moment` | Localization/the navigation stack is still starting | Wait for `NAV_READY` |
| `Navigation running, please cancel current goal before new request` | Navigation is running | Send 1809 first |
| `Navigation is pausing, please wait` | Pause transition state | Wait for `NAV_PAUSED` |
| `Navigation paused, please resume before request` | A new goal was sent while paused | Resume with 1808, or send 1809 first |
| `Localization error too high (x.xxx), please re-localize` | Localization error > `nav_reject_err_thr` (default 0.4 m) | Run 1804 again or check localization |
| `Goal was rejected` | Nav2 rejected the goal (e.g. the goal is inside an obstacle or outside the known area) | Check whether the goal is reachable; see `logs/nav2.log` |
| `Failed to connect nav server` | The Nav2 action server is absent | Check `logs/nav2.log` and whether it was killed; restart navigation |
| `Failed to lookup relative transform` | TF lookup failed during relative navigation | Check `/slam/tf`, `map -> base_link` |
| `Goal was cancelled` / `Failed to reach the goal` | Cancelled / aborted | Look at the scene image in `logs/nav_fail/` |
| `Failed to figure out result` | Result parsing error | Retry; if it persists, check `logs/slam_sdk.log` |

### 5.5 Automatic protections (no action needed from you)

- **Navigation timeout**: a single run longer than `max_nav_time` (default 240 s) is cancelled automatically and 1806 returns `Goal was cancelled`.
- **Odometry heartbeat loss/drift**: 1816 is executed automatically (restart odometry + relocalize); on success the original goal is resumed, on failure the SDK shuts down completely.
- **Mapping drift**: if the speed keeps exceeding the threshold during mapping (default 5 m/s × 3 frames), 1822 automatically terminates mapping.
- **Collision risk**: `/slam/collision_risk` (0~1, 1 Hz) can be used by the upstream to slow down or stop.

### 5.6 Ending navigation

```bash
slam_shutdown                   # 1810: shut down lidar, localization, and navigation, back to IDLE
```

After that the `scene_id` is unbound; you can start 1801/1804 again. **After finishing the task, switch the chassis back to remote-control mode.**

---

## 6. Typical full workflow (one task)

```bash
source /opt/app/slam/.bashrc
bash scripts/launch_sdk.sh                  # or systemctl restart slam_sdk.service
slam_state                                  # IDLE

# ---- Mapping (only needed once; the map can be reused) ----
slam_start_mapping arena
# push/drive around the site at speed ≤0.5 m/s, closing the loop
slam_stop_mapping arena
ls -l data/maps/out                         # out -> arena

# ---- Localization ----
# Put the robot at the corresponding position in the map and keep it still
slam_init_pose arena
slam_state                                  # NAV_READY

# ---- Marking ----
slam_mark start_point
slam_mark goal_point
cat logs/sdk_marks.csv

# ---- Navigation ----
slam_nav_to_pose <goal_x> <goal_y> <goal_yaw>
slam_nav_state                              # check the remaining distance
# after arrival /slam/status shows success=1

# ---- Wrap up ----
slam_shutdown
# switch the chassis back to remote-control mode; keep the remote controller at hand
```

---

## 7. Monitoring, logs, and CPU

### 7.1 Watching interface responses: `/slam/status`

**Keep one terminal open for this:**

```bash
slam_status
# equivalent to
ros2 topic echo /slam/status --full-length
```

- `--full-length` prevents long strings from being truncated (`data` in the JSON can be long).
- Every response contains `request_id`, `task_id`, `success`, `message`.
- **Only look at terminal states where `success` is 1/2**; `success=0` merely means accepted.

**Responses are also written to the log**, which suits later review:

```bash
tail -f logs/slam_sdk.log | grep --line-buffered '\[API'
# or
grep '\[API' logs/slam_sdk.log | tail -50
```

A request/response pair in the log looks roughly like this:

```
[API 1806 nav_to_pose] request_id=test_nav_to_pose scene_id=arena data={...}
[API 1806 nav_to_pose] response: {"request_id":"test_nav_to_pose",...,"success":0,...}
```

### 7.2 Watching the state machine

```bash
slam_state                                  # current state (broadcast at 10 Hz, one sample)
ros2 topic echo /slam/_state                # keep observing
```

For the meaning of the states see the [SDK State Machine](./SDK_STATE_MACHINE.md).

### 7.3 Watching CPU usage (key)

**On the robot, the SLAM algorithms and the reinforcement-learning locomotion controller usually share the same compute unit. When the CPU is saturated, the policy inference rate drops and the control period lengthens, so the robot can no longer keep its balance and falls. CPU is therefore not just a performance metric but a safety prerequisite: do not move joints or send walking commands (including from the remote controller) when CPU is high; reduce the load first.**

```bash
# Overall
top
# One-shot snapshot (sorted by CPU)
top -b -n1 -o %CPU | head -20
# Load (a load average persistently > the core count means overload)
uptime
nproc
# Orin platform
tegrastats
```

Judge "who is eating the CPU" by phase:

| Phase | Main CPU consumers | Description |
|------|------------------|------|
| Mapping | `fastlivo_mapping_t4` / `laserMapping` | Mapping itself is heavy; if it keeps saturating a single core, slow down the site walk |
| Localization startup | `bbs3d_register_node` | The global search is the most CPU-hungry part (multi-threaded); being busy for tens of seconds is normal |
| Navigation | `controller_server` / `planner_server` / `dll_node` / `bt_navigator` | Noticeable when planning is frequent or the costmaps are large |
| Locomotion (RL) | Policy inference / joint-control process (actual name depends on deployment) | Shares CPU with SLAM; a drop in inference rate directly causes loss of balance and falls |
| Visualization | `viz_node_c` (WebUI), `foxglove_bridge` | Can be turned off after debugging to free CPU |

Rules of thumb:

- Total CPU persistently > 80%, or load average > `nproc`: **do not stand up, step, or send walking commands (including from the remote controller)**; cancel navigation and close visualization first, and move only after the CPU drops and the control period recovers.
- If the robot is already walking, send 1809 to cancel navigation and let it stop; **do not keep driving it with the remote controller while CPU is high**.
- If localization startup has not reached `NAV_READY` after 30 s, first use `top` to see whether BBS is running, then look at `logs/dll.log` / `logs/nav2.log`.
- To reduce the BBS cost: lower `bbs_leaf_budget` in `config/params/slam_sdk.yaml` (coarser/faster), or temporarily reduce `bbs.max_num_threads`; restart the SDK (or re-run 1804) afterwards. For navigation speed levels see `config/params/speed_override.yaml` (currently pointing at `medium`).

> For the meaning, defaults, and how changes take effect for tunable parameters, see [Key Parameters](./SDK_KEY_PARAMETERS.md).

### 7.4 Log map

| File/directory | Content | When to look |
|-----------|------|------------|
| `logs/slam_sdk.log` | SDK requests/responses, state transitions, errors | **First priority** |
| `logs/livox.log` | Lidar driver | No point cloud / lidar offline |
| `logs/odom.log` | Mapping/odometry process output | Mapping failure, odometry anomalies |
| `logs/nav2.log` | All Nav2 nodes + pose_verifier | Navigation startup failure, planning/control anomalies |
| `logs/dll.log` | DLL tracking localization | Large localization error, `pose_error` not converging |
| `logs/guide_tour.log` | Guide node (if enabled) | Problems with `/guide_tour/*` services |
| `logs/nav_fail/` | Navigation failure scene PNG + `meta.json` | Navigation "did not get there" — just look at the image |
| `logs/sdk_marks.csv` | Waypoints recorded by `slam_mark` | Marking |
| `logs/marks.yaml` | Waypoints recorded by `/guide_tour/mark` | Guide-side marking |

Common commands:

```bash
tail -f logs/slam_sdk.log
tail -f logs/nav2.log
ls -lt logs/nav_fail | head
grep -E "Timeout|Low pose score|Leftover|drift|Missing odom" logs/slam_sdk.log
```

The navigation failure scene dump is enabled by default (`fail_dump_enable: true` in `config/params/slam_sdk.yaml`); each failure directory usually contains `annotated.png` and `meta.json`.

> `logs/slam_sdk.log` is **appended** on every startup; with long runtimes or high-frequency 1805 polling the file keeps growing, so clean it up or archive it before a task.

### 7.5 Other useful observations

```bash
ros2 topic hz /cloud_registered          # lidar point cloud
ros2 topic hz /aft_mapped_to_init       # odometry
ros2 topic hz /slam/_state              # 10 Hz
ros2 topic echo /slam/collision_risk    # collision risk 0~1
ros2 node list                          # whether all nodes are present
```

---

## 8. Foxglove debugging (strongly recommended)

### 8.1 Installing and connecting

1. Install **Foxglove Studio** on the operator computer (`https://foxglove.dev`, the desktop version is enough).
2. The robot and the operator computer must be on the same network.
3. Start the bridge on the robot:

```bash
bash scripts/foxglove.sh
# port 8776, log logs/foxglove.log
```

4. In Foxglove choose **Open connection → Foxglove WebSocket**, with the address:

```
ws://<robot_ip>:8776
```

5. If it cannot connect:
   - `ss -lntp | grep 8776` to confirm the port is listening;
   - check `logs/foxglove.log`;
   - confirm that the firewall/network ACL allows 8776.

### 8.2 Recommended panel layout

| Panel | Topics | Purpose |
|------|------|------|
| 3D | `/cloud_registered`, `/map`, `/global_costmap/costmap`, `/local_costmap/costmap`, `/plan` | Check whether the point cloud and the map coincide, whether the costmap is blocked, whether the path is reasonable |
| 3D / TF | `/slam/tf`, `/slam/tf_static` | Check whether `map -> odom -> base_link` is continuous |
| Raw Messages | `/slam/status`, `/slam/_state` | Read responses and states one by one |
| Plot | `/slam/collision_risk` | Collision risk trend |
| Raw Messages | `/dll_node/...` | Internal DLL localization values (e.g. fitness) |

> **TF note:** this stack isolates TF: TF is published on `/slam/tf` and `/slam/tf_static`, not `/tf`. In the Foxglove 3D/TF settings point the TF topics at `/slam/tf`; if your version does not support custom TF topics, watch `/slam/tf` in Raw Messages, or use RViz directly.

### 8.3 What can it publish?

The `client_topic_whitelist` in `scripts/foxglove.sh` only allows clients to publish:

- `/initialpose`
- `/slam/cancel_nav`

Therefore **Foxglove is mainly for observation, not for sending navigation goals**. To send a goal manually use `slam_nav_to_pose`; if you really want to send from Foxglove, edit `CLIENT_TOPIC_WHITELIST` in `scripts/foxglove.sh` to add `/slam/nav_to_pose` and the like, then restart the bridge (and mind safety).

### 8.4 Recording bags for review

```bash
ros2 bag record -o logs/bag_$(date +%H%M%S) \
  /slam/status /slam/_state /slam/collision_risk \
  /cloud_registered /map /plan \
  /global_costmap/costmap /local_costmap/costmap \
  /slam/tf /slam/tf_static
```

---

## 9. Safety and on-site rules

### 9.1 Modes and takeover

- **When not navigating: the chassis in remote-control mode, with the remote controller at hand for immediate takeover.** The reason is in 0.1.
- **Before navigating:** confirm that the robot is in navigation mode and that neither the remote controller nor an upper-layer program is sending velocities at the same time; **confirm that the CPU has headroom** — do not stand up, step, or switch to navigation mode when CPU is high.
- **While navigating:** keep people at least 1 m away from the robot; assign a person to guide it in narrow aisles; keep watching CPU.
- **Emergency takeover:** when a collision risk appears, somebody steps in, or the robot accelerates abnormally or swerves:
  - CPU normal: **switch the remote controller to manual/remote mode immediately to take over**; do not first spend time sending cancel commands from a terminal;
  - CPU already high: **do not send walking commands from the remote controller**; send 1809 to cancel navigation and close visualization to reduce the load, keep people away, and handle it after control recovers.
- **After taking over:** once the robot has come to a safe, stable stop, send `slam_cancel_nav` to clean up the navigation state; if localization was disturbed, send 1804 to relocalize and confirm `NAV_READY` before continuing the task.

### 9.2 Pre-task checklist

- [ ] The lidar is firmly mounted with an unobstructed view, and the network cable/power supply are normal.
- [ ] The map is the latest version for the current site (`ls -l data/maps/out`).
- [ ] Localization succeeded and `pose_error` is stably < 0.3 m.
- [ ] 2~3 navigation points have been spot-checked and all are reachable.
- [ ] CPU/memory/disk headroom is normal and no processes are left over (`bash scripts/kill_nav.sh` before starting).
- [ ] **CPU load is not persistently > 80% and load average does not exceed the core count**; do not stand up, step, or send walking commands when CPU is high.
- [ ] The remote controller is at hand and the operator knows the takeover procedure.
- [ ] Foxglove or `slam_status` is connected, so responses are visible in real time.
- [ ] Voice prompts are audible (odometry loss/drift/navigation timeout/`请让一让` (please make way)).

### 9.3 In-run checklist

- [ ] Before every task `slam_state` is `NAV_READY` (not `IDLE`/`NAV_RUNNING`).
- [ ] Every 1806 has a terminal state: `success=1` reached, or `success=2` with a clear reason.
- [ ] CPU is not persistently maxed out; `/cloud_registered` and `/aft_mapped_to_init` rates are normal.
- [ ] **Do not move joints or send walking commands (including from the remote controller) when CPU is high**; cancel navigation and reduce the load first.
- [ ] If the robot stops abnormally or spins in place, look at `/slam/status` and `logs/nav_fail/` first; handle it as described in 9.1 (do not drive it with the remote controller when CPU is high).

---

## 10. Frequently asked questions (FAQ)

| Symptom | Possible cause | Handling |
|------|----------|------|
| `slam_state` stays `IDLE` and 1804 fails | Scene files missing / leftover processes / lidar not started | Check the three files under `data/maps/<scene>`; `bash scripts/kill_nav.sh`; check `logs/livox.log` |
| 1804 succeeds but it never reaches `NAV_READY` | Localization error not converging / score not passing | Check `logs/dll.log`, `pose_error`; move to a feature-rich position and relocalize; give the pose manually |
| Navigation "did not get there" | Localization drifted / costmap blocked / goal unreachable | Look at `annotated.png` in `logs/nav_fail/`; change the goal; relocalize |
| Navigation stops midway with the voice prompt "里程计丢失/漂移" | Odometry stream loss/slipping/control was taken over | Wait for 1816 self-healing; if needed `slam_cancel_nav` then 1804; check the lidar/wheel speeds |
| After 1816 it reports `Low pose score after odom restart` | The robot was moved during the restart | Run 1804 manually to relocalize; before the next task confirm `NAV_READY` |
| `/slam/status` receives nothing | SDK not started / DDS not working / QoS | Whether `slam_state` works; `ros2 topic list`; check `RMW_IMPLEMENTATION`, `ROS_DOMAIN_ID` |
| Foxglove cannot connect | Port/firewall/not started | `bash scripts/foxglove.sh`; `ss -lntp \| grep 8776`; check `logs/foxglove.log` |
| Parameter changes have no effect | The corresponding service was not restarted | After editing `config/params/`, restart the SDK (or re-run 1804); see [Key Parameters](./SDK_KEY_PARAMETERS.md) |
| The robot starts moving as soon as it powers on | An old goal/automatic recovery was not cleared / the chassis is still in navigation mode | Send 1809/1810 first; switch the chassis to remote control; check whether an upstream is resending 1806 |

---

## 11. Quick reference

### 11.1 Commands

```bash
# Environment
source /opt/app/slam/.bashrc

# Start/stop
bash scripts/launch_sdk.sh
bash scripts/launch_sdk.sh shut
systemctl restart slam_sdk.service        # deployed machine
bash scripts/launch_guide_services.sh
bash scripts/kill_nav.sh

# Mapping
slam_start_mapping <scene>
slam_stop_mapping <scene>
slam_terminate_mapping <scene>

# Localization
slam_latest_scene                         # scene id of the latest saved map: slam_init_pose $(slam_latest_scene)
slam_init_pose <scene>
slam_init_pose <scene> <x> <y> <yaw>
slam_global_pose

# Marking
slam_mark <name>
slam_mark <name> --no-save

# Navigation
slam_nav_to_pose <x> <y> <yaw>
slam_nav_relative <x> <y> <yaw>
slam_pause_nav / slam_resume_nav / slam_cancel_nav
slam_nav_state
slam_shutdown                             # 1810, end localization and navigation
slam_cycle_test <x1> <y1> <yaw1> ...

# State/logs
slam_state
slam_status
slam_get_state
slam_restart_odom [scene] [relocalize]

# Visualization
bash scripts/foxglove.sh
bash scripts/launch_viz.sh
```

`request_id` defaults to `test_<interface>` and can be overridden with `SLAM_REQUEST_ID`; the functions are defined in `scripts/sdk_cli.bash`.
Full hand-written `ros2 topic pub` payloads: [SDK Call Examples](./SDK_EXAMPLES.md).

### 11.2 Thresholds and parameters

For commonly used parameters, their defaults, effects, and how changes take effect, see **[Key Parameters](./SDK_KEY_PARAMETERS.md)**.

### 11.3 States

`IDLE`、`MAPPING_STARTING`、`MAPPING_RUNNING`、`MAPPING_STOPPING`、`MAPPING_TERMINATING`、`NAV_STARTING`、`NAV_READY`、`NAV_RUNNING`、`NAV_PAUSING`、`NAV_PAUSED`、`NAV_STOPPING`.

---

## 12. First reactions in an emergency

**Safety first:**

1. In case of collision/personal risk: CPU normal → **switch the remote controller to manual/remote mode to take over**; CPU high → **do not drive it with the remote controller**; cancel navigation and reduce the load first, and keep people away.
2. Once the robot is safe, `slam_state` — what state is it actually in now?
3. `slam_status` / `grep '\[API' logs/slam_sdk.log | tail` — what are the terminal state and `message` of the last request?
4. `top` — are CPU/memory maxed out, causing the locomotion inference rate to drop and the robot to lose balance?
5. The newest directory under `logs/nav_fail/` — why did navigation not get there?
6. When a restart is needed: `bash scripts/kill_nav.sh` → `bash scripts/launch_sdk.sh` → 1804.
