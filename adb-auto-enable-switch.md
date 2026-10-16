# ADB Auto-Enable 编译与安装流程及功能修改

**版本号：v2026.10.16.1**

## 一、开发流程

### 1. Fork 与创建分支

1. 在 GitHub 打开 `mouldybread/adb-auto-enable`
2. 点 **Fork**，fork 到自己的账号
3. 在自己 fork 的仓库中创建分支 `adb-auto-enable-switch`
   - 网页直接创建，或本地创建后 push：

```bash
git clone https://github.com/你的用户名/adb-auto-enable.git
cd adb-auto-enable
git checkout -b adb-auto-enable-switch
git push --set-upstream origin adb-auto-enable-switch
```

### 2. 克隆分支到本地

```bash
git clone -b adb-auto-enable-switch https://github.com/你的用户名/adb-auto-enable.git
cd adb-auto-enable
```

### 3. 安装开发环境

#### 3.1 安装 JDK 17

```bash
sudo apt update
sudo apt install openjdk-17-jdk -y
```

验证：

```bash
java -version
javac -version
```

#### 3.2 安装 Android SDK 命令行工具

下载 `commandlinetools-linux-*.zip` 并解压：

```bash
mkdir -p ~/android-sdk
cd ~/android-sdk
unzip ~/Downloads/commandlinetools-linux-*_latest.zip
```

修正目录结构：

```bash
mkdir -p cmdline-tools/latest
mv cmdline-tools/bin cmdline-tools/lib cmdline-tools/NOTICE.txt cmdline-tools/source.properties cmdline-tools/latest/
```

#### 3.3 配置环境变量

编辑 `~/.bashrc`，添加：

```bash
export ANDROID_HOME=$HOME/android-sdk
export PATH=$PATH:$ANDROID_HOME/cmdline-tools/latest/bin:$ANDROID_HOME/platform-tools
```

生效：

```bash
source ~/.bashrc
```

#### 3.4 安装 SDK 组件

```bash
sdkmanager --licenses
sdkmanager "platform-tools" "platforms;android-35" "build-tools;35.0.0"
```

验证：

```bash
sdkmanager --list | head
adb version
```

### 4. 生成 release keystore 并配置签名

`app/build.gradle.kts` 已内置环境变量方式的 `signingConfigs`，**签名配置代码不需要修改**。需要做的是生成 keystore 并设置环境变量。

#### 4.1 生成 keystore

```cmd
cd C:\svn\adb-auto-enable-switch\app
keytool -genkey -v -keystore keystore.jks -keyalg RSA -keysize 2048 -validity 10000 -alias release
```

记下输入的 store 密码、key 密码、alias。

#### 4.2 设置环境变量（Windows）

```cmd
setx KEYSTORE_FILE "C:\svn\adb-auto-enable-switch\app\keystore.jks"
setx KEYSTORE_PASSWORD "你的store密码"
setx KEY_ALIAS "release"
setx KEY_PASSWORD "你的key密码"
```

`setx` 只对新开的命令行窗口生效，设置后需重新打开 cmd。

#### 4.3 验证

新开 cmd 窗口，确认环境变量可读取：

```cmd
echo %KEYSTORE_PASSWORD%
echo %KEY_ALIAS%
```

### 5. 修改 versionCode 默认值

`app/build.gradle.kts` 中，把 `versionCode` 默认值从 `12` 改为 `306`：

```kotlin
versionCode = (project.findProperty("CI_VERSION_CODE") as? String)?.toIntOrNull() ?: 306
versionName = project.findProperty("CI_VERSION_NAME") as? String ?: "0.3.3"
```

原因：设备上已装版本的 versionCode 为 305，默认值 12 会导致 `INSTALL_FAILED_VERSION_DOWNGRADE`。改为 306 后版本号递增，可正常覆盖安装。

### 6. 编译生成 APK

```bash
./gradlew assembleRelease
```

生成的 APK 位于：

```
app/build/outputs/apk/release/app-release.apk
```

### 7. 上传到机顶盒安装

```bash
adb connect 机顶盒IP:45555
adb install -r app-release.apk
```

若 adb install 被限制：

```bash
adb push app-release.apk /data/local/tmp/
adb shell pm install -r /data/local/tmp/app-release.apk
```

#### 7.1 安装报错处理

**降级失败**：`INSTALL_FAILED_VERSION_DOWNGRADE: Downgrade detected: Update version code 12 is older than current 305`

- 原因：APK 的 versionCode 低于设备上已装版本。
- 处理：确认第 5 步已把默认值改为 306，重新编译安装。若设备上版本更高，继续调大默认值。
- `adb install -d` 在部分厂商 ROM 上无效，不要依赖。

