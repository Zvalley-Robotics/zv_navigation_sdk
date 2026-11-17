# SLAM SDK 全流程操作手册（建图 → 定位 → 打点 → 导航）

> **适用对象：** 机器人部署、集成与现场操作人员
> **适用范围：** 本仓库提供的 SLAM SDK（接口编码 1801~1822），部署目录 `/opt/app/slam`
> **配套文档：** [SDK 接口文档](./SDK接口文档.md) · [SDK 状态机](./SDK状态机.md) · [SDK 调用示例](./SDK调用示例.md) · [关键参数速查](./SDK关键参数速查.md)
> **最后修改时间：** 2026-11-16
> **English:** [Full SDK Workflow Guide](./en/SDK_FULL_WORKFLOW_GUIDE.md)

---

## 0. 使用前必读

### 0.1 安全第一

1. **机器人会真的动。** `1806 nav_to_pose`、`1808 resume_nav`，以及里程计自愈后的自动恢复，都会让底盘自主运动。操作前确认机器人周围无人、无障碍，且遥控器在手、可随时接管。
2. **不导航时，把底盘切到遥控模式，并确保遥控器在手。** 不要让机器人长期停留在“导航模式 + SDK 空闲”状态：误发一条 1806、上游重试，或 1816 自愈后的自动恢复，都可能让机器人突然起步。
3. **CPU 高时绝对不要动关节。** 机器人上 SLAM 算法与强化学习运控通常共用同一块计算单元；CPU 打满会让运控模型推理频率下降、控制周期拉长，机器人无法保持平衡，直接后果是摔倒。因此：
   - 起身、迈步、切导航模式前，先确认 CPU 有余量（见 7.3）；
   - **CPU 占用高时，不要移动关节，也不要用遥控器下发行走命令**；
   - 先取消导航、关闭可视化/调试工具降低负载，等 CPU 降下来、控制周期恢复后再动。
4. **紧急接管要看 CPU 状态。** 有碰撞/人员风险时：
   - CPU 正常：立即用遥控器切到手动/遥控模式抢控制权，这比在终端里发 1809 更快；
   - CPU 已经很高：**不要用遥控器驱使它行走**，优先发 1809 取消导航、关闭可视化降低负载，人员保持距离；等控制恢复后再处理。
   - 接管后 SDK 可能检测到里程计/定位异常，触发 1816 或返回 `Goal was cancelled`，这是预期行为；机器人安全停稳后，再按需发 1809 清理导航状态、发 1804 重新定位，确认 `NAV_READY` 后再继续。
5. **建图、定位期间保持机器人静止启动。** 1801 后要等 `/cloud_registered` 出现（15 s），1804 后要等定位收敛（默认最多 90 s）；这段时间搬动机器人会导致启动失败或定位错误。
6. **不要重复启动 SDK。** 同一台机器只应有一套 SDK 节点在运行；切换/重启使用提供的启动脚本或服务命令，不要手工重复拉起。

### 0.2 四个关键认知

| 认知 | 说明 |
|------|------|
| **SDK 只有 topic + JSON** | 请求发到 `/slam/xxx`（`std_msgs/String`，`data` 是 JSON 字符串），响应统一从 `/slam/status` 回来。没有 REST/SDK 库。 |
| **`success=0` 不是终态** | `0` = 请求已接收；必须按 `request_id` 等 `success=1`（成功）或 `2`（失败）。 |
| **状态机决定能不能发** | 建图只在 `IDLE`；导航只在 `NAV_READY`；`MAPPING_*` 与 `NAV_*` 互斥。先查 `slam_state` 再操作。 |
| **地图是契约，参数改动要重启** | 场景目录必须有 `map.pcd`、`map.yaml`、`map.pgm`；`data/maps/out` 指向当前场景。改了参数后重启 SDK（或对应服务）才生效。 |

> 全部状态、迁移和自动保护见 [SDK 状态机](./SDK状态机.md)；需要调整阈值时先看 [关键参数速查](./SDK关键参数速查.md)。

### 0.3 一页纸最短流程（照抄即可）

```bash
source /opt/app/slam/.bashrc          # 每个新终端执行一次
bash scripts/launch_sdk.sh            # 启动雷达 + SDK（部署机可用 systemctl restart slam_sdk.service）
slam_state                            # 期望 IDLE

slam_start_mapping site_a             # 1801，等 success=1 后推着机器人走场
slam_stop_mapping site_a              # 1802，等 success=1

slam_init_pose site_a                 # 1804，等状态变 NAV_READY
slam_mark entrance                    # 打点（可选）
slam_nav_to_pose 5.0 3.0 1.57         # 1806
slam_shutdown                         # 1810
```

