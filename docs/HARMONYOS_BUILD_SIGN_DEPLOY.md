# HarmonyOS 本地构建 / 签名 / 安装（可复用流程）

> **用途**：把本仓库编译成可安装到真机的 HAP。适用于在 DevEco Studio 之外用命令行构建、以及排查签名与安装类错误。
>
> **适用前提**：Windows + DevEco Studio（自带 SDK / hvigor / ohpm / hdc / JBR）。本文命令以本机实测路径为例，换机时只需替换 `$deveco`。

---

## 0. 一次性准备

### 0.1 工具链

| 组件 | 路径（示例） | 说明 |
|---|---|---|
| DevEco Studio | `D:\DevEco Studio` | 自带下述全部组件，无需另行下载 SDK |
| SDK | `D:\DevEco Studio\sdk`（API 26 / 26.0.0.105） | `default\openharmony` + `default\hms` |
| hvigor | `D:\DevEco Studio\tools\hvigor\bin\hvigorw.js` | **6.26.8** |
| ohpm | `D:\DevEco Studio\tools\ohpm\bin\ohpm.bat` | |
| node | `D:\DevEco Studio\tools\node\node.exe` | |
| JBR | `D:\DevEco Studio\jbr` | 打包需要可用 Java Runtime |
| hdc | `D:\DevEco Studio\sdk\default\openharmony\toolchains\hdc.exe` | |

> ⚠️ **不要用仓库自带的 `hvigorw.js`**：它按 `hvigor/node_modules` → `node_modules` → 全局命令的顺序查找 hvigor，本机三者都不存在，会报 `Cannot find hvigor`。而且仓库 `hvigor/package.json` 钉的是 **6.24.4**，**不认识 API 26**（CI 也是为此升到 6.26+）。直接用 DevEco 自带的 `hvigorw.js` 最省事。

### 0.2 三个必须自建的文件（都被 .gitignore 忽略）

```powershell
cd <repo>

# ① 构建配置：仓库只提供 .example，真文件含签名信息故被忽略
Copy-Item build-profile.json5.example build-profile.json5

# ② 本地密钥：缺失会让 ArkTS 编译报 "Cannot find module"
cd entry\src\main\ets\config
Copy-Item DevKeySecret.ets.example DevKeySecret.ets
Copy-Item GitHubOAuthConfig.ets.example GitHubOAuthConfig.ets
```

### 0.3 原生模块的硬依赖

`nativelib` 通过 `cmake/ResolveMoonlightAudioHaptics.cmake` 解析 audio-haptics SDK，**缺失直接 `FATAL_ERROR`**。按 CI 的锁定版本检出到仓库根：

```powershell
cd <repo>
git clone https://github.com/AlkaidLab/moonlight-audio-haptics.git audio-haptics-sdk
cd audio-haptics-sdk
git checkout afbbedf19e8ca1616c5c0b6a63051cc1f1723818   # 与 CI 一致，勿用 master
```

---

## 1. 签名

在 DevEco Studio 中：**File → Project Structure → Signing Configs → 勾选 Automatically generate signature**（需登录华为开发者账号，且真机已连接以便绑定设备 UDID）。

生成的材料位于 `C:\Users\<用户名>\.ohos\config\`：`.p12`（密钥库）、`.cer`（证书）、`.p7b`（profile）、`.csr`。

### ⚠️ 关键一步：product 必须引用签名配置

DevEco 会往 `build-profile.json5` 写入 `signingConfigs` 数组，**但不会自动给 product 加引用**。缺引用时 hvigor 只打印一行 WARN 就继续，最终产出**未签名 HAP**（`SignHap` 几毫秒结束）：

```
hvigor WARN: No signingConfig found for product default
```

必须手动补上：

```json5
"products": [
  {
    "name": "default",
    "signingConfig": "default",   // ← 必须有，与 signingConfigs[].name 对应
    ...
  }
]
```

**判断是否真的签名了**：看 `SignHap` 的耗时（真有签名是**秒级**，未签名是 2~3ms），并确认产物里有 `entry-default-signed.hap`。

---

## 2. 构建

```powershell
$repo    = '<repo>'
$deveco  = 'D:\DevEco Studio'

