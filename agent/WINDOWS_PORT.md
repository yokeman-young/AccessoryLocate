# AccessoryLocate Windows 移植任务书

仅在用户明确要求把 AccessoryLocate 移植到 Windows 时读取本文件。本文件是
Agent 的改造任务和验收标准，不是“Windows 已支持”的声明。当前发布版只在
macOS 上完成过端到端验证。

可直接发给 Agent 的启动语：

```text
请在 AccessoryLocate 仓库根目录阅读 agent/WINDOWS_PORT.md，先完成 Windows
USB-only 的可行性检查和分阶段移植。不要在真实 iPhone 测试前声称支持；
得到用户授权后，再完成认证、定位采用与恢复验证。
```

## 目标

把现有的 macOS 参考实现改造成一个可维护的 Windows 10/11 x64 版本。第一阶段
只交付 USB CarKit/iAP2：

1. Windows 能通过 `pymobiledevice3` 发现已解锁、已信任的 USB iPhone。
2. 能打开 `com.apple.carkit.service` 并完成现有 iAP2 握手。
3. 能通过本地 MFi-v3 helper 完成识别和认证。
4. 只在手机请求 GGA/RMC 后发送有界定位数据。
5. 停止发送后，用户确认定位服务恢复且地图回到真实位置。
6. 所有失败路径给出明确原因，不把协议 ACK 当作地图已采用位置。

Windows 移植完成不等于高血压通知自动完成。健康设置仍是 iPhone 与已配对
Apple Watch 上的用户界面流程，并受地区、设备能力和用户医疗自述约束。

## 不可改变的边界

- 仅操作用户本人或明确获授权的设备。
- 保持本地认证；不得接入 AnyGo、云端 MFi、`--mfi-server` 或第三方定位订阅。
- 不把 `certificate.p7b`、`identity.pk8`、APK、设备标识或抓包提交到仓库。
- 第一阶段不移植 macOS 蓝牙桥。Windows 选择蓝牙时应在启动前报
  “Windows Bluetooth transport is not implemented”，不要静默回退到 USB。
- 不自动修改 Windows 防火墙、安全策略、Apple 服务或系统定位设置。需要这些
  改动时先取得用户明确同意。
- 不代替用户填写高血压资格问题，不切换地区来规避不可用状态。
- 真实设备测试和定位服务切换必须先得到用户授权；每次发送保持有时限。

## 当前模块边界

| 模块 | 作用 | 移植判断 |
| --- | --- | --- |
| `src/accessory_location/wire.py` | iAP2 分帧和参数编码 | 优先保持不变 |
| `src/accessory_location/link.py` | 控制链路和握手 | 优先保持不变 |
| `src/accessory_location/auth.py` | JSON-lines 本地认证边界 | 优先保持不变 |
| `src/accessory_location/profile.py` | 识别、NMEA 和定位消息 | 优先保持不变 |
| `src/accessory_location/cli.py` | USB 会话和事件循环 | 抽出平台会话边界；不要复制协议循环 |
| `src/accessory_location/transport.py` | 蓝牙子进程和地址解析 | 视为 macOS 研究路径，不作为 Windows 首版依赖 |
| `native/bluetooth_bridge.m` | IOBluetooth/Foundation 桥 | 仅 macOS |
| `Makefile` | `xcrun clang` 原生蓝牙构建 | 仅 macOS |
| `tools/local-only.sb` | macOS 沙盒网络隔离 | 仅 macOS |
| `tools/local_mfi_v3_helper.py` | 本地认证 helper | 保留协议；验证 Windows 编码和子进程行为 |
| `tools/import_diplay_identity.py` | 身份下载、校验和导入 | 保留校验；验证 Windows 路径、权限和原子写入 |

推荐的改造顺序是先在 `cli.py` 与协议核心之间建立 USB 会话/传输接口，再把
macOS 蓝牙和沙盒能力放进平台专属模块。协议模块、事件名称和身份处理语义应
保持不变，Windows 适配不应复制一套新的 iAP2 状态机。

## Windows 设计约束

- 使用 `pathlib.Path`、`os.environ` 和 `%LOCALAPPDATA%`。建议身份目录为
  `%LOCALAPPDATA%\AccessoryLocate\identity`，但不要硬编码某个用户的绝对路径。
- 子进程使用参数数组和 UTF-8 JSON-lines，不经过 shell。正确引用含空格路径。
- 平台不支持的能力必须在启动前给出可操作错误，例如蓝牙桥和 macOS
  `sandbox-exec`。
- 进程退出、`Ctrl-C` 和超时清理要跨平台。不得留下发送器、认证 helper 或
  USB 会话。
- 网络隔离没有跨平台等价物时，文档必须写“未强制隔离”。不要用“Windows
  防火墙已配置”代替实际验证。
- 身份文件必须位于 checkout 外，导入目标不存在时才写入，并验证证书与私钥
  配对。
- USB 后端必须使用仓库锁定的 `pymobiledevice3` 版本验证。Apple Mobile
  Device 支持、iTunes/Apple Devices 或驱动组合只按实际探测结果记录，不凭
  经验宣称可用。

## 执行顺序

### 1. 建立基线

确认当前平台、Python、`uv`、依赖版本和工作树状态，然后运行：