状态怎么走：

```
IDLE ──1801──> MAPPING_* ──1802──> IDLE
  │
  └──1804──> NAV_READY ──1806──> NAV_RUNNING ──(到达/失败)──> NAV_READY
                                    └──1807 / 1808 / 1809──> NAV_PAUSED / NAV_READY
```

> 每一步的判据、失败原因、走场规范见第 2~5 章；命令全表见 11.1；
> 需要手写 `ros2 topic pub`（字段级 JSON）时看 [SDK 调用示例](./SDK调用示例.md)。

---

## 1. 环境与启动

### 1.1 登录与环境

部署机上工程位于 `/opt/app/slam`。登录后执行一次：

```bash
source /opt/app/slam/.bashrc
```

它会设置 `SLAM_ROOT`、ROS 环境变量，并加载 `slam_*` 快捷命令。

> 每个新开的终端都需要执行一次 `source`。

### 1.2 自检清单

```bash
echo "$SLAM_ROOT"                 # 应为 /opt/app/slam
ros2 pkg prefix slam_control_c    # 能定位到安装目录
nproc                             # CPU 核数（后面判断负载用）
free -h                           # 内存
df -h "$SLAM_ROOT/data/maps"      # 地图盘空间
ls -l "$SLAM_ROOT/data/maps/out"  # 当前场景软链
```

### 1.3 启动 SDK

部署机通常已配置开机自启，先检查：

```bash
systemctl status slam_sdk.service
```

需要手动启动或重启时：

```bash
# 脚本方式
bash /opt/app/slam/scripts/launch_sdk.sh

# 或服务方式（部署机）
systemctl restart slam_sdk.service
```

启动内容：雷达驱动 + SDK 节点。日志在 `logs/livox.log`、`logs/slam_sdk.log`。

约 10~30 s 后确认：

```bash
slam_state          # 等价于 ros2 topic echo /slam/_state --once
# 期望输出：IDLE
```

> **还没定位，不能导航。** 必须执行 1804 进入 `NAV_READY`。

### 1.4 开三个终端（推荐现场工作方式）

| 终端 | 用途 | 命令 |
|------|------|------|
| A | 启动/重启（一次性） | `bash scripts/launch_sdk.sh` 或 `systemctl restart slam_sdk.service` |
| B | **持续看接口响应** | `slam_status`（或 `ros2 topic echo /slam/status --full-length`） |
| C | 发指令 / 看状态 / 看 CPU | `slam_state`、`slam_global_pose`、`top` |

> 所有请求和响应也会写入 `logs/slam_sdk.log`，现场来不及盯终端时可直接事后 grep（见第 7 节）。

---

## 2. 建图（1801 → 1802）

### 2.1 标准流程

```bash
# 1) 确认空闲
slam_state                      # 必须是 IDLE

# 2) 开始建图，scene_id 自定义（建议英文+数字，如 site_a_20261116）
slam_start_mapping site_a

# 3) 等 1801 终态
#    /slam/status 出现 task_id=1801, success=1, message="mapping started"
slam_state                      # 期望 MAPPING_RUNNING

# 4) 推着机器人走完场地（或遥控低速走）

# 5) 结束建图并保存
slam_stop_mapping site_a
#    期望 task_id=1802, success=1, message="Map saved"
slam_state                      # 回到 IDLE
```

### 2.2 走场规范（直接决定定位成功率）

1. **速度 ≤ 0.5 m/s**，转弯更慢；不要猛停猛转、不要原地快速旋转。
2. **闭环**：最后回到起点附近，让点云重访，检查地图是否“重影”。
3. **覆盖完整**：走廊尽头、房间门口都要走到；不要只走主通道。
4. **避免退化场景**：长直走廊、大片玻璃/白墙、空旷大厅容易导致建图漂移；尽量让雷达看到墙角、门框、家具等特征。
5. **不要上下楼梯**（2D 栅格地图不适用）；不要在有大量行人时建图。
6. **建图中保持网络/电源稳定**；雷达被遮挡或断流会直接导致建图失败。

### 2.3 建图期间的监控

```bash
# 状态
slam_state
# 点云/里程计是否持续
ros2 topic hz /cloud_registered
ros2 topic hz /aft_mapped_to_init
# CPU（第 7 节有详细说明）
top -b -n1 -o %CPU | head -15
```

### 2.4 结束建图的产物

```
data/maps/<scene_id>/
├── map.pcd      # 3D 点云地图：定位（BBS/DLL）使用
├── map.pgm      # 2D 栅格图：定位校验、导航底图
└── map.yaml     # 分辨率/原点等元数据
data/maps/out -> <scene_id>     # 软链：指向最近一次成功保存的场景
```

