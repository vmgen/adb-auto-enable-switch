# ADB Auto-Enable 编译与安装流程及功能修改

**版本号：v2026.10.13.18**

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
adb connect 机顶盒IP:5555
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
- 启用后异步触发端口切换，确保 targetPort 真正可用。

**无 root 实现方式：**

- App 通过 `WRITE_SECURE_SETTINGS` 权限修改系统设置项 `adb_wifi_enabled`（即 `Settings.Global.ADB_WIFI_ENABLED`）。
- 启用时设为 `1`，关闭时设为 `0`。
- `WRITE_SECURE_SETTINGS` 无法由普通应用自行申请，需通过 ADB 命令手动授予一次。
- 关闭开关时，App 将 `adb_wifi_enabled` 设为 `0`，触发系统停止无线调试监听。
- adbd 监听端口是否立即关闭、已配对连接是否立即断开，取决于机顶盒 Android 版本和厂商实现，无 root 方案下无法完全控制。

### 功能 2：Web 服务登入用户名密码

- Web 页面增加“用户名”和“密码”输入框，默认 `admin` / `admin`，可修改。
- 认证方式：HTTP Basic Auth。
- Web 服务（9093）所有请求均需通过用户名密码验证。
- 密码不传送到前端，密码框始终留空，placeholder 提示“留空表示不修改”。
- 用户名或密码变化后，保存时返回 401，浏览器弹登录框要求重新登录。

### 功能 3：Web 页面配置表单

- 所有配置项（ADB启用、ADB超时时间、用户名、密码）放在页面顶部“⚙️ ADB 设置”区块。
- 整页一个表单，底部一个统一“保存”按钮，一次 POST 提交，写入存储。
- 保存后立即生效。

## 三、Web 页面结构

### 现有区块（保留）

- **System Status**：配对状态、端口状态、设备 IP、上次启动状态、当前端口
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
│  密码:   [留空表示不修改]     │
│           [ 💾 保存 ]        │
├─────────────────────────────┤
│ 📊 System Status             │  ← 保留原有
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

## 四、配置项定义

| prefs key | 画面名称 | 默认值 | 备注 |
|-----------|----------|--------|------|
| `adb_enabled` | ADB启用 | `true` | checkbox |
| `adb_expiry_minutes` | ADB超时时间（分钟） | `1440` | 最大 4320 |
| `web_username` | 用户名 | `admin` | |
| `web_password` | 密码 | `admin` | 不传前端 |
| `adb_enabled_at` | （内部） | `0` | 起始时间戳 |

**`adb_enabled_at` 语义：**

- 只在用户保存设置、且 `adb_enabled=true` 时刷新为当前时间。
- 开机 / 服务自然启动时不刷新，只读取旧时间戳判断是否超时。
- 超时 → `adb_wifi_enabled=0`，不启动无线调试，直到用户重新保存设置。

## 五、需要修改的代码文件

### 1. WebServer.java

**文件路径**：`app/src/main/java/com/tpn/adbautoenable/WebServer.java`

#### 改动 1：新增 import

```java
import android.content.Intent;
import java.util.Base64;
```

#### 改动 2：serve() 入口加认证 + 路由

```java
@Override
public Response serve(IHTTPSession session) {
    String uri = session.getUri();
    Method method = session.getMethod();

    // ---- HTTP Basic Auth ----
    String authHeader = session.getHeaders().get("authorization");
    if (!isAuthorized(authHeader)) {
        Response resp = newFixedLengthResponse(
                Response.Status.UNAUTHORIZED, "text/plain", "Unauthorized");
        resp.addHeader("WWW-Authenticate", "Basic realm=\"ADB Auto-Enable\"");
        return resp;
    }

    if (uri.equals("/api/settings") && method == Method.POST) {
        return handleSaveSettings(session);
    } else if (uri.equals("/api/pair") && method == Method.POST) {
        return handlePairing(session);
    } else if (uri.equals("/api/status")) {
        return handleStatus();
    } else if (uri.equals("/api/test")) {
        return handleTest();
    } else if (uri.equals("/api/switch")) {
        return handleSwitch();
    } else if (uri.equals("/api/port") && method == Method.POST) {
        return handleSetPort(session);
    } else if (uri.equals("/api/logs")) {
        return handleLogs();
    } else if (uri.equals("/api/reset") && method == Method.POST) {
        return handleReset();
    } else if (uri.equals("/api/webserver") && method == Method.POST) {
        return handleToggleWebServer(session);
    } else {
        return newFixedLengthResponse(getHTML());
    }
}
```

#### 改动 3：新增认证与保存设置方法

