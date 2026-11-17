# SLAM SDK Key Parameters

> This is the English version of [SDK关键参数速查.md](../SDK关键参数速查.md).
>
> **Intended audience:** on-site commissioning and integration personnel
> **Parameter directory:** `/opt/app/slam/config/params/`
> **Last updated:** 2026-11-16
>
> This document only lists the parameters you are most likely to care about on site. **Parameters that are not listed here (internal DLL/BBS solver parameters, Nav2 plugin parameters, etc.) should not be changed on your own**; if you really must adjust them, back up the original file first and record the behavior before and after the change.

---

## 0. First: how changes take effect

| Parameter file | How changes take effect |
|----------|----------|
| `slam_sdk.yaml` | **Restart the SDK**: `bash scripts/launch_sdk.sh shut` and then `bash scripts/launch_sdk.sh` again; or `systemctl restart slam_sdk.service` |
| `bbs3d_params.yaml` / `dll_params.yaml` / `pose_verifier_params.yaml` / `nav2_params.yaml` | Re-run 1804 (run 1810 to shut down first if necessary); or restart the SDK |
| `speed_override.yaml` → `rpp_presets/*.yaml` | Hot-update through the guide `/guide_tour/set_speed`; or restart the navigation stack (re-run 1804) |
| Map files (`map.pcd` / `map.pgm` / `map.yaml`) | Re-run 1804 with the corresponding scene; or use the map-change flow |
| Program files, binaries in the install directory | **Do not modify by hand**; use the provided deployment package when an upgrade is needed |

> `cp` a backup before changing anything; after changing, first confirm the behavior with `slam_state`, `slam_status`, and `logs/slam_sdk.log`.

---

## 1. Core localization and navigation thresholds (`slam_sdk.yaml`)

| Parameter | Default | Effect | When to tune | Risk |
|------|--------|------|------------|------|
| `pose_err_thr` | `0.20` m | Error threshold for waiting on localization convergence during 1804 startup | Slightly loosen when localization is always just short of ready and the floor/lidar noise on site is high | Loosening it makes localization worse at `NAV_READY` and navigation more prone to drift |
| `nav_reject_err_thr` | `0.4` m | Rejection threshold for a new navigation request (1806) | Slightly loosen when the localization error often fluctuates between 0.3~0.4 m and requests are frequently rejected | Too large and navigation starts with an obvious offset |
| `nav_wait_time` | `90` s | Maximum wait for navigation readiness during 1804 | Increase for large scenes / low-compute platforms | Too large and a localization failure takes a long time to report |
| `max_nav_time` | `240` s | Timeout for a single navigation run; cancelled automatically on timeout | Increase for long-distance tasks | Too large and a stuck navigation keeps running for a long time |
| `odom_wait_time` | `30` s | Odometry heartbeat grace period during the `NAV_STARTING` phase | Increase on platforms that start slowly | Heartbeat loss is detected more slowly |
| `odom_run_wait_time` | `1` s | Odometry heartbeat loss threshold during the running phase | Increase when the lidar/odometry rate is low | Too large and an odometry stream dropout is detected late |
| `odom_drift_thr` | `5.0` m/s | Odometry drift speed threshold during navigation | Usually leave it alone | Too small and 1816 triggers falsely; too large and real drift is missed |
| `odom_drift_confirm_n` | `5` | How many consecutive abnormal frames are needed to declare drift | Slightly increase when there are many occasional jumps on site | Too large and protection is delayed |
| `restart_cooldown` | `10` s | Minimum interval between automatic restarts | Usually leave it alone | Too small and you get a restart storm |
| `odom_restart_max_fail` | `3` | Consecutive failure limit of the odometry restart script | Usually leave it alone | Once reached, the SDK shuts down completely |
| `odom_restart_search_range` | `3.0` m | BBS search radius for the limited-range search after a restart; 0=disabled | Increase when the robot may be moved a lot during the restart | Larger is slower |
| `odom_restart_threads` | `1` | Number of BBS threads for the limited-range search after a restart; 0=do not modify | Keep it at 1 when CPU is tight | Increasing it speeds up the search but costs more CPU |
| `map_drift_speed_thr` | `5.0` m/s | Mapping drift speed threshold | Usually leave it alone | Too small and mapping is terminated falsely |
| `map_drift_confirm_n` | `3` | Number of consecutive abnormal frames for mapping | Usually leave it alone | Same as above |

> Built-in code defaults may differ from the yaml (for example `nav_reject_err_thr` defaults to 0.3 in code and is 0.4 in the yaml). **The actual value in `config/params/slam_sdk.yaml` of the deployment directory is authoritative.**

---

## 2. Navigation speed (`speed_override.yaml` → `rpp_presets/`)

The current `speed_override.yaml` points at `rpp_presets/medium.yaml`. The three levels only override 5 parameters of `FollowPath`; the rest are inherited from `pure_pursuit_params.yaml`.

| Level | File | `desired_linear_vel` | `lookahead_dist` | `approach_velocity_scaling_dist` |
|------|------|----------------------|------------------|----------------------------------|
| Low | `rpp_presets/low.yaml` | `0.4` m/s | `0.8` m | `0.8` m |
| Medium | `rpp_presets/medium.yaml` | `0.75` m/s | `1.5` m | `1.2` m |
| High | `rpp_presets/high.yaml` | `1.2` m/s | `2.4` m | `1.6` m |