可选文件（平台后处理，不是 SDK 必需）：

- `map_nav.yaml` / `map_nav.pgm`：导航专用 P 图；存在时导航用它，**定位/定位校验仍用原图**。
- `keepout.yaml` / `keepout.pgm`：禁行区；需 `use_keepout_zone:=True` 才启用。

检查：

```bash
ls -l data/maps/out
ls -l data/maps/$(basename "$(readlink -f data/maps/out)")
slam_latest_scene               # 输出最近一次保存的 scene_id
```

### 2.5 建图失败/中止

| 情况 | 表现 | 处理 |
|------|------|------|
| 启动失败 | 1801 `success=2 "mapping start failed"` | 看 `logs/livox.log`、`logs/odom.log`；确认雷达在线、`/cloud_registered` 能出现；清理后重试 |
| 建图中止 | 语音“里程计漂移”，状态自动回 `IDLE`，1802 不再可用 | 本次地图不可信，重新 1801；检查是否推得太快/打滑/雷达被挡 |
| 主动放弃 | 发 1822 | `slam_terminate_mapping site_a`，**不保存地图**，状态回 `IDLE` |
| 保存失败 | 1802 `success=2 "Missing map files"` / `"Copy map error"` | 看 `logs/slam_sdk.log`；检查 `data/maps/<scene>` 权限和磁盘空间；`out` 会指向 `_ERROR` |

---

## 3. 定位（1804）

> 定位 = 拉起 BBS 全局重定位 + DLL 跟踪 + Nav2 + pose_verifier。成功后进入 `NAV_READY`，才能导航。

### 3.1 自动估计位姿（推荐）

```bash
slam_state                                  # 必须是 IDLE（或 NAV_STARTING/READY/PAUSED 下更新位姿）
slam_init_pose site_a                       # auto_init_pose=true
# 等 /slam/status: task_id=1804, success=1, message="Ready for navigation"
slam_state                                  # 期望 NAV_READY
```

### 3.2 手动指定位姿（已知起点时）

```bash
slam_init_pose site_a 1.5 2.0 1.57
# 参数含义：scene_id x y yaw(弧度)
```

手动模式适合：

- 机器人每次从固定充电位出发；
- 自动重定位失败（走廊/空旷/相似场景），但你知道机器人在哪。

### 3.3 等待与判据

- 等待上限 `nav_wait_time`（默认 90 s）。
- 定位误差 `pose_error` 收敛到 `pose_err_thr`（默认 0.20 m）以内。
- 再等 3 s 让误差稳定，调用 `/slam/get_pose_score` 做位姿评分；通过后进入 `NAV_READY`。

### 3.4 失败原因与处理

| 返回 message | 含义 | 处理 |
|--------------|------|------|
| `Scene id <id> not exist or missing files: [map.pcd, map.yaml, map.pgm]` | 场景不存在或缺文件 | 检查 `data/maps/<id>/` 三件套；先用 1802 正常保存一次 |
| `Request mismatch with current scene id: ...` | 导航绑定场景后不能换图 | 先 1810 关停，再用新 `scene_id` 发 1804 |
| `Leftover nav processes detected, clean and retry: ...` | 上次导航进程没清干净 | 再执行一次 1804（内部会先清理），或 `bash scripts/kill_nav.sh` |
| `Timeout to start navigation, please restart` | 90 s 内定位误差未收敛 | 看 `logs/dll.log`、`logs/nav2.log`；确认雷达点云正常、CPU 不过载；重试或改手动位姿 |
| `Low pose score, please relocalize` | 位姿评分不达标（定位可能偏了） | 把机器人移到特征明显的位置，重新 `slam_init_pose`；或改手动位姿 |
| `Pose score service unavailable, please restart` | pose_verifier 服务不可用 | 看 `logs/nav2.log` 中 pose_verifier 是否启动；重启 SDK |
| `Navigation shutdown during odom restart` | 里程计自愈期间导航被关停 | 等自愈结束后重新 1804 |

### 3.5 定位后自检

```bash
slam_global_pose
# 响应 data 里看 pose_error（应 < 0.3）和 global_pose

slam_nav_state                  # 此时不在导航中，会返回失败，属正常
```

在 Foxglove 里看 `/cloud_registered` 与 `/map` 是否重合（见第 8 节），是最直观的定位正确性检查。

---

## 4. 打点（记录点位）

SDK 本身没有“打点”接口；打点 = **读取当前位姿并记录下来**。有两种方式：

### 4.1 方式一：`slam_mark`（推荐，纯 SDK）

```bash
slam_mark entrance              # 记录到 logs/sdk_marks.csv
```

