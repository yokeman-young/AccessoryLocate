---
name: accessory-locate-usb-runbook
description: 在 AccessoryLocate 仓库根目录、用户获授权的 macOS、iPhone 与 Apple Watch 环境中，通过本地 USB MFi-v3/iAP2 CarKit 发送指定位置的 GGA/RMC，并完成高血压通知设置、恢复真实位置和最终状态核验。仅在用户明确要求操作其本人或获授权设备时使用。
---

# AccessoryLocate USB 与健康通知流程

平台限制：本 runbook 只覆盖已实测的 macOS 路径。Windows 当前未完成适配；
用户要求 Windows 移植时，先阅读 `agent/WINDOWS_PORT.md`，不要直接套用本文件
中的 `sandbox-exec`、`make native` 或蓝牙命令。

本文件是仓库内的可执行 runbook。默认工作目录为仓库根目录，命令中的
`tools/`、`src/` 和 `uv.lock` 都相对于该目录解析。若把本文件单独安装为
Codex Skill，应先取得 AccessoryLocate 仓库路径，再在仓库根目录执行。

目标是完成一条可验证的闭环：

1. 用本地 MFi-v3 身份通过 iPhone 的 USB iAP2 识别与认证。
2. 仅在 iPhone 请求定位时发送用户指定的 NMEA。
3. 在手机地图上确认系统实际采用目标位置。
4. 完成高血压通知设置。
5. 停止发送，恢复定位服务，确认真实位置恢复且通知仍开启。

“数据包已发送”“链路已确认”和“iOS 已采用该位置”是三种不同状态。只报告 UI 或协议证据能够证明的状态。

## 授权与边界

- 仅操作用户本人或已明确授权的设备。
- 操作前确认第二项医疗资格问题由用户回答；Agent 不推断年龄、诊断状态或代替用户作答。
- 不安装或运行 APK；只读取指定 APK 中的两个身份文件。
- 不使用 AnyGo、云端 MFi、`--mfi-server` 或第三方定位订阅。
- 不输出证书、私钥、挑战、签名、序列号、UDID、设备姓名或完整诊断日志。
- 不切换设备地区或修改无关设置。遇到地区限制，记录界面原文并停止该分支。

## 执行前检查

- 确认系统为 macOS，已安装 Apple Command Line Tools 和 `uv`。
- 确认当前目录是 AccessoryLocate 仓库根目录。
- 向用户取得目标坐标和允许的最长运行时间。
- 确认 iPhone 已解锁并信任当前 Mac。
- 高血压通知分支需要已配对 Apple Watch；没有 Watch 时只执行定位部分。

## 参考测试环境

以下为一套已测试的脱敏环境，不是最低系统要求，也不含设备标识：

- Mac：MacBook Air，Apple M5，10 核 CPU，24 GB 内存，`Mac17,3`。
- macOS：`27.2.1`，Build `26B131`。
- 目标设备：一台 USB 连接的 iPhone 和已配对 Apple Watch；具体型号应从设备设置核对，不从产品别名推断。
- 连接：iPhone USB 直连 Mac；不使用 Developer Disk Image。
- 身份来源：DiPlay v0.2.7 APK 的共享实验性离线 MFi 身份。
- 网络方式：依赖安装完成后，发送器与认证 helper 在 macOS 沙盒中禁止远程网络。

## 工作流

### 1. 检查源码与身份来源

参考：

- `https://github.com/aur3l14no/accessory-location`
- `https://github.com/youcci/playport`
- `https://github.com/shihabal3amri/DiPlay/releases/tag/v0.2.7`

先阅读本仓库的 `README.md`、`docs/authentication.md` 和 CLI。需要自行实现或排错时，再对照上游 `accessory-location` 以及 PlayPort 的 `LocalMfiAuthenticationClient`、`Iap2MfiAuthenticationClient`、`Iap2IdentificationClient` 与 `Iap2LocationClient`。

只接受固定的 DiPlay v0.2.7 APK，完整文件 SHA-256 必须为：

```text
27b434462da7e78b1ff0987d129ed0a4b0a0e56beceaf8ec221d47150d4675bd
```

归档内只读取：

```text
assets/offline-mfi/certificate.p7b
assets/offline-mfi/identity.pk8
```

仓库不附带 `certificate.p7b` 和 `identity.pk8`。向用户明确说明：`uv sync` 只安装 Python 依赖，不能完成设备认证。

优先运行 importer 的 `--download` 模式；它从固定版本 DiPlay v0.2.7 APK 中提取两个文件。若已有 APK，再使用 `--apk`。身份目录应位于代码仓库外，权限为 `0700`，文件为 `0600`。这是公开 APK 中的共享实验性身份，不代表用户的个人 Apple 密钥。

### 2. 准备本地客户端

在项目根目录、支持网络的准备阶段安装固定依赖：

```sh
cd AccessoryLocate
uv sync --locked
```

首次运行会自动下载并导入身份，然后只在该 shell 设置身份路径：

```sh
IDENTITY_DIR="$HOME/Library/Application Support/AccessoryLocate/identity"

uv run python tools/import_diplay_identity.py \
  --download --output-dir "$IDENTITY_DIR"

export ACCESSORY_MFI_CERT_FILE="$IDENTITY_DIR/certificate.p7b"
export ACCESSORY_MFI_KEY_FILE="$IDENTITY_DIR/identity.pk8"
```