```java
private boolean isAuthorized(String authHeader) {
    if (authHeader == null || !authHeader.startsWith("Basic ")) {
        return false;
    }
    try {
        String decoded = new String(Base64.getDecoder().decode(authHeader.substring(6)));
        String[] parts = decoded.split(":", 2);
        if (parts.length != 2) return false;

        SharedPreferences prefs = getPrefs();
        String user = prefs.getString("web_username", "admin");
        String pass = prefs.getString("web_password", "admin");
        return parts[0].equals(user) && parts[1].equals(pass);
    } catch (Exception e) {
        Log.e(TAG, "Auth decode error", e);
        return false;
    }
}

private Response handleSaveSettings(IHTTPSession session) {
    try {
        Map<String, String> files = new HashMap<>();
        session.parseBody(files);
        Map<String, List<String>> params = session.getParameters();

        SharedPreferences prefs = getPrefs();

        // 写之前先记录旧凭证，用于判断是否变化
        String oldUsername = prefs.getString("web_username", "admin");
        String oldPassword = prefs.getString("web_password", "admin");

        SharedPreferences.Editor editor = prefs.edit();

        editor.putBoolean("adb_enabled", params.containsKey("adb_enabled"));

        List<String> expList = params.get("adb_expiry_minutes");
        if (expList != null && !expList.isEmpty()) {
            try {
                int m = Integer.parseInt(expList.get(0));
                editor.putInt("adb_expiry_minutes", Math.max(1, Math.min(4320, m)));
            } catch (NumberFormatException ignored) {}
        }

        List<String> userList = params.get("web_username");
        if (userList != null && !userList.isEmpty() && !userList.get(0).isEmpty()) {
            editor.putString("web_username", userList.get(0));
        }

        List<String> passList = params.get("web_password");
        if (passList != null && !passList.isEmpty() && !passList.get(0).isEmpty()) {
            editor.putString("web_password", passList.get(0));
        }

        editor.apply();
        Log.i(TAG, "Web API: settings saved");

        // 保存后立即刷新 adbd 状态，并重新计时
        Intent intent = new Intent(context, AdbConfigService.class);
        intent.putExtra("refresh_adb", true);
        context.startForegroundService(intent);

        // 判断凭证是否变化
        String newUsername = prefs.getString("web_username", "admin");
        String newPassword = prefs.getString("web_password", "admin");
        boolean credentialsChanged =
                !oldUsername.equals(newUsername) || !oldPassword.equals(newPassword);

        if (credentialsChanged) {
            // 凭证变了，返回 401，浏览器立刻弹登录框
            Log.i(TAG, "Web API: credentials changed, requesting re-auth");
            Response resp = newFixedLengthResponse(
                    Response.Status.UNAUTHORIZED, "text/plain",
                    "凭证已修改，请重新登录");
            resp.addHeader("WWW-Authenticate", "Basic realm=\"ADB Auto-Enable\"");
            return resp;
        }

        Response resp = newFixedLengthResponse(
                Response.Status.REDIRECT, "text/plain", "");
        resp.addHeader("Location", "/");
        return resp;
    } catch (Exception e) {
        Log.e(TAG, "Web API: save settings error", e);
        return newFixedLengthResponse(Response.Status.INTERNAL_ERROR,
                "application/json", "{\"error\":\"" + e.getMessage() + "\"}");
    }
}
```

#### 改动 4：状态 JSON 不包含密码

```java
boolean adbEnabled = prefs.getBoolean("adb_enabled", true);
int adbExpiryMinutes = prefs.getInt("adb_expiry_minutes", 1440);
String webUsername = prefs.getString("web_username", "admin");

String json = String.format(Locale.US,
        "{\"lastStatus\":\"%s\",\"currentPort\":%d,\"isPaired\":%b,\"adb5555Available\":%b,\"targetPort\":%d,\"webServerEnabled\":%b,\"adbEnabled\":%b,\"adbExpiryMinutes\":%d,\"webUsername\":\"%s\"}",
        lastStatus, currentPort, isPaired, adbTargetAvailable, targetPort, webServerEnabled,
        adbEnabled, adbExpiryMinutes, webUsername
);
```

#### 改动 5：HTML 顶部加设置区块

在 `<h1>🔧 ADB Auto-Enable Configuration</h1>` 之后、第一个 `<div class="card">` 之前插入：