输出示例：

```
entrance,1.2345,2.3456,0.7854,0.1200
→ 导航命令: slam_nav_to_pose 1.2345 2.3456 0.7854
已追加到 /opt/app/slam/logs/sdk_marks.csv
```

- 需要状态为 `NAV_READY` / `NAV_RUNNING` / `NAV_PAUSED`（1805 的可用状态）。
- 默认追加到 `logs/sdk_marks.csv`（列：`name,x,y,yaw,pose_error`）。
- `pose_error > 0.3 m` 会打印告警：点位不可靠，先重新定位。
- 只打印不落盘：`slam_mark p1 --no-save`。
- 指定文件：`slam_mark p1 --file /tmp/p1.csv`。

### 4.2 方式二：导览服务 `/guide_tour/mark`（需要导览节点）

```bash
bash scripts/launch_guide_services.sh
ros2 service call /guide_tour/mark reception_demo/srv/GuideTourMark \
  "{id: 'p1', annotations: 'entrance', is_build_map: false}"
```

点位写入 `logs/marks.yaml`：

```yaml
waypoints:
  p1: {x: 1.2345, y: 2.3456, a: 0.7854}
```

> 注意：导览节点**每次启动会清空** `logs/marks.yaml`，需要长期保存请自行备份；建图期间也可以打点（`is_build_map: true`），SDK 的 1805 则只在 `NAV_*` 状态下可用。

### 4.3 打点规范

1. **先定位到 `NAV_READY`，确认 `pose_error` 小再打点。**
2. 点位放在通道中间、离墙 ≥ 0.5 m，避免紧贴门口/玻璃。
3. yaw 是机器人**面向方向**（弧度，逆时针为正）；导航到点会同时对齐该朝向。
4. 命名用英文/数字/下划线，如 `lobby_01`、`room_a_door`。
5. 打完点先抽查 2~3 个点，用 `slam_nav_to_pose` 实跑，再批量采集。

### 4.4 批量验证点位

```bash
# 按顺序逐点导航，每个点到达后停留 5 s；异常立即停止
slam_cycle_test 1.2 2.3 0.78 4.5 2.3 0.0 -1.0 3.0 1.57
```

- `-d 10`：每点停留 10 s；`-n 3`：整组跑 3 轮；`-n 0`：无限循环，Ctrl-C 中断；`-t 120`：单点最长等待 120 s（默认 300）。
- 上一个点收到成功回复后才发下一个点；Ctrl-C/超时会自动发 1809 取消，避免机器人继续跑。
- 逻辑见 `scripts/slam_cycle_test.py`；退出码 `0` 全部到达、`1` 异常、`2` 参数错误、`130` 被中断。

---

## 5. 导航（1806 + 1807/1808/1809/1814）

### 5.1 绝对坐标导航

```bash
slam_state                      # 必须 NAV_READY
slam_nav_to_pose 5.0 3.0 1.57   # x(m) y(m) yaw(rad)
```

- 立即返回 `1806 success=0 "Success to send nav request"`。
- 到达后返回 `1806 success=1 "Success to reach the goal"`。
- 失败返回 `success=2`，见 5.4。

### 5.2 相对当前位姿导航

```bash
slam_nav_relative 2.0 0 0       # 沿当前朝向前进 2 m，保持朝向
```

`relative=true` 时 SDK 会用 `map -> base_link` 的 TF 把相对目标转成绝对坐标再下发，因此 TF 必须正常。

### 5.3 运行中的控制

| 操作 | 命令 | 说明 |
|------|------|------|
| 查进度 | `slam_nav_state` | 返回目标位姿、已导航时间、剩余距离 |
| 暂停 | `slam_pause_nav` | 取消当前目标但保留，状态 `NAV_PAUSED` |
| 恢复 | `slam_resume_nav` | 以原目标重新下发，状态回 `NAV_RUNNING` |
| 取消 | `slam_cancel_nav` | 回 `NAV_READY`，清空目标 |
| 换目标 | 先 `slam_cancel_nav`，再 `slam_nav_to_pose ...` | `NAV_RUNNING` 下不能直接改目标 |

> 暂停后**不要发新目标**：`NAV_PAUSED` 下只有“恢复”能进导航，其它 1806 会被拒绝（`Navigation paused, please resume before request`）。

### 5.4 导航失败原因