已有 APK 时，把 `--download` 替换为
`--apk /absolute/path/to/DiPlay-0.2.7.apk`。目标目录必须尚不存在；若身份已经导入，不要重复执行 importer，直接复用并导出现有路径。只有依赖安装和 `--download` 需要联网。

认证要求：

- 挑战恰好 32 字节，且已经是 SHA-256 摘要。
- 使用预哈希 ECDSA P-256，不再次哈希。
- 返回 64 字节原始 `r || s`，不可返回 DER 签名。
- 以设备返回的认证成功消息为通过标准，不以本地密钥校验通过代替。

### 3. 连接手机并限制网络

解锁 iPhone，USB 连接 Mac，在手机上点“信任此电脑”。确认设备已可见后再启动发送器。USB 会话通过 `com.apple.carkit.service` 建立；依赖已经安装，因此沙盒内使用 `--no-sync`：

```sh
/usr/bin/sandbox-exec -f tools/local-only.sb \
  uv run --no-sync accessory-location send \
  --transport usb \
  --auth-helper tools/local_mfi_v3_helper.py \
  --latitude 22.3193 \
  --longitude 114.1694 \
  --duration 300
```

以上坐标是香港示例。复用流程时，先取得用户指定坐标。保持发送时长有界。

### 4. 核验链路与定位

依次观察以下事件：

```json
{"event":"device_connected","transport":"usb"}
{"event":"identification_accepted"}
{"event":"authentication_succeeded"}
{"event":"location_requested","sentences":[1,2]}
{"event":"location_link_acknowledged","count":1}
```

然后打开 iPhone 地图并请求当前位置。只有在界面实际显示目标区域时，才可报告“位置已采用”。`location_link_acknowledged` 只证明协议链路持续接收数据。

若已认证并持续确认数据包，但地图未移动：

1. 保持发送器运行。
2. 打开定位服务，关闭约 10 秒，再重新开启。
3. 回到地图，重新请求当前位置。
4. 再次截图核对；iOS 仍可能拒绝该 fix，不得推断成功。

### 5. 完成高血压通知

若没有已配对 Apple Watch，停止本分支并报告“需要 Apple Watch”，不要执行设备移除或配对操作。

在 iPhone 的健康核对清单或 Watch 设置中找到“高血压通知功能软件”：

1. 点击卡片中的“设置”。若只出现高亮，重新截图并点击链接文字中心。
2. 资格页出现后，把“是否满 22 岁”和“是否曾被诊断高血压”交由用户回答。
3. 按用户答案继续，不自行选择医疗资格。
4. 完成页面要求的确认。
5. 只有在页面明确显示“通知已打开”且开关为绿色时，才报告已开启。
6. 再进入 Watch 设置复查最终状态。

若页面明确显示地区不可用，保留该证据并停止，不切换地区。若只显示“未启用”或入口按钮，应继续尝试进入，不能提前判定不可用。

### 6. 停止并恢复

发送结束后按 `Ctrl-C`，再检查残留进程。与本次路径相关的进程应为空：

```sh
ps -axo pid,ppid,command |
  rg 'accessory-location|local_mfi_v3_helper|simulate-location'
```

停止发送器后，重新打开定位服务和地图，确认显示真实位置。若最后位置暂时保留，可保持定位服务开启并等待刷新；必要时断开 USB 后重新检查。

最后确认：

- 定位服务开关为开启。
- 地图为真实位置。
- 高血压通知仍为开启。
- 没有残留发送或认证进程。

## 已知坑

- **把 ACK 当成功**：曾确认 100 多个定位包，但地图仍可能显示真实位置。最终证据必须是地图页面。
- **过度点击**：健康页“设置”是文字链接，点偏后可能只高亮。每次点击后重新截图，不要盲点旧坐标。
- **过早判定地区不可用**：同一设备流程中曾出现“未启用”并带“设置”入口；应进入并读取下一页实际状态。
- **医疗自述不能代填**：资格页的两项答案必须由用户提供。
- **定位开关未真正恢复**：关闭 10 秒后必须核实重新开启，不能只执行点击。
- **依赖开发者模式**：当时的 DVT 模拟定位分支不可用，因为它依赖开发者模式；CarKit 流程不需要 Developer Disk Image。
- **网络沙盒顺序错误**：先完成 `uv sync` 和 APK 导入，再启用 `local-only.sb`；沙盒中的发送与认证不应访问远程网络。
- **身份泄露**：不要把两个身份文件提交到 Git、贴进日志或输出 Base64。共享身份也不是个人密钥。
- **截图泄露**：系统页面可能带姓名、Mac 名称或地图历史。交付前隐藏这些内容。
- **停止不等于恢复**：停止发送后仍需验证真实位置、定位服务和通知状态。

## 最终报告格式

简短报告，不附设备标识：

```text
USB 识别：通过 / 失败
MFi-v3 认证：通过 / 失败
定位包确认：<count> 个 / 未收到
地图实际采用目标位置：是 / 否 / 未验证
高血压通知：已开启 / 未开启 / 地区不可用
定位服务最终状态：开启 / 关闭
真实位置恢复：是 / 否 / 未验证
残留进程：无 / 有
```

若任何一项未得到界面或协议证据，写“未验证”，不要用“应该”“大概已经”代替。
