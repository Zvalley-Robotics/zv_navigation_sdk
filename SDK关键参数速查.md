# SLAM SDK 关键参数速查

> **适用对象：** 现场调试与集成人员
> **参数目录：** `/opt/app/slam/config/params/`
> **最后修改时间：** 2026-11-16
> **English:** [Key Parameters](./en/SDK_KEY_PARAMETERS.md)
>
> 本文只列出现场最可能需要关注的参数。**未列出的参数（DLL/BBS 求解器内部参数、Nav2 插件参数等）不建议自行修改**；确需调整时先备份原文件，并记录改动前后现象。

---

## 0. 先看：改完怎么生效

| 参数文件 | 生效方式 |
|----------|----------|
| `slam_sdk.yaml` | **重启 SDK**：`bash scripts/launch_sdk.sh shut` 后重新 `bash scripts/launch_sdk.sh`；或 `systemctl restart slam_sdk.service` |
| `bbs3d_params.yaml` / `dll_params.yaml` / `pose_verifier_params.yaml` / `nav2_params.yaml` | 重新执行 1804（必要时先 1810 关停）；或重启 SDK |
| `speed_override.yaml` → `rpp_presets/*.yaml` | 通过导览 `/guide_tour/set_speed` 热更新；或重启导航栈（重新 1804） |
| 地图文件（`map.pcd` / `map.pgm` / `map.yaml`） | 重新 1804 选对应场景；或换图流程 |
| 程序文件、安装目录中的二进制 | **不要手动修改**；需要升级时使用提供的部署包 |

> 修改前先 `cp` 备份；修改后先用 `slam_state`、`slam_status` 和 `logs/slam_sdk.log` 确认行为符合预期。

---

## 1. 定位与导航核心阈值（`slam_sdk.yaml`）

| 参数 | 默认值 | 作用 | 什么时候调 | 风险 |
|------|--------|------|------------|------|
| `pose_err_thr` | `0.20` m | 1804 启动阶段等待定位收敛的误差阈值 | 定位总是差一点就绪、现场地面/雷达噪声较大时可小幅放宽 | 放宽会让 `NAV_READY` 时定位更差，导航更容易偏 |
| `nav_reject_err_thr` | `0.4` m | 新导航请求（1806）的拒绝阈值 | 定位误差经常在 0.3~0.4 m 波动、频繁被拒绝时可小幅放宽 | 过大会带着明显偏差起步 |
| `nav_wait_time` | `90` s | 1804 等待导航就绪的最长时间 | 大场景/低算力平台可适当加大 | 太大时定位失败要等很久才报错 |
| `max_nav_time` | `240` s | 单次导航超时，超时自动取消 | 长距离任务可适当加大 | 太大会让卡住的导航持续很久 |
| `odom_wait_time` | `30` s | `NAV_STARTING` 阶段的里程计心跳宽限 | 启动慢的平台可加大 | 心跳丢失发现变慢 |
| `odom_run_wait_time` | `1` s | 运行阶段里程计心跳丢失阈值 | 雷达/里程计频率较低时可加大 | 太大会延迟发现里程计断流 |
| `odom_drift_thr` | `5.0` m/s | 导航里程计漂移速度阈值 | 一般不动 | 太小会误触发 1816，太大会漏掉真实漂移 |
| `odom_drift_confirm_n` | `5` | 连续多少帧异常才判定漂移 | 现场偶发跳变多时可适当加大 | 太大会延迟保护 |
| `restart_cooldown` | `10` s | 自动重启最小间隔 | 一般不动 | 太小会重启风暴 |
| `odom_restart_max_fail` | `3` | 里程计重启脚本连续失败上限 | 一般不动 | 达到后 SDK 会完整关停 |
| `odom_restart_search_range` | `3.0` m | 重启后限定范围 BBS 搜索半径；0=关闭 | 重启期间机器人可能被移动较多时可加大 | 越大越慢 |
| `odom_restart_threads` | `1` | 重启后限定范围搜索的 BBS 线程数；0=不修改 | CPU 紧张时保持 1 | 加大可加快搜索但更吃 CPU |
| `map_drift_speed_thr` | `5.0` m/s | 建图漂移速度阈值 | 一般不动 | 太小会误终止建图 |
| `map_drift_confirm_n` | `3` | 建图连续异常帧数 | 一般不动 | 同上 |

> 代码内建缺省与 yaml 可能不同（例如 `nav_reject_err_thr` 代码缺省 0.3、yaml 为 0.4）。**以部署目录中 `config/params/slam_sdk.yaml` 的实际值为准**。

---

## 2. 导航速度（`speed_override.yaml` → `rpp_presets/`）

当前 `speed_override.yaml` 指向 `rpp_presets/medium.yaml`。三个档位只覆盖 `FollowPath` 的 5 个参数，其余继承 `pure_pursuit_params.yaml`。

| 档位 | 文件 | `desired_linear_vel` | `lookahead_dist` | `approach_velocity_scaling_dist` |
|------|------|----------------------|------------------|----------------------------------|
| 低速 | `rpp_presets/low.yaml` | `0.4` m/s | `0.8` m | `0.8` m |
| 中速 | `rpp_presets/medium.yaml` | `0.75` m/s | `1.5` m | `1.2` m |
| 高速 | `rpp_presets/high.yaml` | `1.2` m/s | `2.4` m | `1.6` m |