**签名不匹配**：`INSTALL_FAILED_UPDATE_INCOMPATIBLE: Existing package com.tpn.adbautoenable signatures do not match newer version`

- 原因：自己生成的 keystore 与设备上原有版本签名不一致，Android 不允许覆盖安装。
- 处理：先卸载再安装：

```cmd
adb -s 机顶盒IP:端口 uninstall com.tpn.adbautoenable
adb -s 机顶盒IP:端口 install -r app-release.apk
```

- 注意：卸载会丢失配对信息、ADB 密钥和 prefs 配置（`adb_enabled`、`adb_expiry_minutes`、`web_username`、`web_password` 回到默认值），需重新配对。

### 8. 安装后必做

首次安装后，手动打开一次该 App，否则后台 Web 服务不会启动，无法通过 `http://机顶盒IP:9093` 访问配置界面。

## 二、功能修改需求

### 功能 1：ADB启用 + ADB超时时间

- Web 页面增加“ADB启用”checkbox，默认勾选。
- Web 页面增加“ADB超时时间（分钟）”输入框，默认 1440，最大 4320，输入框下方小字提示“最大 4320（3 天）”。
- 逻辑：adbd 启用后超过设定时长，WiFi 调试不再启动。
- 启用后自动触发一次端口切换，把 adbd 从随机端口切到 targetPort，确保 targetPort 真正可用。
- 运行中由监控线程每分钟检查一次，过期后主动关闭无线调试，无需等下次重启。
- 超时后，用户在 Web 页面点“保存”即可重新计时并恢复启用。

**无 root 实现方式：**

- App 通过 `WRITE_SECURE_SETTINGS` 权限修改系统设置项 `adb_wifi_enabled`（即 `Settings.Global.ADB_WIFI_ENABLED`）。
- 启用时设为 `1`，关闭时设为 `0`。
- `WRITE_SECURE_SETTINGS` 无法由普通应用自行申请，需通过 ADB 命令手动授予一次。
- 关闭开关时，App 将 `adb_wifi_enabled` 设为 `0`，触发系统停止无线调试监听。
- adbd 监听端口是否立即关闭、已配对连接是否立即断开，取决于机顶盒 Android 版本和厂商实现，无 root 方案下无法完全控制。

**无线调试端口机制说明：**

- 无线调试（`adb_wifi_enabled=1`）启动后，adbd 监听的端口是**随机的**（通常为 3xxxx 高位端口），不是固定端口。
- 固定端口（如 45555）需要执行 `tcpip:<targetPort>` 才能让 adbd 切换过去。
- 因此“ADB启用”打开后，仅设 `adb_wifi_enabled=1` 不会让 targetPort 可用，必须额外触发一次端口切换。

### 功能 2：Web 服务登入用户名密码

- Web 页面增加“用户名”和“密码”输入框，默认 `admin` / `admin`，可修改。
- 认证方式：HTTP Basic Auth。
- Web 服务（9093）所有请求均需通过用户名密码验证。
- 密码框始终留空，placeholder 提示“留空表示不修改”；密码不通过 API 返回给前端。
- 用户名或密码修改后，保存时返回 401，浏览器弹登录框，用户输入新凭证即可登录。

### 功能 3：Web 页面配置表单

- 所有配置项（ADB启用、ADB超时时间、用户名、密码）放在页面顶部“⚙️ ADB 设置”区块。
- 整页一个表单，底部一个统一“保存”按钮，一次 POST 提交，写入存储。
- 保存后立即生效。

### 功能 4：System Status 显示 ADB 剩余时间

- System Status 区块增加一行“ADB 剩余时间”。
- 数据来源：`/api/status` 返回 `adbRemainingMinutes`（数值，单位分钟）。
- 前端转换为易读格式：
  - 未启用或无起始时间戳：`N/A`
  - 已过期：`Expired`
  - 不足 1 小时：`X 分钟`
  - 1 小时以上：`X 小时 Y 分钟`（Y=0 时显示 `X 小时`）
  - 超过 24 小时：`X 天 Y 小时`（Y=0 时显示 `X 天`）
- 属于只读状态，由 `refreshStatus()` 每 5 秒刷新。

## 三、Web 页面结构

### 现有区块（保留）

- **System Status**：配对状态、端口状态、设备 IP、上次启动状态、当前端口、ADB 剩余时间
- **Initial Pairing**：配对码、配对端口 + “Pair Device”按钮
- **Device Paired**：已配对时显示，“Reset Pairing”按钮
- **Target Port Configuration**：目标端口输入 + “Save Port” / “Save & Switch”
- **Switch Target Port**：手动切换端口按钮
- **Web Interface Control**：Web 服务开关（与 ADB启用无关）
- **Testing**：Run Test Now
- **Live Logs**：日志区