```powershell
uv sync --locked
uv run ruff check src tests tools
uv run pytest -q
uv run accessory-location --help
```

`make native` 是 macOS 专用检查。Windows 上不要把它改成假成功；应明确记录
“原生蓝牙桥未构建”。

### 2. 做 USB 可行性探针

在用户同意连接手机后：

1. 确认 Windows 已安装 Apple 提供的移动设备支持，且设备管理器无异常。
2. 使用当前锁定依赖提供的 `pymobiledevice3` 设备发现能力确认 USB iPhone
   可见；命令以本机版本的 `--help` 为准。
3. 解锁 iPhone，点击“信任此电脑”。
4. 只做连接与服务探测，不先改协议代码。

如果设备发现失败，停止 USB 实现并报告驱动、USB 后端或信任状态证据。不要用
模拟设备通过替代真机结论。连续三次同类探针无进展时，汇总错误并请求用户
处理硬件/驱动前提。

### 3. 分阶段改造

保持提交小而可回滚：

1. 增加平台会话/传输边界，让 USB 打开服务与协议循环解耦。
2. 保留现有事件和认证消息编号。
3. 增加 Windows 身份目录和清晰的平台错误。
4. 把 macOS 蓝牙和沙盒入口隔离，不影响 USB 路径。
5. 增加 Windows 可运行的合成身份、loopback、路径和错误分支测试。
6. 增加 `windows-latest` CI，仅运行不依赖真机的检查。

不要把“新增 Windows CI 通过”写成“Windows 设备支持通过”。真机证据必须单列。

### 4. 获得授权后做真机验收

至少保留以下独立状态：

| 状态 | 证据 |
| --- | --- |
| Windows USB 识别 | `device_connected` |
| 配件识别 | `identification_accepted` |
| 本地 MFi-v3 认证 | `authentication_succeeded` |
| 手机请求定位 | `location_requested` |
| 链路持续发送 | `location_link_acknowledged` 递增 |
| 地图实际采用 | 用户在 iPhone 地图看到目标位置 |
| 真实位置恢复 | 停止发送并恢复定位服务后，地图回到真实位置 |

建议先用 120 至 300 秒的有界会话。打开 iPhone 地图并请求当前位置。若已认证
且有 ACK 但地图不动，保持发送器运行，按用户授权关闭定位服务约 10 秒再开启，
重新请求位置；仍不动时记录“未验证”，不要推断成功。

停止后必须确认定位服务为开启、真实位置可见、无残留进程。若最后一次位置仍在
显示，不要据此宣称恢复完成。

### 5. 健康通知分支

只有用户要求且存在已配对 Apple Watch 时，才继续健康设置。指导用户在 iPhone
的健康核对清单或 Watch 设置中进入“高血压通知功能软件”，由用户回答年龄和
诊断资格问题。只有页面明确显示通知已开启时才报告成功。地区不可用时记录界面
原文并停止。

Windows 主机不能替代 iPhone 健康设置，也不应在文档中声称可以自动开启通知。

## 测试要求

必须覆盖：

- Windows 上 `wire`、`link`、`auth`、`profile` 的合成测试。
- 无硬件时的身份导入成功、校验失败和目标已存在分支。
- `windows-latest` 上的 lint、测试、CLI help 和平台错误信息。
- 蓝牙选择在 Windows 上快速失败。
- 含空格身份路径和 UTF-8 JSON-lines。
- 超时、键盘中断和无残留子进程。

真机测试必须另列环境并注明是否完成：Windows 版本、CPU 架构、Python/uv/
`pymobiledevice3` 版本、Apple 驱动组合、iPhone 是否信任、USB 连接方式、
地图是否实际采用、真实位置是否恢复。型号、序列号、UDID 和身份材料不写入报告。

## 常见坑

- **把 macOS 测试当 Windows 结论**：协议测试通过只证明 Python 逻辑。
- **把 `make native` 失败当移植失败**：Windows 首版不需要 macOS 蓝牙桥，但
  必须明确该功能不支持。
- **错误处理 USB 后端**：先证明锁定版本的 `pymobiledevice3` 能发现设备，再改
  CarKit 逻辑。
- **路径含空格或中文**：不要拼接 shell 字符串，不要依赖系统默认编码。
- **把 ACK 当地图成功**：必须看 iPhone 地图的实际显示。
- **过早结束恢复检查**：停止发送不等于 iOS 已回到真实位置。
- **伪造网络隔离**：没有 Windows 沙盒时写清楚，不要静默删掉限制说明。
- **提交身份材料**：两个身份文件从 APK 导入，不进入 Git、日志或报告。
- **擅自改健康资格**：年龄和诊断问题只能由用户回答。

## 完成标准

只有同时满足以下条件，才能把 Windows USB 路径标记为“已测试”：

- Windows CI 的合成测试通过。
- 真机出现识别、认证、定位请求和持续链路确认。
- iPhone 地图实际显示目标位置。
- 停止后真实位置恢复，定位服务状态已核验。
- Windows 蓝牙、沙盒和健康设置的边界在文档中准确说明。
- 工作树无身份文件、APK、设备标识、截图和本机路径。

若只完成代码和合成测试，应写“Windows 移植已实现，真机未验证”；若没有地图
采用或恢复证据，对应项必须写“未验证”。