```java
"    <h1>🔧 ADB Auto-Enable Configuration</h1>\n" +
"    \n" +
"    <div class=\"card\">\n" +
"        <h2>⚙️ ADB 设置</h2>\n" +
"        <form method=\"POST\" action=\"/api/settings\">\n" +
"            <label><input type=\"checkbox\" id=\"adb-enabled-input\" name=\"adb_enabled\" /> ADB启用</label><br><br>\n" +
"            <label>ADB超时时间（分钟）: <input type=\"number\" id=\"adb-expiry-input\" name=\"adb_expiry_minutes\" min=\"1\" max=\"4320\" /></label><br>\n" +
"            <small style=\"font-size:12px;color:#888;\">最大 4320（3 天）</small><br><br>\n" +
"            <label>用户名: <input type=\"text\" id=\"web-username-input\" name=\"web_username\" /></label><br><br>\n" +
"            <label>密码: <input type=\"password\" id=\"web-password-input\" name=\"web_password\" placeholder=\"留空表示不修改\" /></label><br><br>\n" +
"            <button type=\"submit\">💾 保存</button>\n" +
"        </form>\n" +
"    </div>\n" +
"    \n" +
"    <div class=\"card\">\n" +
"        <h2>📊 System Status</h2>\n" +
```

#### 改动 6：前端重构，职责分离

新增 `loadInitialConfig()`，只在页面加载时回填配置项初值：

```javascript
function loadInitialConfig() {
    fetch('/api/status')
        .then(r => r.json())
        .then(data => {
            const portInput = document.getElementById('target-port-input');
            if (portInput) portInput.value = data.targetPort;

            const adbEnabledInput = document.getElementById('adb-enabled-input');
            if (adbEnabledInput) adbEnabledInput.checked = data.adbEnabled;

            const adbExpiryInput = document.getElementById('adb-expiry-input');
            if (adbExpiryInput) adbExpiryInput.value = data.adbExpiryMinutes;

            const webUsernameInput = document.getElementById('web-username-input');
            if (webUsernameInput) webUsernameInput.value = data.webUsername;

            // 密码框不回填，保持为空
        });
}
```

`refreshStatus()` 只刷新 System Status 只读字段，删掉所有配置输入框的回填：

```javascript
function refreshStatus() {
    fetch('/api/status')
        .then(r => r.json())
        .then(data => {
            document.getElementById('pairing-status').innerHTML = data.isPaired ?
                '<span class="status good">✓ Paired</span>' :
                '<span class="status bad">✗ Not paired</span>';
            document.getElementById('port-status').innerHTML = data.adb5555Available ?
                '<span class="status good">✓ Available</span>' :
                '<span class="status bad">✗ Not available</span>';
            document.getElementById('last-status').textContent = data.lastStatus;
            document.getElementById('current-port').textContent = (data.currentPort === -1) ? 'NONE' : data.currentPort;

            if (data.isPaired) {
                document.getElementById('pairing-card').style.display = 'none';
                document.getElementById('paired-card').style.display = 'block';
            } else {
                document.getElementById('pairing-card').style.display = 'block';
                document.getElementById('paired-card').style.display = 'none';
            }

            if (data.adb5555Available) {
                document.getElementById('switch-card').style.display = 'none';
            } else {
                document.getElementById('switch-card').style.display = 'block';
            }
        });
}
```

初始化处：

```javascript
loadInitialConfig();
refreshStatus();
refreshLogs();

setInterval(refreshStatus, 5000);
startLogsAutoRefresh();
```

### 2. AdbConfigService.java

**文件路径**：`app/src/main/java/com/tpn/adbautoenable/AdbConfigService.java`

#### 改动 1：新增 shouldEnableAdb()

```java
private boolean shouldEnableAdb() {
    SharedPreferences prefs = getPrefs();

    if (!prefs.getBoolean("adb_enabled", true)) {
        return false;
    }

    long enabledAt = prefs.getLong("adb_enabled_at", 0);
    if (enabledAt > 0) {
        int expiryMinutes = prefs.getInt("adb_expiry_minutes", 1440);
        long elapsedMinutes = (System.currentTimeMillis() - enabledAt) / 60000;
        if (elapsedMinutes >= expiryMinutes) {
            return false;
        }
    }
    return true;
}
```

#### 改动 2：enableWirelessDebuggingImmediately() 加判断与参数

