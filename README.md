# 文档导航

> 最后修改时间：2026-11-16
> SDK 接口编码：1801~1822

## 给部署/集成方的入口

| 文档 | 内容 | 语言 |
|------|------|------|
| [SDK 全流程操作手册](./SDK全流程操作手册.md) | **建图 → 定位 → 打点 → 导航**、安全规范、CPU/日志监控、Foxglove 调试、故障排查；含 §0.3 一页纸最短流程 | 中文 |
| [Full SDK Workflow Guide](./en/SDK_FULL_WORKFLOW_GUIDE.md) | 同上，英文版 | English |
| [SDK 关键参数速查](./SDK关键参数速查.md) | 现场常用参数、默认值、作用、生效方式与调整风险 | 中文 |
| [Key Parameters](./en/SDK_KEY_PARAMETERS.md) | 同上，英文版 | English |
| [SDK 接口文档](./SDK接口文档.md) | 13 个 topic 接口、字段、错误码、状态码、日志 | 中文 |
| [SDK Interface Reference](./en/SDK_INTERFACE.md) | 同上，英文版 | English |
| [SDK 状态机](./SDK状态机.md) | 11 个状态、完整迁移、`freezing` 子流程、自动保护 | 中文 |
| [SDK State Machine](./en/SDK_STATE_MACHINE.md) | 同上，英文版 | English |
| [SDK 调用示例](./SDK调用示例.md) | 报文级：每个接口可直接复制的 `ros2 topic pub` JSON | 中文 |
| [SDK Examples](./en/SDK_EXAMPLES.md) | 同上，英文版 | English |

## 其它文档

| 文档 | 说明 |
|------|------|
| [导览接口文档](./导览接口文档.md) | `/guide_tour/*` 服务接口（导览适配层，不是 SDK） |
| [导览操作手册](./导览操作手册.md) | 导览部署/WebUI/SSH |
| [异常事件上报文档](./异常事件上报文档.md) | 导览侧异常事件与错误码 |

## 使用约定

1. **SDK 只有 topic + JSON**：请求发 `/slam/xxx`，响应从 `/slam/status` 回来。
2. **`success=0` 不是终态**：必须按 `request_id` 等 `1`（成功）或 `2`（失败）。
3. **所有请求/响应都写入 `logs/slam_sdk.log`**（前缀 `[API <code> <name>]`）。
4. **参数/地图变更后重启对应服务**；程序文件不要手动修改。
5. **同一台机器只运行一套 SDK**，不要重复启动。
6. **不导航时把底盘切到遥控模式并确保遥控器在手**；CPU 高时不要移动关节或下发行走命令（包括遥控器），紧急情况先降低负载、人员保持距离。

## 维护说明

- 中文文档与英文文档内容对应；改接口/状态机时请同时更新两侧。
- 文档中的命令、JSON、错误消息以代码为准；参数默认值以 `config/params/slam_sdk.yaml` 为准。