| message | 含义 | 处理 |
|---------|------|------|
| `Navigation not started, invalid request` | 未定位（`IDLE`） | 先 1804 |
| `Navigation is starting, please wait for a moment` | 定位/导航栈还在启动 | 等 `NAV_READY` |
| `Navigation running, please cancel current goal before new request` | 正在导航 | 先 1809 |
| `Navigation is pausing, please wait` | 暂停过渡态 | 等 `NAV_PAUSED` |
| `Navigation paused, please resume before request` | 暂停态发新目标 | 用 1808 恢复，或先 1809 |
| `Localization error too high (x.xxx), please re-localize` | 定位误差 > `nav_reject_err_thr`（默认 0.4 m） | 重新 1804 或检查定位 |
| `Goal was rejected` | Nav2 拒绝目标（目标在障碍/未知区外等） | 检查目标点是否可达；看 `logs/nav2.log` |
| `Failed to connect nav server` | Nav2 action server 不在 | 看 `logs/nav2.log`、是否被 kill；重启导航 |
| `Failed to lookup relative transform` | 相对导航时 TF 查询失败 | 检查 `/slam/tf`、`map -> base_link` |
| `Goal was cancelled` / `Failed to reach the goal` | 被取消 / 中止 | 看 `logs/nav_fail/` 现场图 |
| `Failed to figure out result` | 结果解析异常 | 重试；持续出现看 `logs/slam_sdk.log` |

### 5.5 自动保护（不用你操作）

- **导航超时**：单次 > `max_nav_time`（默认 240 s）自动取消，1806 返回 `Goal was cancelled`。
- **里程计心跳丢失/漂移**：自动执行 1816（重启里程计 + 重新定位），成功后会恢复原目标；失败则完整关停。
- **建图漂移**：建图中速度连续超阈值（默认 5 m/s × 3 帧）自动 1822 终止建图。
- **碰撞风险**：`/slam/collision_risk`（0~1，1 Hz）可用于上游降速/停车。

### 5.6 结束导航

```bash
slam_shutdown                   # 1810：关雷达、定位、导航，回 IDLE
```

之后 `scene_id` 解绑；可以重新 1801/1804。**结束任务后把底盘切回遥控模式。**

---

## 6. 典型完整流程（一次任务）

```bash
source /opt/app/slam/.bashrc
bash scripts/launch_sdk.sh                  # 或 systemctl restart slam_sdk.service
slam_state                                  # IDLE

# ---- 建图（只需一次，地图可复用）----
slam_start_mapping arena
# 推场/遥控走场，速度 ≤0.5 m/s，闭环
slam_stop_mapping arena
ls -l data/maps/out                         # out -> arena

# ---- 定位 ----
# 把机器人放到地图对应位置，静止
slam_init_pose arena
slam_state                                  # NAV_READY

# ---- 打点 ----
slam_mark start_point
slam_mark goal_point
cat logs/sdk_marks.csv

# ---- 导航 ----
slam_nav_to_pose <goal_x> <goal_y> <goal_yaw>
slam_nav_state                              # 看剩余距离
# 到达后 /slam/status 出现 success=1

# ---- 收尾 ----
slam_shutdown
# 底盘切回遥控模式，遥控器在手
```

---

## 7. 监控、日志与 CPU

### 7.1 看接口响应：`/slam/status`

**推荐常开一个终端：**

```bash
slam_status
# 等价于
ros2 topic echo /slam/status --full-length
```

- `--full-length` 防止长字符串被截断（JSON 里 `data` 可能很长）。
- 每条响应都包含 `request_id`、`task_id`、`success`、`message`。
- **只看 `success` 为 1/2 的终态**；`success=0` 只是受理。

**响应也已写入日志**，适合事后复核：

```bash
tail -f logs/slam_sdk.log | grep --line-buffered '\[API'
# 或
grep '\[API' logs/slam_sdk.log | tail -50
```

日志中一条请求/响应大致长这样：

```
[API 1806 nav_to_pose] request_id=test_nav_to_pose scene_id=arena data={...}
[API 1806 nav_to_pose] response: {"request_id":"test_nav_to_pose",...,"success":0,...}
```

### 7.2 看状态机

```bash
slam_state                                  # 当前状态（10 Hz 广播，取一次）
ros2 topic echo /slam/_state                # 持续观察
```

状态含义见 [SDK 状态机](./SDK状态机.md)。

### 7.3 看 CPU 占用（重点）

**机器人上 SLAM 算法与强化学习运控通常部署在同一块计算单元上。CPU 打满时，运控模型的推理频率会下降、控制周期被拉长，机器人无法保持平衡，直接后果就是摔倒。** 因此 CPU 不是单纯的“性能指标”，而是安全前提：**CPU 高时不要移动关节，也不要用遥控器下发行走命令**，先降低负载。

```bash
# 整体
top
# 一次性快照（按 CPU 排序）
top -b -n1 -o %CPU | head -20
# 负载（load average 持续 > 核数 就是过载）
uptime
nproc
# Orin 平台
tegrastats
```