```java
private void enableWirelessDebuggingImmediately(boolean refreshTimer) {
    try {
        if (!shouldEnableAdb()) {
            Log.i(TAG, "ADB disabled or expired, setting adb_wifi_enabled=0");
            Settings.Global.putInt(getContentResolver(), "adb_wifi_enabled", 0);
            return;
        }

        Log.i(TAG, "Enabling wireless debugging & disabling key revocation...");
        Settings.Global.putInt(getContentResolver(), "adb_wifi_enabled", 1);
        Settings.Global.putLong(getContentResolver(), "adb_allowed_connection_time", 0L);

        if (refreshTimer) {
            getPrefs().edit()
                .putLong("adb_enabled_at", System.currentTimeMillis())
                .apply();
            Log.i(TAG, "adb_enabled_at refreshed by user save");
        }
    } catch (SecurityException e) {
        Log.e(TAG, "Settings write failed - WRITE_SECURE_SETTINGS missing", e);
    } catch (Exception e) {
        Log.e(TAG, "Settings write unexpected error", e);
    }
}
```

#### 改动 3：onStartCommand() 调用点调整 + 新增端口切换触发

```java
ensureTclAutostartProtectionAsync();

boolean refreshAdb = intent != null && intent.getBooleanExtra("refresh_adb", false);
// 开机/服务启动时 refreshTimer=false，不重置计时；保存设置触发时 refreshTimer=true，重新计时
enableWirelessDebuggingImmediately(refreshAdb);

// 用户保存设置触发启用后，异步切换 adbd 到 targetPort，确保端口真正可用
if (refreshAdb && shouldEnableAdb()) {
    switchToTargetPortAsync();
}
```

#### 改动 4：新增 switchToTargetPortAsync()

```java
private void switchToTargetPortAsync() {
    new Thread(() -> {
        try {
            Log.i(TAG, "Post-enable: waiting for adbd to start...");
            Thread.sleep(5000);  // 等 adbd 在随机端口启动

            if (!shouldEnableAdb()) {
                Log.i(TAG, "Post-enable: ADB no longer enabled, aborting port switch");
                return;
            }

            Log.i(TAG, "Post-enable: discovering current ADB port...");
            int port = discoverAdbPortViaMdns();
            if (port == -1) {
                Log.i(TAG, "Post-enable: mDNS failed, scanning ports...");
                port = scanForAdbPort();
            }

            if (port == -1) {
                Log.e(TAG, "Post-enable: could not find ADB port");
                updateStatus("Failed - port not found after enable");
                return;
            }

            Log.i(TAG, "Post-enable: found ADB on port " + port);

            int targetPort = getTargetPort();
            if (port == targetPort) {
                Log.i(TAG, "Post-enable: already on target port " + targetPort);
                updateStatus("Success - ADB on port " + targetPort);
                return;
            }

            Log.i(TAG, "Post-enable: switching to target port " + targetPort + "...");
            AdbHelper adbHelper = new AdbHelper(this);
            String deviceIP = getDeviceIP();

            boolean success = adbHelper.switchToPort("127.0.0.1", port, targetPort);
            if (!success && !deviceIP.equals("127.0.0.1")) {
                Log.i(TAG, "Post-enable: switch failed via loopback, retrying via " + deviceIP);
                success = adbHelper.switchToPort(deviceIP, port, targetPort);
            }

            if (success) {
                Log.i(TAG, "Post-enable: switched to port " + targetPort);
                updateStatus("Success - ADB on port " + targetPort);
            } else {
                Log.e(TAG, "Post-enable: failed to switch to port " + targetPort);
                updateStatus("Failed - could not switch port after enable");
            }
        } catch (InterruptedException e) {
            Log.e(TAG, "Post-enable: interrupted", e);
        } catch (Exception e) {
            Log.e(TAG, "Post-enable: unexpected error", e);
        }
    }, "post-enable-port-switch").start();
}
```

#### 改动 5：configureAdb() 加判断

```java
private boolean configureAdb() {
    try {
        if (!shouldEnableAdb()) {
            Log.i(TAG, "ADB disabled or expired, skipping configureAdb");
            Settings.Global.putInt(getContentResolver(), "adb_wifi_enabled", 0);
            updateStatus("ADB disabled or expired");
            return false;
        }

        // Re-assert wireless debugging & auto-revocation settings
        Settings.Global.putInt(getContentResolver(), "adb_wifi_enabled", 1);
        Settings.Global.putLong(getContentResolver(), "adb_allowed_connection_time", 0L);

        // 原有逻辑...
    }
}
```

### 3. build.gradle.kts

**文件路径**：`app/build.gradle.kts`

修改 `versionCode` 默认值：

```kotlin
versionCode = (project.findProperty("CI_VERSION_CODE") as? String)?.toIntOrNull() ?: 306
versionName = project.findProperty("CI_VERSION_NAME") as? String ?: "0.3.3"
```

### 4. MainActivity.java

**不需要修改。** 现有 UI 是 Web 服务开关，保留。