$env:DEVECO_SDK_HOME = "$deveco\sdk"
$env:JAVA_HOME       = "$deveco\jbr"
$env:NODE_HOME       = "$deveco\tools\node"
$env:NODE_OPTIONS    = '--max_old_space_size=8192'

$node    = "$deveco\tools\node\node.exe"
$hvigorw = "$deveco\tools\hvigor\bin\hvigorw.js"

Set-Location $repo

# ① 先构建原生 HAR（与 CI 顺序一致）
& $node $hvigorw assembleHar --mode module -p module=nativelib@default `
    -p product=default -p buildMode=debug --no-daemon --stacktrace

# ② 再打包应用
& $node $hvigorw assembleApp --mode project `
    -p product=default -p buildMode=debug --no-daemon --stacktrace
```

**产物**

| 文件 | 说明 |
|---|---|
| `entry\build\default\outputs\default\entry-default-signed.hap` | **可安装**（签名版） |
| `entry\build\default\outputs\default\entry-default-unsigned.hap` | 未签名，装不上 |
| `nativelib\build\default\outputs\default\nativelib.har` | 原生模块产物 |

首次构建会联网拉 ohpm 依赖；原生 C++ 全量编译较慢，之后增量构建约 20~30 秒。

---

## 3. 安装

### 3.1 USB

```powershell
$hdc = 'D:\DevEco Studio\sdk\default\openharmony\toolchains\hdc.exe'
& $hdc list targets                     # 确认设备，状态需为 Connected（Unauthorized 表示平板未点「允许」）
& $hdc -t <serial> install -r <signed.hap>
```

### 3.2 无线调试（USB 口被占用时首选）

平板：设置 → 系统和更新 → 开发者选项 → 打开「**无线调试**」，记下 IP:端口。

```powershell
& $hdc tconn <ip>:<port>                # 成功输出 Connect OK
& $hdc list targets                     # 形如 172.22.21.134:43235  TCP  Connected
& $hdc -t <ip>:<port> install -r <signed.hap>
```

> 串流走网络，与 USB 无关——USB 只是控制通道，所以无线调试完全不影响功能验证。端口每次重新开关「无线调试」可能变化。

---

## 4. 安装报错对照表（实测）

| 错误码 / 现象 | 原因 | 处理 |
|---|---|---|
| `No signingConfig found for product default` | product 未引用 `signingConfig` | 见 §1，补 `"signingConfig": "default"` |
| `Cannot find module '../config/DevKeySecret'` | 缺本地配置 | 见 §0.2 从 `.example` 复制 |
| `moonlight-audio-haptics not found` | 缺 SDK 检出 | 见 §0.3 |
| `Cannot find hvigor` | 用了仓库自带 wrapper | 改用 DevEco 的 `hvigorw.js` |
| `9568263 install version downgrade` | 目标上已有更高版本 | 加 `-d`：`install -r -d` |
| `9568286 install provision type not same` | 已装**正式（release）签名**版，debug 不能覆盖 | 先卸载正式版（**任何参数都绕不过**，系统级限制） |
| `9568332 install sign info inconsistent` | 用 `uninstall -k` 保留了旧签名数据，新签名读不了 | 再执行一次**不带 `-k`** 的卸载即可清掉（应用已卸载时该命令依然有效） |
| 平板一直 `Unauthorized` | 未接受调试授权 | 解锁平板保持亮屏；设置里「撤销 USB 调试授权」后拔插线重试 |

---

## 5. 签名有效期（**14 天**）与它的真正根因

从 `C:\Users\<用户名>\.ohos\config\*.p7b` 可直接读出：

```json
"validity":{"not-before":1790934982,"not-after":1792144581}   // 相差正好 14 天
"type":"debug"
"bundle-name":"com.alkaidlab.sdream"
"app-identifier":"6918743880746403483"
```

`.cer` 里其实是 3 段 PEM，解出来看有效期：

| 文件 | 有效期 |
|---|---|
| 根证书 `Huawei CBG Root CA G2` | 2020-03-16 → 2049-03-16 |
| 中间证书 `Huawei CBG Developer Relations CA G2` | 2020-07-09 → 2030-07-07 |
| **叶证书** `CN="unknown(...),Development"` | **14 天** |
| **provision profile** | **14 天** |

**根因不是「自动签名只能给 14 天」，而是账号未实名认证。** 官方规则（[证书 FAQ](https://developer.huawei.com/consumer/cn/doc/doccenter-getting-started/agc-help-cert-faq-0000002329508280)、[申请调试证书](https://developer.huawei.com/consumer/cn/doc/App/agc-help-debug-cert-0000002283256797)、[申请发布证书](https://developer.huawei.com/consumer/cn/doc/App/agc-help-release-cert-0000002283336729)）：

| 账号状态 | 调试证书 | 发布证书 |
|---|---|---|
| 已实名认证 | **1 年** | **3 年** |
| **未实名** | **14 天** | 不可申请 |

两个互相独立的证据都指向「未实名」：叶证书有效期正好 14 天；叶证书 subject 里是 `O=unknown`——实名账号这里会是开发者姓名或企业名。官方文档只给了「证书」的有效期数字，Profile 未单列；实测调试 Profile 与调试证书同为 14 天，应属同源。

**到期后**：系统启动应用时校验 profile，debug 包会失效（打不开）。HAP 文件本身不过期，是签名授权过期——所以**存包没用**。

**续期**：

- **推荐**：完成华为开发者账号实名认证（个人：身份证 + 人脸，免费），然后在 DevEco 重新执行一次「Automatically generate signature」。证书 / Profile 变成 **1 年**，**bundleName 不用改、不需要重新配对**。
- 兜底：不实名，每 14 天重新自动签名一次 → 重新构建 → 重新安装。

> ⚠️ **副作用提醒**：DevEco 自动签名是**通过 AGC 申请**证书和 Profile 的（证书名以 `auto_debug` 开头），所以它已经在你账号下为 `com.alkaidlab.sdream` 建好了 APP ID。也就是说调试上游 fork 时，上游的包名会被占用在你账号下——可在 AGC「APP ID」列表核对。若上游作者以后要用同一包名上架，会撞车。

### 5.1 想要「长期有效」的三条路线

官方把签名分成三组，**只有前两组能用 `hdc install`**（[证书和Profile类型及使用场景](https://developer.huawei.com/consumer/cn/doc/harmonyos-faqs/faqs-appgallery-81)）：

| 路线 | 证书类型 | Profile 类型 | `hdc install` | 有效期 | 要换 bundleName |
|---|---|---|---|---|---|
| **A. 本地调试**（当前在用） | 调试证书 | 调试 | ✅ | 1 年（未实名 14 天） | 不用 |
| **B. 指定设备发布** | 发布证书 | **指定设备发布** | ✅ | 随发布证书（3 年） | **要** |
| **C. 上架发布** | 发布证书 | 发布 | ❌ | 随发布证书（3 年） | **要** |

路线 C 用 `hdc` 装会直接报错：

```
INSTALL_FAILED_APP_SOURCE_NOT_TRUSTED
```

官方 FAQ 原文：「AGC 发布的证书仅支持上架使用，不支持本地安装。」（[出处](https://developer.huawei.com/consumer/cn/doc/harmonyos-faqs/faqs-package-structure-51)）

路线 B 是唯一「release 签名 + 本地安装」的组合，官方 FAQ 也确认：「指定设备发布证书是 release 的，可以用 hdc 直接安装，不需要发起邀请测试。」（[出处](https://developer.huawei.com/consumer/cn/doc/doccenter-tools-faq/faqs-app-debugging-71)）

**走路线 B 的完整流程：**

1. **建应用**（即时生效，**不涉及审核**）：AGC → 证书、APP ID和Profile → APP ID → 新建 → 应用类型选「HarmonyOS应用」→ 填**你自己的包名**（如 `com.yourname.sdream`）→ 再到「发布」里关联创建一个「待发布」应用（[文档](https://developer.huawei.com/consumer/cn/doc/doccenter-getting-started/agc-help-create-app-0000002247955506)）。
2. **生成密钥和 CSR**：DevEco → Build → Generate Key and CSR，得到 `.p12` + `.csr`。`.p12` 丢了无法找回，务必备份。
3. **申请发布证书**：AGC → 证书 → 新增证书 → 类型「发布证书」→ 上传 CSR → 下载 `.cer`。每账号最多 3 个。
4. **申请 Profile**：AGC → Profile → 添加 → 类型选 **「指定设备发布」**（**不要选「发布」**）→ 选发布证书 → 选测试设备（**最多 100 台**，需设备 UDID；仅支持 手机 / PC-2in1 / 平板）→ 下载 `.p7b`。每应用最多 100 个 Profile（[文档](https://developer.huawei.com/consumer/cn/doc/app/agc-help-internaltest-profile-0000002283260129)）。
5. **本地配置签名**：`build-profile.json5` 的 `signingConfigs` 填 `storeFile` / `certpath` / `profile` / `keyAlias` / 密码，并在 product 里加 `"signingConfig": "default"`。
6. **构建安装**：`hvigorw assembleHap` → `hdc install`。

**为什么对「临时验证修复」不划算：**

- 必须**换 bundleName** → 新 APP ID、不继承任何应用数据 → **需要重新和 Sunshine 配对**；而且换包名不可能进 PR。
- 设备 UDID 要写进 Profile，换设备得「编辑设备」并**重新下载 Profile**。
- 必须先实名认证。

**关于「审核」的澄清**：建应用、申请证书、申请 Profile 都是**即时生效、不经过审核**。审核只发生在最后一步——**提交版本上架**。所以想拿长期签名并不会被审核卡住，真正的成本是**换包名**。

---

## 6. 验证清单

```powershell
# 版本与权限（设备侧真实登记情况）
& $hdc -t <t> shell "bm dump -n com.alkaidlab.sdream" | Select-Object -Last 3