按阶段判断“谁在吃 CPU”：

| 阶段 | 主要 CPU 消耗者 | 说明 |
|------|------------------|------|
| 建图 | `fastlivo_mapping_t4` / `laserMapping` | 建图本身很重；若持续占满单核，降低走场速度 |
| 定位启动 | `bbs3d_register_node` | 全局搜索最吃 CPU（多线程），几十秒内属于正常 |
| 导航 | `controller_server` / `planner_server` / `dll_node` / `bt_navigator` | 规划频繁或代价地图大时明显 |
| 运控（RL） | 策略推理/关节控制进程（名字以实际部署为准） | 与 SLAM 共用 CPU；推理频率下降会直接导致失衡、摔倒 |
| 可视化 | `viz_node_c`（WebUI）、`foxglove_bridge` | 调试完可关掉，释放 CPU |

经验阈值：

- 总 CPU 持续 > 80%，或 load average > `nproc`：**不要起身/迈步/下发行走命令（包括遥控器）**；先取消导航、关闭可视化，等 CPU 降下来且控制周期恢复后再动。
- 如果机器人已经在走，优先发 1809 取消导航让它停下；**不要在 CPU 高时用遥控器继续驱使它行走**。
- 定位启动超过 30 s 还没 `NAV_READY`，先看 `top` 是不是 BBS 在跑，再看 `logs/dll.log` / `logs/nav2.log`。
- 需要降 BBS 开销：调小 `config/params/slam_sdk.yaml` 的 `bbs_leaf_budget`（更粗/更快），或临时把 `bbs.max_num_threads` 调小；改完重启 SDK（或重新 1804）。导航速度档位见 `config/params/speed_override.yaml`（当前指向 `medium`）。

> 可调参数的含义、默认值和生效方式见 [关键参数速查](./SDK关键参数速查.md)。

### 7.4 日志地图

| 文件/目录 | 内容 | 什么时候看 |
|-----------|------|------------|
| `logs/slam_sdk.log` | SDK 请求/响应、状态迁移、错误 | **第一优先** |
| `logs/livox.log` | 雷达驱动 | 点云没有/雷达掉线 |
| `logs/odom.log` | 建图/里程计进程输出 | 建图失败、里程计异常 |
| `logs/nav2.log` | Nav2 全部节点 + pose_verifier | 导航启动失败、规划/控制异常 |
| `logs/dll.log` | DLL 跟踪定位 | 定位误差大、`pose_error` 不收敛 |
| `logs/guide_tour.log` | 导览节点（如启用） | `/guide_tour/*` 服务问题 |
| `logs/nav_fail/` | 导航失败现场 PNG + `meta.json` | 导航“没走到”，直接看图 |
| `logs/sdk_marks.csv` | `slam_mark` 记录的点位 | 打点 |
| `logs/marks.yaml` | `/guide_tour/mark` 记录的点位 | 导览打点 |

常用：

```bash
tail -f logs/slam_sdk.log
tail -f logs/nav2.log
ls -lt logs/nav_fail | head
grep -E "Timeout|Low pose score|Leftover|drift|Missing odom" logs/slam_sdk.log
```

导航失败现场默认开启（`config/params/slam_sdk.yaml` 的 `fail_dump_enable: true`），每个失败目录里通常有 `annotated.png` 和 `meta.json`。

> `logs/slam_sdk.log` 每次启动都是**追加**；长时间运行或高频轮询 1805 时文件会持续变大，任务前建议清理或备份归档。

### 7.5 其它有用的观测

```bash
ros2 topic hz /cloud_registered          # 雷达点云
ros2 topic hz /aft_mapped_to_init       # 里程计
ros2 topic hz /slam/_state              # 10 Hz
ros2 topic echo /slam/collision_risk    # 碰撞风险 0~1
ros2 node list                          # 节点是否齐全
```

---

## 8. Foxglove 调试（强烈建议）

### 8.1 安装与连接

1. 在操作电脑安装 **Foxglove Studio**（`https://foxglove.dev`，桌面版即可）。
2. 机器人与操作电脑同一网络。
3. 机器人端启动 bridge：

```bash
bash scripts/foxglove.sh
# 端口 8776，日志 logs/foxglove.log
```

4. Foxglove 选择 **Open connection → Foxglove WebSocket**，地址：

```
ws://<机器人IP>:8776
```

5. 连不上时：
   - `ss -lntp | grep 8776` 确认端口在听；
   - 看 `logs/foxglove.log`；
   - 确认防火墙/网络 ACL 放行 8776。

### 8.2 推荐面板布局

