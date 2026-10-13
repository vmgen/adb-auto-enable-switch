> [!NOTE]
> **本程序基于 [mouldybread/adb-auto-enable](https://github.com/mouldybread/adb-auto-enable) 修改，新增以下功能：**
>
> 1. **Web 界面需要认证** —— 访问 Web 界面（9093 端口）需要用户名和密码（默认 `admin` / `admin`），不再是网络内任何人都能打开。
> 2. **在 Web 界面开关无线调试** —— Web 界面提供“ADB启用”开关，可直接开启或关闭无线调试，无需重启设备。
> 3. **无线调试到期自动关闭** —— 提供“ADB超时时间（分钟）”设置（默认 1440，最大 4320），从最后一次启用起算，超过设定时长后自动关闭无线调试。

# 🔧 adb-auto-enable

**每次开机自动启用无线 ADB 调试，并切换到指定端口（默认 5555），无需 root！**

Android 14 引入了更严格的 ADB 安全机制，设备休眠或重启后会禁用并随机化 ADB 端口，导致自动化流程中断。本程序会自动重新启用无线 ADB，保持非交互设备的远程访问能力。

> [!WARNING]
> **安全警告：** 本程序会在指定端口（默认 5555）上启用 Android 调试桥（ADB），这相当于为设备开放了拥有完整系统权限的远程访问入口。虽然 ADB 连接需要 RSA 密钥认证（首次配对需在设备上确认），但**一旦某台电脑获得授权，它就拥有永久且无限制的访问权限**，可以安装应用、读取全部数据、执行 shell 命令、完全控制设备，且不再有任何提示。此外，RSA 认证弹窗存在覆盖攻击（overlay attack）风险，恶意应用可以诱导用户授权。**本程序只能在隔离或可信网络中使用**（例如位于防火墙之后、未做端口转发的家庭网络），**绝不能在公共 WiFi、访客网络或任何你无法完全掌控的网络中使用**。将 ADB 暴露到公网或不可信网络，一旦攻击者通过社工、覆盖攻击或物理接触已配对电脑获得授权，可能导致设备被完全控制。请仅在自己拥有的设备上使用本工具，并确保网络已配置防火墙阻止外部访问。

<p align="center">
  <img src="gui2.png" alt="GUI Description" width="700">
</p>

## 快速开始

### 环境要求：

- Android 14+（已在 Chromecast with Google TV 和较新的 Android 版本上测试）
- Wi-Fi 连接

### 1. 安装

从 [Releases](https://github.com/mouldybread/adb-auto-enable/releases) 下载并安装 APK，或从源码构建：

```bash
git clone https://github.com/mouldybread/adb-auto-enable.git
cd adb-auto-enable
./gradlew assembleRelease
adb install app/build/outputs/apk/release/app-release.apk
```

> [!IMPORTANT]
> **首次安装注意：** 安装 APK 后，**必须在设备上手动打开一次本应用**（*设置 → 应用 → 查看所有应用 → ADB Auto-Enable*），后台 Web 服务才会启动，之后才能访问 `http://设备IP:9093`。

### 2. 初始配对

1. 在浏览器中打开 `http://设备IP:9093`。
2. 在 Android 设备上：
   - 进入 **设置 → 开发者选项 → 无线调试**
   - 点击 **“使用配对码配对设备”**
3. 保持“使用配对码配对设备”界面打开，把界面上的**配对码**和**配对端口**填入 Web 界面。
4. 点击 **“Pair Device”**。

### 3. 验证配置

1. （可选）在 Web 界面点击 **“Switch Target Port Now”** 手动测试配置。
2. （可选）等待 30 秒后查看状态，应显示 “✓ Available”。
3. 重启设备，等待 30-60 秒（服务初始化 + 30 秒系统稳定等待），然后连接指定端口：

```bash
adb connect 设备IP:5555
```

## 故障排查

### 配对失败

如果配对失败，请提交 issue，并附上设备型号、版本、Web 界面的日志和/或 ADB 日志：

```
adb logcat "adbd:V ADBAutoEnable:V *:S"
```

### 开机配置未运行

在 Web 界面 `http://设备IP:9093` 查看日志，或通过 ADB 查看：

```bash
adb logcat "adbd:V ADBAutoEnable:V *:S"
```

关注以下日志：

- `"Boot event detected, starting ADB configuration service immediately..."`
- `"Step 0: Immediately enabling wireless debugging setting..."`
- `"Waiting for WiFi connection..."`
- `"Successfully configured ADB on port 5555!"`

### Web 服务无法访问

Web 服务在前台服务中以 9093 端口运行。如果无法访问：

1. 检查服务是否在运行：
   ```bash
   adb shell dumpsys activity services | grep AdbConfigService
   ```
2. 在主界面确认设备 IP 地址。
3. 确认你在同一网段访问：`http://设备IP:9093`。

## 工作原理

1. **一次性配对**：应用通过 localhost 与自身配对，保存认证密钥。
2. **自动授予权限**：配对后，应用通过本地 ADB 自动为自己授予 `WRITE_SECURE_SETTINGS` 权限。
3. **开机激活**：设备开机时，`BootReceiver` 立即启动 `AdbConfigService`，以满足 Android 14+ 前台服务启动规则。
4. **提前写入设置**：在任何网络或休眠延迟之前，先写入 `adb_wifi_enabled = 1` 到 `Settings.Global`。
5. **稳定与网络等待**：服务等待 Wi-Fi/以太网获取到 IP，并留出 30 秒系统稳定期。
6. **端口发现**：通过 mDNS 或 64 线程并行 socket 扫描（`32768–60999`）发现随机化的 ADB 端口。
7. **自连接与切换**：连接本地 ADB daemon（优先 127.0.0.1 回环，失败则回退到设备 LAN IP），发送 `tcpip:<target_port>` 命令。
8. **完成！** ADB 已在目标端口上可供外部连接。

## 技术细节

### 项目结构

```text
adb-auto-enable/
├── app/src/main/java/com/tpn/adbautoenable/
│   ├── MainActivity.java          # 含 Target Port UI 的 Activity
│   ├── AdbConfigService.java      # 执行开机流程的前台服务
│   ├── BootReceiver.java          # 即时开机广播接收器
│   ├── AdbHelper.java             # Wire 协议实现与端口切换
│   ├── NetworkUtils.java          # 实时 IP 获取与连接检查
│   └── WebServer.java             # 基于 NanoHTTPD 的 Web UI，含实时日志与状态
├── app/src/main/AndroidManifest.xml
└── README.md
```

认证密钥在本地生成，存放于 `/data/data/com.tpn.adbautoenable/files/`。

### 开机流程

```text
LOCKED_BOOT_COMPLETED / BOOT_COMPLETED → BootReceiver
  ↓
启动 AdbConfigService 前台服务（立即）
  ↓
启动 Web 服务（9093 端口）
  ↓
Step 0：立即写入 adb_wifi_enabled = 1
  ↓
Step 1：等待 Wi-Fi 连接（最多 60 秒）
  ↓
Step 2：等待系统稳定（30 秒）
  ↓
Step 3：发现随机化的 ADB 端口（mDNS → 64 线程 socket 扫描兜底）
  ↓
Step 4：连接 ADB daemon（优先 127.0.0.1 回环 → 回退设备 LAN IP）
  ↓
发送 tcpip:<target_port> 命令
  ↓
成功！（必要时最多重试 3 次）
```

## 致谢

- Google，强制我的硬件升级，从而催生了这一堆麻烦
- [This Home Assistant Issue](https://github.com/home-assistant/core/issues/148359)
- 使用 [NanoHTTPD](https://github.com/NanoHttpd/nanohttpd) 作为内嵌 Web 服务器
- 使用 [libadb-android](https://github.com/MuntashirAkon/libadb-android) 实现 ADB 协议