### 5. BootReceiver.java

**不需要修改。** 它只负责启动 Service。

### 6. AdbHelper.java

**不需要修改。** 端口切换逻辑与 ADB启用、ADB超时时间正交。

## 六、保存设置的生效模型

| 场景 | 是否刷新 `adb_enabled_at` | 是否触发端口切换 | 行为 |
|------|--------------------------|-----------------|------|
| 开机 / 服务自然启动 | 否 | 否（由开机流程 configureAdb 处理） | 读旧时间戳，超时就 `adb_wifi_enabled=0` |
| 用户保存设置 | 是 | 是（`switchToTargetPortAsync`） | 重新计时，立即启用，异步切到 targetPort |

保存后立即生效的链路：

1. 用户在 Web 页面点“保存”
2. `POST /api/settings` → `handleSaveSettings()`
3. 写入 prefs
4. `startForegroundService(AdbConfigService, refresh_adb=true)`
5. `AdbConfigService.onStartCommand()` 识别 `refresh_adb`
6. 调用 `enableWirelessDebuggingImmediately(true)` → 刷新 `adb_enabled_at`，立即写 `adb_wifi_enabled`
7. 若 `shouldEnableAdb()` 为 true，触发 `switchToTargetPortAsync()` → 等 5 秒 → mDNS 发现随机端口 → `switchToPort(targetPort)`
8. 端口切换完成后，targetPort 真正可用，页面 System Status 显示 Available

## 七、关键机制说明

### 无线调试端口 vs targetPort

- **无线调试（`adb_wifi_enabled`）端口是系统随机分配的**，通常是 3xxxx 高位端口，与 targetPort 无关。
- **targetPort（默认 5555，可配置为 45555 等）只有在执行 `tcpip:<targetPort>` 之后才会监听。**
- 所以“启用 ADB”后，targetPort 不会自动可用，必须触发一次端口切换。
- 页面 System Status 的 `ADB Target Port` 检查的是 targetPort，不是无线调试端口。
- JSON 字段名 `adb5555Available` 是原项目历史命名，值对应 targetPort，不是固定 5555。

### 启用 ADB 后的端口切换

- 开关打开 → `adb_wifi_enabled=1` → adbd 在随机端口启动。
- `switchToTargetPortAsync()` 等 5 秒后，通过 mDNS 发现随机端口（失败则端口扫描），再 `tcpip:<targetPort>` 切换到 targetPort。
- 切换成功后，targetPort 才真正可用。

### 开机路径

- 开机走 `isBootConfigMode` → `configureAdbWithRetries()` → `configureAdb()`。
- 内含 60 秒 WiFi 等待、30 秒稳定、15 秒等 adbd、mDNS 10 秒、端口扫描 15 秒。
- 顺利路径约 60~90 秒，mDNS 失败路径可能 100 秒以上。
- 这些等待是原项目设计，本次不改。

## 八、注意事项

- **JDK 版本必须匹配**：项目 AGP 为 9.3.1，Gradle 9.7.0，JDK 21 可用。
- **SDK 版本**：`compileSdk = 35`，`minSdk = 26`，`targetSdk = 35`，SDK 组件安装 `android-35` / `build-tools;35.0.0`。
- **NanoHTTPD 默认绑定行为**：当前版本既无三参构造函数，也无 `setInetAddress()` 方法，无法从代码层面强制只监听 IPv4。默认监听在 Android 上表现为 `tcp6 :::9093`（IPv4-mapped IPv6），IPv4 客户端访问正常。
- **release 签名**：`app/build.gradle.kts` 已内置环境变量方式，需生成 `app/keystore.jks` 并设置 `KEYSTORE_FILE`、`KEYSTORE_PASSWORD`、`KEY_ALIAS`、`KEY_PASSWORD` 四个环境变量。
- **versionCode 默认值**：已改为 `306`，高于设备上已装版本的 `305`。
- **安装报错处理**：
  - `INSTALL_FAILED_VERSION_DOWNGRADE`：调大 versionCode 默认值后重新编译安装。
  - `INSTALL_FAILED_UPDATE_INCOMPATIBLE`：签名不匹配，需先 `uninstall` 再 `install`。
- **目录结构不能错**：`bin`/`lib` 必须在 `cmdline-tools/latest/` 下，否则 `sdkmanager` 报错。
- **gradlew 无需单独安装 Gradle**：首次运行会自动下载对应版本。
- **安全风险**：该 App 将 ADB 固定在 targetPort，一旦授权连接即获得永久无限制访问权限。仅在自家可控网络使用，切勿暴露到公网。