| 面板 | 话题 | 用途 |
|------|------|------|
| 3D | `/cloud_registered`、`/map`、`/global_costmap/costmap`、`/local_costmap/costmap`、`/plan` | 看点云与地图是否重合、代价地图是否堵、路径是否合理 |
| 3D / TF | `/slam/tf`、`/slam/tf_static` | 检查 `map -> odom -> base_link` 是否连续 |
| Raw Messages | `/slam/status`、`/slam/_state` | 逐条看响应和状态 |
| Plot | `/slam/collision_risk` | 碰撞风险趋势 |
| Raw Messages | `/dll_node/...` | DLL 定位内部量（如 fitness） |

> **TF 注意：** 本栈做了 TF 隔离，TF 发在 `/slam/tf`、`/slam/tf_static`，不是 `/tf`。在 Foxglove 的 3D/TF 设置里把 TF 话题指到 `/slam/tf`；如果你的版本不支持自定义 TF 话题，用 Raw Messages 看 `/slam/tf`，或直接用 RViz。

### 8.3 能发布什么？

`scripts/foxglove.sh` 的 `client_topic_whitelist` 默认只允许客户端发布：

- `/initialpose`
- `/slam/cancel_nav`

因此 **Foxglove 主要用于观察，不用来发导航目标**。需要手动发目标请用 `slam_nav_to_pose`；如果确实要从 Foxglove 发，编辑 `scripts/foxglove.sh` 的 `CLIENT_TOPIC_WHITELIST` 加上 `/slam/nav_to_pose` 等，再重启 bridge（注意安全）。

### 8.4 录包复盘

```bash
ros2 bag record -o logs/bag_$(date +%H%M%S) \
  /slam/status /slam/_state /slam/collision_risk \
  /cloud_registered /map /plan \
  /global_costmap/costmap /local_costmap/costmap \
  /slam/tf /slam/tf_static
```

---

## 9. 安全与现场规范

### 9.1 模式与接管

- **不导航时：底盘切遥控模式，遥控器在手、可随时接管。** 原因见 0.1。
- **导航前：** 确认机器人处于导航模式，且没有被遥控器/上层程序同时下发速度；**确认 CPU 有余量**，CPU 高时不要起身/迈步/切导航模式。
- **导航中：** 人员与机器人保持 ≥ 1 m；狭窄通道安排人引导；持续关注 CPU。
- **紧急接管：** 出现碰撞风险、人员闯入、机器人异常加速/画龙时：
  - CPU 正常：**直接用遥控器切手动/遥控模式抢控制权**，不要先花时间在终端发取消命令；
  - CPU 已经很高：**不要用遥控器下发行走命令**，先发 1809 取消导航、关闭可视化降低负载，人员保持距离，等控制恢复后再处理。
- **接管之后：** 机器人安全停稳后，再发 `slam_cancel_nav` 清理导航状态；如果定位被扰动，发 1804 重新定位，确认 `NAV_READY` 再继续任务。

### 9.2 任务前检查清单

- [ ] 雷达固定牢靠、视野无遮挡，网线/供电正常。
- [ ] 地图是当前场地的最新版本（`ls -l data/maps/out`）。
- [ ] 定位成功，`pose_error` 稳定 < 0.3 m。
- [ ] 抽测 2~3 个导航点，均能到达。
- [ ] CPU/内存/磁盘余量正常，无残留进程（`bash scripts/kill_nav.sh` 后再启动）。
- [ ] **CPU 负载未持续 > 80%、load average 未超过核数**；CPU 高时不要起身/迈步/下发行走命令。
- [ ] 遥控器在手，操作人员明确接管流程。
- [ ] Foxglove 或 `slam_status` 已连上，能实时看到响应。
- [ ] 语音提示可闻（里程计丢失/漂移/导航超时/请让一让）。

### 9.3 运行中检查清单

- [ ] 每次任务前 `slam_state` 是 `NAV_READY`（不是 `IDLE`/`NAV_RUNNING`）。
- [ ] 每条 1806 都有终态：`success=1` 到达，或 `success=2` 有明确原因。
- [ ] CPU 没有持续打满；`/cloud_registered`、`/aft_mapped_to_init` 频率正常。
- [ ] **CPU 高时不要移动关节、不要用遥控器下发行走命令**；先取消导航、降低负载。
- [ ] 机器人异常静止/原地打转时，先看 `/slam/status` 和 `logs/nav_fail/`；有风险时按 9.1 处理（CPU 高时不要用遥控器驱使它行走）。

---

## 10. 常见问题（FAQ）