### 修改后结构

```
┌─────────────────────────────┐
│ ⚙️ ADB 设置                  │  ← 新增区块，顶部
│  [✓] ADB启用                 │
│  ADB超时时间（分钟）: [1440]  │
│   最大 4320（3 天）           │
│  用户名: [admin]             │
│  密码:   [placeholder]       │
│           [ 💾 保存 ]        │
├─────────────────────────────┤
│ 📊 System Status             │  ← 保留原有，新增 ADB 剩余时间
├─────────────────────────────┤
│ 🔐 Initial Pairing           │  ← 保留原有
├─────────────────────────────┤
│ ✅ Device Paired             │  ← 保留原有
├─────────────────────────────┤
│ ⚙️ Target Port Configuration │  ← 保留原有
├─────────────────────────────┤
│ 🔄 Switch Target Port        │  ← 保留原有
├─────────────────────────────┤
│ 🌐 Web Interface Control     │  ← 保留原有
├─────────────────────────────┤
│ 🧪 Testing                   │  ← 保留原有
├─────────────────────────────┤
│ 📋 Live Logs                 │  ← 保留原有
└─────────────────────────────┘
```

### 前端职责划分

| 函数 | 职责 | 频率 |
|------|------|------|
| `loadInitialConfig()` | 回填配置输入框初值（ADB启用、ADB超时时间、用户名、targetPort） | 页面加载时一次 |
| `refreshStatus()` | 刷新 System Status 只读字段（含 ADB 剩余时间） | 每 5 秒 |
| `refreshLogs()` | 刷新日志区 | 每 3 秒 |
| `formatRemaining()` | 把剩余分钟数转成易读格式 | 按需调用 |

配置输入框只在页面加载时回填一次，`refreshStatus()` 不碰配置输入框，避免覆盖用户正在编辑的内容。

## 四、配置项定义

| prefs key | 画面名称 | 默认值 | 备注 |
|-----------|----------|--------|------|
| `adb_enabled` | ADB启用 | `true` | checkbox |
| `adb_expiry_minutes` | ADB超时时间（分钟） | `1440` | 最大 4320 |
| `web_username` | 用户名 | `admin` | |
| `web_password` | 密码 | `admin` | 不回传前端 |
| `adb_enabled_at` | （内部） | `0` | 起始时间戳 |

**`adb_enabled_at` 语义：**

- 在 adbd **从“没启动”变成“启动”** 的状态跃迁时刷新为当前时间。
- 刷新条件（任一成立）：
  1. 用户把 `adb_enabled` 从 false 改成 true（从关到开）。
  2. 首次无值（`adb_enabled_at == 0`）。
  3. 已超时（`elapsed >= adb_expiry_minutes`）且用户保存设置（开关仍启用）—— 视为重新启用。
- 重启 / 服务自然启动时**不刷新**，保留旧时间戳，超时就不启用。
- 超时 → `adb_wifi_enabled=0`，不启动无线调试。
- 监控线程每分钟检查一次，过期后主动关闭无线调试。

## 五、需要修改的代码文件

| 文件 | 路径 | 是否修改 |
|------|------|----------|
| WebServer.java | `app/src/main/java/com/tpn/adbautoenable/WebServer.java` | 是 |
| AdbConfigService.java | `app/src/main/java/com/tpn/adbautoenable/AdbConfigService.java` | 是 |
| build.gradle.kts | `app/build.gradle.kts` | 是（versionCode 默认值） |
| MainActivity.java | `app/src/main/java/com/tpn/adbautoenable/MainActivity.java` | 否 |
| BootReceiver.java | `app/src/main/java/com/tpn/adbautoenable/BootReceiver.java` | 否 |
| AdbHelper.java | `app/src/main/java/com/tpn/adbautoenable/AdbHelper.java` | 否 |

## 六、保存设置的生效模型

| 场景 | `refresh_adb` | 是否刷新 `adb_enabled_at` |
|------|--------------|--------------------------|
| 重启 | false | 否，保留旧值，超时就不启用 |
| 用户关掉再打开 ADB | true | 是，重新计时 |
| 用户只改超时时间，ADB 一直开着、未超时 | false | 否，保持原计时 |
| 已超时，用户点保存（开关仍勾着） | true | 是，重新计时并启用 |
| 首次安装（`enabledAt==0`） | false | 是（靠 `enabledAt == 0` 兜底） |