- **推荐现场用导览 `/guide_tour/set_speed` 切档**，不需要改文件。
- 直接改软链或 yaml 后，需要重启导航栈（重新 1804）才生效。
- 起步加速度限制在 `pure_pursuit_params.yaml` 的 `max_linear_accel`（默认 `0.5` m/s²）；这个值不建议随意加大。
- 人多、地面滑、车体重心高时，优先用低速档。

---

## 3. BBS 全局重定位（`bbs3d_params.yaml` + `slam_sdk.yaml` 自适应）

| 参数 | 默认值 | 作用 | 建议 |
|------|--------|------|------|
| `bbs.min_level_res` | 由 SDK 按地图包围盒自动计算（`slam_sdk.yaml` 的 `bbs_leaf_budget`/`bbs_min_level_res_range`） | BBS 叶层分辨率 | 一般不要手工写死；大场景自动变粗，小场景自动变细 |
| `bbs_leaf_budget` | `7000000` | 叶层节点预算；越大越细、越慢、越准 | 定位经常失败且 CPU 有余量时可加大；定位很慢/CPU 紧张时调小 |
| `bbs_min_level_res_range` | `[1.0, 8.0]` m | 自动分辨率的上下限 | 小场景可把下限调小，大场景可把上限调大 |
| `bbs.max_num_threads` | `2` | BBS 搜索线程数 | CPU 紧张时保持或调小；不要超过物理核数 |
| `bbs.eval_num_threads` | `4` | 细尺度重打分线程数 | 一般不动 |
| `bbs.timeout` | `30000` ms | 单次全局搜索超时 | 大场景可适当加大；太小会定位失败 |

> BBS 只在 1804 启动/重定位时集中吃 CPU，几十秒内属于正常；如果长时间满载，先看 `bbs_leaf_budget` 和线程数，再考虑换更小的地图范围。

---

## 4. 位姿评分（`pose_verifier_params.yaml`）

| 参数 | 默认值 | 作用 | 建议 |
|------|--------|------|------|
| `thr_wall_alignment` | `0.2` | 点云与墙面贴合度阈值 | 一般不动；地图与现场差异大时才考虑 |
| `thr_range_consistency` | `0.2` | 量程一致性阈值 | 同上 |
| `thr_pierce` | `0.8` | 穿墙比例上限 | 同上 |
| `cloud_topic` | `/cloud_registered_body_filtered` | 评分用点云 | 不要改，除非点云链路变更 |
| `map_path` | 由启动参数覆盖 | 评分用 2D 地图 | 不要手工改，换图时会自动传 |

评分不通过时，SDK 会拒绝进入 `NAV_READY` 或拒绝导航。**优先重新定位/移动到特征明显处，不要为了“过评分”而放宽阈值。**

---

## 5. DLL 跟踪定位（`dll_params.yaml`）

| 参数 | 默认值 | 作用 | 建议 |
|------|--------|------|------|
| `pcd_downsample_res` | `0.05` m（运行时由 SDK 按点数自动覆盖） | 地图下采样分辨率 | 一般不要写死 |
| `fitness_threshold` | `0.5` | 定位分数阈值，超过则本帧不可信 | 不要随意放宽，否则错误位姿会被接受 |
| `update_min_time` | `1.0` s | 最小更新间隔 | 一般不动 |
| `filter_type` | `queue_average` | 位姿滤波方式 | 抖动大时可评估 `one_euro`，需实测 |
| `filter_window` | `5` | 窗口平均帧数 | 加大更稳但更滞后 |
| `align_method` | `5`（DCReg） | 配准算法 | 不要改，除非明确知道影响 |

---

## 6. 碰撞风险与现场辅助

| 参数/话题 | 默认值 | 作用 |
|-----------|--------|------|
| `/slam/collision_risk` | 1 Hz，`Float32` 0~1 | 前方碰撞风险，越大越危险；可接上游降速/停车 |
| `risk_range` | `1.5` m | 前方检测长度 |
| `risk_half_width` | `0.4` m | 前方检测半宽 |
| `risk_cost_thr` | `50` | 平均代价达到该值即风险=1 |
| `fail_dump_enable` | `true` | 导航失败时把现场渲染到 `logs/nav_fail/` |

碰撞风险只做辅助判断，**不能替代人工观察和遥控器接管**。

---

## 7. 参数调整原则

1. **一次只改一个参数**，改完记录现象，避免多个变量混在一起。
2. **先改“行为类”参数**（速度、超时、碰撞风险），再考虑“算法类”参数（BBS/DLL/评分）。
3. **不要为了让一次任务通过而放宽安全/评分阈值**；优先换地图、重新定位、降低速度。
4. **CPU 高时不要移动关节、不要下发行走命令（包括遥控器）**：SLAM 与强化学习运控共用 CPU，负载过高会让运控频率下降、无法保持平衡。先取消导航、关闭可视化降低负载。
5. 现场调参前先备份：

```bash
cp -a config/params config/params.bak_$(date +%Y%m%d_%H%M%S)
```

6. 改完重启对应服务，并用 `slam_status` / `logs/slam_sdk.log` 验证；异常时回滚备份。