| 现象 | 可能原因 | 处理 |
|------|----------|------|
| `slam_state` 一直 `IDLE`，1804 失败 | 场景缺文件 / 残留进程 / 雷达未起 | 查 `data/maps/<scene>` 三件套；`bash scripts/kill_nav.sh`；看 `logs/livox.log` |
| 1804 成功但一直不 `NAV_READY` | 定位误差不收敛 / 评分不过 | 看 `logs/dll.log`、`pose_error`；移动到特征明显处重定位；手动给位姿 |
| 导航“没走到” | 定位偏了 / 代价地图堵 / 目标不可达 | 看 `logs/nav_fail/` 的 `annotated.png`；换目标；重新定位 |
| 导航中途停下并语音“里程计丢失/漂移” | 里程计断流/打滑/被抢控制 | 等 1816 自愈；必要时 `slam_cancel_nav` 后 1804；检查雷达/轮速 |
| 1816 后报 `Low pose score after odom restart` | 重启期间机器人被移动 | 手动 1804 重新定位；之后任务前确认 `NAV_READY` |
| `/slam/status` 收不到 | SDK 没起 / DDS 不通 / QoS | `slam_state` 是否正常；`ros2 topic list`；检查 `RMW_IMPLEMENTATION`、`ROS_DOMAIN_ID` |
| Foxglove 连不上 | 端口/防火墙/未启动 | `bash scripts/foxglove.sh`；`ss -lntp \| grep 8776`；看 `logs/foxglove.log` |
| 改了参数没效果 | 没重启对应服务 | 参数在 `config/params/` 改完后重启 SDK（或重新 1804）；见[关键参数速查](./SDK关键参数速查.md) |
| 机器人一启动就自己动 | 上一次目标/自动恢复未清 / 底盘仍在导航模式 | 先 1809/1810；把底盘切遥控；检查是否有上游在重发 1806 |

---

## 11. 速查表

### 11.1 命令

```bash
# 环境
source /opt/app/slam/.bashrc

# 启动/停止
bash scripts/launch_sdk.sh
bash scripts/launch_sdk.sh shut
systemctl restart slam_sdk.service        # 部署机
bash scripts/launch_guide_services.sh
bash scripts/kill_nav.sh

# 建图
slam_start_mapping <scene>
slam_stop_mapping <scene>
slam_terminate_mapping <scene>

# 定位
slam_latest_scene                         # 最近一次建图保存的场景 ID：slam_init_pose $(slam_latest_scene)
slam_init_pose <scene>
slam_init_pose <scene> <x> <y> <yaw>
slam_global_pose

# 打点
slam_mark <name>
slam_mark <name> --no-save

# 导航
slam_nav_to_pose <x> <y> <yaw>
slam_nav_relative <x> <y> <yaw>
slam_pause_nav / slam_resume_nav / slam_cancel_nav
slam_nav_state
slam_shutdown                             # 1810，结束定位与导航
slam_cycle_test <x1> <y1> <yaw1> ...

# 状态/日志
slam_state
slam_status
slam_get_state
slam_restart_odom [scene] [relocalize]

# 可视化
bash scripts/foxglove.sh
bash scripts/launch_viz.sh
```

`request_id` 默认是 `test_<接口名>`，可用 `SLAM_REQUEST_ID` 覆盖；函数定义见 `scripts/sdk_cli.bash`。
手写 `ros2 topic pub` 的完整报文见 [SDK 调用示例](./SDK调用示例.md)。

### 11.2 阈值与参数

常用参数、默认值、作用和生效方式见 **[关键参数速查](./SDK关键参数速查.md)**。

### 11.3 状态

`IDLE`、`MAPPING_STARTING`、`MAPPING_RUNNING`、`MAPPING_STOPPING`、`MAPPING_TERMINATING`、`NAV_STARTING`、`NAV_READY`、`NAV_RUNNING`、`NAV_PAUSING`、`NAV_PAUSED`、`NAV_STOPPING`。

---

## 12. 紧急与异常时的第一反应

**安全优先：**

1. 有碰撞/人员风险：CPU 正常 → **用遥控器切手动/遥控模式抢控制权**；CPU 高 → **不要用遥控器驱使它行走**，先取消导航、降低负载，人员保持距离。
2. 机器人安全后，`slam_state` —— 现在到底在什么状态？
3. `slam_status` / `grep '\[API' logs/slam_sdk.log | tail` —— 最后一条请求的终态和 `message` 是什么？
4. `top` —— CPU/内存是否打满，导致运控频率下降、无法保持平衡？
5. `logs/nav_fail/` 最新目录 —— 导航为什么没走到？
6. 需要重启时：`bash scripts/kill_nav.sh` → `bash scripts/launch_sdk.sh` → 1804。