`refresh_adb` 的计算：`enableTransition || expired`，其中：
- `enableTransition = !oldEnabled && newEnabled`
- `expired = newEnabled && enabledAt>0 && (elapsed >= adb_expiry_minutes)`

保存后立即生效的链路：

1. 用户在 Web 页面点“保存”
2. `POST /api/settings` → `handleSaveSettings()`
3. 写入 prefs（`adb_enabled`、`adb_expiry_minutes`、`web_username`、`web_password`）
4. 计算 `refresh_adb`，通过 `startForegroundService(AdbConfigService, refresh_adb)` 传递
5. `AdbConfigService.onStartCommand()` 识别 `refresh_adb`
6. 调用 `enableWirelessDebuggingImmediately(refreshAdb)`，条件满足时刷新 `adb_enabled_at`
7. 若 `shouldEnableAdb()` 为 true，调用 `startExpiryMonitor()` 启动超时监控
8. 若 `refreshAdb && shouldEnableAdb()`，调用 `switchToTargetPortAsync()` → 等 5 秒 → mDNS 发现 → 端口扫描兜底 → `switchToPort(targetPort)`

## 七、超时监控行为

| 情况 | 监控线程行为 |
|------|-------------|
| `adb_enabled=false` | 下一分钟检查时退出 |
| `adb_enabled_at=0` | 下一分钟检查时退出 |
| `elapsed >= adb_expiry_minutes` | 设 `adb_wifi_enabled=0`，状态改为 `Expired - ADB disabled`，退出 |
| 未过期 | 继续每分钟轮询 |
| 用户运行中修改超时时间 | 下次轮询自动读取新值，无需重启线程 |
| 用户关闭再启用 ADB | `adb_enabled_at` 刷新，监控线程按新时间判断 |
| 超时后用户点保存 | `refresh_adb=true`，`adb_enabled_at` 刷新，监控线程重新启动 |

监控线程靠 `expiryMonitorRunning` 标志防重复启动。每次服务启动时若 `shouldEnableAdb()` 为 true，调用 `startExpiryMonitor()`，已运行则直接返回。

## 八、注意事项

- **JDK 版本必须匹配**：项目 AGP 为 9.3.1，Gradle 9.7.0，JDK 21 可用。
- **SDK 版本**：`compileSdk = 35`，`minSdk = 26`，`targetSdk = 35`，SDK 组件安装 `android-35` / `build-tools;35.0.0`。
- **NanoHTTPD 默认绑定行为**：当前版本既无三参构造函数，也无 `setInetAddress()` 方法，无法从代码层面强制只监听 IPv4。默认监听在 Android 上表现为 `tcp6 :::9093`（IPv4-mapped IPv6），IPv4 客户端访问正常，无需特殊处理。
- **release 签名**：`app/build.gradle.kts` 已内置环境变量方式，需生成 `app/keystore.jks` 并设置 `KEYSTORE_FILE`、`KEYSTORE_PASSWORD`、`KEY_ALIAS`、`KEY_PASSWORD` 四个环境变量。
- **versionCode 默认值**：已改为 `306`，高于设备上已装版本的 `305`，避免降级安装失败。
- **无线调试端口机制**：无线调试启动后 adbd 监听随机端口，targetPort（如 45555）需要 `tcpip:<targetPort>` 切换后才会监听。“ADB启用”打开后必须触发一次端口切换，否则 targetPort 长期不可用。
- **超时后恢复**：超时后 adbd 被关闭，`adb_enabled` 仍为 true。用户在 Web 页面点“保存”，`refresh_adb` 因 `expired=true` 被置为 true，触发重新计时并启用，不会死锁。
- **开机后 targetPort 可用时间**：开机流程 `configureAdb()` 包含 60 秒 WiFi 等待 + 30 秒稳定等待 + 15 秒等 adbd + mDNS/扫描 + 切换，顺利路径约 60~90 秒，mDNS 失败时更长。这是原项目设计，未修改。
- **安装报错处理**：
  - `INSTALL_FAILED_VERSION_DOWNGRADE`：调大 versionCode 默认值后重新编译安装。
  - `INSTALL_FAILED_UPDATE_INCOMPATIBLE`：签名不匹配，需先 `uninstall` 再 `install`，卸载会丢失配对信息、ADB 密钥和 prefs 配置。
- **目录结构不能错**：`bin`/`lib` 必须在 `cmdline-tools/latest/` 下，否则 `sdkmanager` 报错。
- **gradlew 无需单独安装 Gradle**：首次运行会自动下载对应版本。
- **安全风险**：该 App 将 ADB 固定在 targetPort，一旦授权连接即获得永久无限制访问权限。仅在自家可控网络使用，切勿暴露到公网。