- **On site, switch levels with the guide `/guide_tour/set_speed`**; no file editing needed.
- After changing the symlink or the yaml directly, the navigation stack must be restarted (re-run 1804) for the change to take effect.
- The launch acceleration limit is `max_linear_accel` in `pure_pursuit_params.yaml` (default `0.5` m/s²); increasing this value at will is not recommended.
- When there are many people, the floor is slippery, or the vehicle has a high center of gravity, prefer the low-speed level.

---

## 3. BBS global relocalization (`bbs3d_params.yaml` + `slam_sdk.yaml` adaptivity)

| Parameter | Default | Effect | Recommendation |
|------|--------|------|------|
| `bbs.min_level_res` | Computed automatically by the SDK from the map bounding box (`bbs_leaf_budget` / `bbs_min_level_res_range` in `slam_sdk.yaml`) | BBS leaf-level resolution | Generally do not hard-code it; large scenes get coarser automatically, small scenes finer |
| `bbs_leaf_budget` | `7000000` | Leaf-level node budget; larger means finer, slower, and more accurate | Increase when localization often fails and CPU has headroom; decrease when localization is very slow / CPU is tight |
| `bbs_min_level_res_range` | `[1.0, 8.0]` m | Lower and upper bounds of the automatic resolution | Small scenes can lower the lower bound, large scenes can raise the upper bound |
| `bbs.max_num_threads` | `2` | Number of BBS search threads | Keep it or reduce it when CPU is tight; do not exceed the number of physical cores |
| `bbs.eval_num_threads` | `4` | Number of threads for fine-scale rescoring | Usually leave it alone |
| `bbs.timeout` | `30000` ms | Timeout of a single global search | Increase for large scenes; too small and localization fails |

> BBS only consumes CPU in bursts during 1804 startup/relocalization; being busy for tens of seconds is normal. If it stays at full load for a long time, look at `bbs_leaf_budget` and the thread counts first, then consider using a smaller map extent.

---

## 4. Pose scoring (`pose_verifier_params.yaml`)

| Parameter | Default | Effect | Recommendation |
|------|--------|------|------|
| `thr_wall_alignment` | `0.2` | Threshold for how well the point cloud fits the walls | Usually leave it alone; only consider it when the map differs greatly from the site |
| `thr_range_consistency` | `0.2` | Range consistency threshold | Same as above |
| `thr_pierce` | `0.8` | Upper limit of the wall-piercing ratio | Same as above |
| `cloud_topic` | `/cloud_registered_body_filtered` | Point cloud used for scoring | Do not change it unless the point-cloud pipeline changes |
| `map_path` | Overridden by the launch arguments | 2D map used for scoring | Do not change by hand; it is passed automatically when the map is changed |

When scoring fails, the SDK refuses to enter `NAV_READY` or refuses navigation. **Prefer relocalizing / moving to a feature-rich position over loosening the thresholds just to "pass the score".**

---

## 5. DLL tracking localization (`dll_params.yaml`)

| Parameter | Default | Effect | Recommendation |
|------|--------|------|------|
| `pcd_downsample_res` | `0.05` m (overridden automatically by the SDK based on the point count at runtime) | Map downsampling resolution | Generally do not hard-code it |
| `fitness_threshold` | `0.5` | Localization score threshold; above it the current frame is untrustworthy | Do not loosen it at will, otherwise wrong poses will be accepted |
| `update_min_time` | `1.0` s | Minimum update interval | Usually leave it alone |
| `filter_type` | `queue_average` | Pose filtering method | When the jitter is large, `one_euro` can be evaluated, subject to actual testing |
| `filter_window` | `5` | Number of frames in the window average | Larger is steadier but lags more |
| `align_method` | `5` (DCReg) | Registration algorithm | Do not change it unless you clearly understand the impact |

---

## 6. Collision risk and on-site assistance

| Parameter/topic | Default | Effect |
|-----------|--------|------|
| `/slam/collision_risk` | 1 Hz, `Float32` 0~1 | Forward collision risk; the larger, the more dangerous; can be wired to upstream slowdown/stop |
| `risk_range` | `1.5` m | Forward detection length |
| `risk_half_width` | `0.4` m | Forward detection half-width |
| `risk_cost_thr` | `50` | Once the average cost reaches this value, the risk becomes 1 |
| `fail_dump_enable` | `true` | Render the scene into `logs/nav_fail/` when navigation fails |

Collision risk is only an auxiliary signal; **it cannot replace human observation and remote-controller takeover**.

---

## 7. Principles for adjusting parameters

1. **Change only one parameter at a time**, and record the observed behavior after each change, so that multiple variables do not get mixed together.
2. **Change "behavior" parameters first** (speed, timeouts, collision risk), then consider "algorithm" parameters (BBS/DLL/scoring).
3. **Do not loosen safety/scoring thresholds just to get one task through**; prefer changing the map, relocalizing, or reducing the speed.
4. **Do not move joints or send walking commands (including from the remote controller) when CPU is high**: SLAM and the reinforcement-learning locomotion controller share the CPU, and an overloaded CPU lowers the locomotion rate and makes it impossible to keep balance. Cancel navigation and close visualization to reduce the load first.
5. Back up before tuning on site:

```bash
cp -a config/params config/params.bak_$(date +%Y%m%d_%H%M%S)
```

6. After changing, restart the corresponding service and verify with `slam_status` / `logs/slam_sdk.log`; roll back from the backup if anything is abnormal.