# 确认产物内容（HAP 就是 zip，可读 module.json）
Add-Type -AssemblyName System.IO.Compression.FileSystem
$zip = [System.IO.Compression.ZipFile]::OpenRead($hap)
($zip.Entries | Where-Object FullName -eq 'module.json') | ForEach-Object {
  $sr = New-Object System.IO.StreamReader($_.Open()); $sr.ReadToEnd(); $sr.Close() }
$zip.Dispose()
```

**日志过滤**：应用日志域为 **`0x4D4C`**（`'ML'` 的十六进制），tag 为各源文件名，便于 `hilog` 精确定位：

```powershell
& $hdc -t <t> hilog > build.log      # 抓取，然后按 tag / 关键字过滤
```

> ⚠️ **平台限制**：`ohos.permission.INPUT_MONITORING` 对三方应用**不可获取**——实测在 `module.json5` 声明后，HAP 内确实包含该权限，但系统安装时静默丢弃、设备只登记其余条目。因此依赖它的 C++ 输入管线监听（全速鼠标轮询）在零售设备上始终失败并回退 ArkUI 事件，日志表现为：
> ```
> MouseInterceptor: 鼠标监听器启动失败: 201 (需要 INPUT_MONITORING 权限, 将回退到 ArkUI 事件)
> ```

---

## 7. 一页速查

```powershell
# 准备（仅首次）
Copy-Item build-profile.json5.example build-profile.json5
Copy-Item entry\src\main\ets\config\DevKeySecret.ets.example      entry\src\main\ets\config\DevKeySecret.ets
Copy-Item entry\src\main\ets\config\GitHubOAuthConfig.ets.example entry\src\main\ets\config\GitHubOAuthConfig.ets
git clone https://github.com/AlkaidLab/moonlight-audio-haptics.git audio-haptics-sdk
# DevEco 自动签名 + 手补 "signingConfig": "default"
#   账号未实名 → Profile 仅 14 天；实名认证后为 1 年（详见 §5）

# 构建
$d='D:\DevEco Studio'; $env:DEVECO_SDK_HOME="$d\sdk"; $env:JAVA_HOME="$d\jbr"
& "$d\tools\node\node.exe" "$d\tools\hvigor\bin\hvigorw.js" assembleHar --mode module -p module=nativelib@default -p product=default -p buildMode=debug --no-daemon
& "$d\tools\node\node.exe" "$d\tools\hvigor\bin\hvigorw.js" assembleApp --mode project -p product=default -p buildMode=debug --no-daemon

# 安装（USB 或无线）
& "$d\sdk\default\openharmony\toolchains\hdc.exe" tconn <ip>:<port>
& "$d\sdk\default\openharmony\toolchains\hdc.exe" -t <ip>:<port> install -r entry\build\default\outputs\default\entry-default-signed.hap
```
