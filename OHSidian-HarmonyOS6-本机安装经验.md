---
tags: [HarmonyOS, OHSidian, Obsidian, DevEcoCLI, HDC]
date: 2026-10-29
---

# OHSidian 在 HarmonyOS 6 上的本机安装经验

## 最终安装结果

本次在没有 DevEco Studio、没有第二台设备协助的情况下，完成了 **OHSidian v1.14.4.2（Obsidian 1.14.4）** 的本机签名、安装和启动验证。

- 设备：华为擎云 HM940
- 系统：HarmonyOS 6.1，API 24，ARM64
- 最终包名：`com.handwer.ohsidianhm`
- 最终安装包：`/storage/Users/currentUser/Download/OHSidian/OHSidian-v1.14.4.2-local-signed.hap`
- 最终 SHA-256：

```text
957fc01c888a6065f8186bbfb97d8166874369c276ef351485f10a5182d87852
```

原包名 `com.mikannqaq.obsidian` 没有卸载，旧数据也没有修改。安装过程中产生的无效测试包已经移除。

## 一、路线选择：哪些方案可行，哪些存在限制

### 1. DevEco Studio 构建

这是原始教程提供的路线，但本机没有 DevEco Studio。完整源码构建还需要：

- HarmonyOS SDK；
- `ohpm`、`hvigor`；
- 自动签名环境；
- 补齐 Git LFS 对应的大体积资源。

本次最终没有重编译整个项目，直接使用了官方 Release 中的预编译 HAP。

### 2. HoKit、小白调试助手

这类工具支持 HAP 重签名和侧载，但首次安装工具自身也会遇到签名问题。部分使用场景还需要另一台设备辅助安装。

本机已经具备终端、Node.js、npm 和 HarmonyBrew，因此采用了更直接的命令行路线。

### 3. Command Line Tools + DevEco CLI

这是本次采用的核心路线：

- `DevEco CLI 1.3.10`：负责华为账号授权、设备管理、调试证书和 Profile 申请；
- `ohos-sdk-toolchains 26.0.0.38`：提供 `hdc`、`hap-sign-tool` 等工具；
- 官方预编译 HAP：提供应用本体；
- 本机无线调试：负责安装、启动和日志检查。

工具链版本为 API 26，应用兼容版本为 API 22，本机为 API 24。这里工具链主要负责连接和签名，没有改变应用的兼容版本，因此可以配合工作。

## 二、安装包来源与完整性检查

### 1. 版本确认

从项目官方 GitCode Release 确认最新版本为：

```text
OHSidian v1.14.4.2
```

源码和发布包均使用同一个标签：

```text
v1.14.4.2
```

避免了源码与二进制资源版本不一致的问题。

### 2. 官方发布包校验

官方未签名 ZIP 的 SHA-256 为：

```text
1c3eea13c6eab68b123bda4ecce8560f4074b592026a87f41be74384a906f7f9
```

下载后计算出的值与 Release 元数据完全一致，没有使用第三方重新打包的 HAP。

### 3. Obsidian 内核校验

补全源码资源后，对 `obsidian.asar` 做了独立检查：

- 大小：`27,477,966` 字节
- MD5：`a2bde3f94594c1d697492998ea6e20aa`

与项目文档记录一致。后续只修改了包名及相关模块索引，**Obsidian 官方 `obsidian.asar` 内核没有被修改**。

这里形成了三层校验思路：

1. 官方 ZIP 用 SHA-256 验证来源；
2. `obsidian.asar` 用大小和 MD5 验证核心内容；
3. 最终本机签名 HAP 单独记录新的 SHA-256。

## 三、本机工具链准备

### 1. 安装 DevEco CLI

通过 npm 安装：

```bash
npm install -g @deveco/deveco-cli@1.3.10
```

确认命令可用：

```bash
devecocli --version
devecocli --help
```

### 2. 安装 `hdc` 和签名工具

通过 HarmonyBrew 安装：

```bash
brew install ohos-sdk-toolchains
```

得到的主要工具包括：

- `hdc 3.2.0f`
- `hap-sign-tool`
- `binary-sign-tool`
- 其他 OpenHarmony 打包工具

### 3. 配置 `COMMAND_LINE_TOOL_PATH`

DevEco CLI 初次运行时报告：

```text
No toolchain found. Set COMMAND_LINE_TOOL_PATH.
```

为此准备了：

```text
/storage/Users/currentUser/opencode/deveco-tools
```

并配置：

```bash
export COMMAND_LINE_TOOL_PATH=/storage/Users/currentUser/opencode/deveco-tools
```

需要特别说明：本次目录中的 `ohpm`、`hvigor` 占位内容只用于满足设备管理和签名流程的检查，**不能把这个精简目录当成完整构建环境**。如需从源码编译，仍应安装完整的官方 Command Line Tools。

## 四、无线调试连接

最初：

```bash
hdc list targets
```

返回 `[Empty]`。开启开发者选项和无线调试后，使用系统显示的地址连接：

```bash
hdc tconn <无线调试地址:端口>
```

连接成功后，DevEco CLI 正确识别到：

- 华为擎云 HM940；
- 设备类型 `2in1`；
- 本机 UDID；
- API 24。

中间还出现过一次目标地址变化：原先的 IP 地址连接失败，但 `hdc list targets` 中已经出现 `127.0.0.1:44303`。后续改用当前列表中的目标后恢复正常。

**经验：无线调试端口可能变化，执行安装前应重新检查 `hdc list targets`，不要长期写死之前的地址。**

## 五、华为授权与实名状态问题

这一部分遇到了三个不同问题。

### 1. 授权链接会过期

每次执行：

```bash
devecocli auth login --no-prompt
```

都会生成带随机端口和随机 `code` 的链接。

第一次链接等待过久后失效，表现为浏览器没有完成授权，CLI 最后返回 `Login failed`。重新生成新链接后解决。

**经验：授权链接应使用当前这一次命令生成的地址，不要复用旧链接。**

### 2. 浏览器已操作，CLI 却没有收到凭据

CLI 会等待本机 `/callback`。如果浏览器只打开授权页面，没有继续到“允许/同意”，本地登录流程就不会完成。

本次重新打开最新链接并完成授权后，CLI 才收到登录信息。

### 3. 已实名仍提示 `realName=false`

签名时反复出现：

```text
Users without real-name verification are not supported.
Complete real-name verification in AppGallery Connect.
```

检查 DevEco CLI 的实现后确认，它读取的是华为服务器返回的 `realName` 字段，检查对象与普通的“华为账号个人信息实名”存在区别。

完成开发者管理中心的认证后，旧授权令牌仍返回原状态。最后通过以下方式刷新：

```bash
devecocli auth logout
devecocli auth login --no-prompt
```

重新授权后才成功申请证书和 Profile。

**经验：**

- 华为账号实名与开发者签名服务使用的实名状态需要分别确认；
- 刚完成认证但接口仍显示旧状态时，退出 DevEco CLI 并重新授权，比单纯重复签名更有效。

## 六、直接签名官方预编译 HAP

生成调试证书和 Profile 后，没有重编译整个项目，而是直接签名官方 HAP。

DevEco CLI 写入 `build-profile.json5` 的 `storePassword` 和 `keyPassword` 是加密内容，不能直接作为明文传给 `hap-sign-tool`。为此使用了本地辅助脚本：

```text
/storage/Users/currentUser/opencode/sign-hap-with-deveco-material.mjs
```

它只在当前操作中读取 DevEco 签名材料并调用官方签名工具，没有把证书密码写入日志或额外明文文件。

签名后还执行了：

- `verify-app`：检查 HAP 签名和摘要；
- `verify-profile`：检查 Profile；
- 包名检查；
- SHA-256 记录。

确认签名有效后，再进入安装步骤。

## 七、遇到的主要错误与解决方法

| 阶段 | 现象 | 实际原因 | 处理方式 |
| --- | --- | --- | --- |
| 工具准备 | `No toolchain found` | 未配置工具链路径 | 安装工具并设置 `COMMAND_LINE_TOOL_PATH` |
| 设备连接 | `No active devices` | 无线调试未连接 | 开启无线调试并执行 `hdc tconn` |
| 华为授权 | `Login failed` | 授权链接超时或流程未完成 | 生成新链接，重新完成授权 |
| 实名检查 | `realName=false` | 开发者认证状态未生效，旧令牌未刷新 | 完成开发者认证后退出并重新登录 |
| 部署预编译包 | 缺少 `output_metadata.json` | `devecocli run --skip-build` 仍要求已有构建元数据 | 对预编译 HAP 改用 `hdc install` |
| 第一次安装 | `9568332 install sign info inconsistent` | 旧包名残留不同的 `appIdentifier` | 使用自定义包名，保留旧数据 |
| 第二次安装 | `9568329 verify signature failed` | Profile 已改包名，但 HAP 内仍记录旧包名 | 同步修改 `pack.info` 和 `module.json` |
| 启动阶段 | `Cannot find module 'ets/Application/AbilityStage'` | `modules.abc` 的模块索引仍包含旧包名 | 使用等长包名并同步更新编译索引 |

## 八、最关键的包名问题

这是本次最值得记录的部分。

### 1. 系统没有查到应用，仍可能存在签名身份残留

第一次安装原包名时，`bm dump` 查不到已安装应用，但系统日志明确指出：

```text
the appIdentifier or appId of the new bundle is not the same as old one
```

说明仅检查“当前是否安装”还不够。出现签名冲突时，应继续查看 `hilog` 中的 `BMSInstaller` 记录。

直接卸载可能清除旧应用数据，所以本次选择让新包使用独立包名。

### 2. `pack.info` 不是唯一的包名来源

只修改 `pack.info` 后，安装器仍报告：

```text
provisionBundleName: com.dragonxing.ohsidian
bundleName: com.mikannqaq.obsidian
```

进一步扫描发现旧包名还存在于：

- `module.json`
- `resources/base/profile/shortcuts_config.json`
- `ets/modules.abc`
- `ets/symbolMap.map`

其中安装器包名校验直接使用了 `module.json` 中的值。

### 3. 只改文本配置还不足以保证启动

修改 `pack.info`、`module.json` 和快捷方式后，应用安装成功，但启动时 ArkTS 无法找到 `AbilityStage`。

原因是编译后的 `modules.abc` 仍按旧包名组织模块索引。

### 4. 为什么最终选择 `com.handwer.ohsidianhm`

旧包名：

```text
com.mikannqaq.obsidian
```

与最终包名：

```text
com.handwer.ohsidianhm
```

长度相同，都是 22 个字符。这样可以在不改变模块偏移的情况下，同步替换：

- `pack.info`：1 处；
- `module.json`：1 处；
- `shortcuts_config.json`：1 处；
- `ets/modules.abc`：287 处；
- `ets/symbolMap.map`：5,087 处。

替换后再次扫描，包内已没有旧包名残留，最终安装和启动均成功。

**这种方法成立的关键是“等长替换 + 全项校验”。换用其他长度的包名时，不能直接对 `modules.abc` 做简单字节替换。拥有完整构建环境时，从源码重新生成包会更通用。**

## 九、安装后的验证方式

本次没有只依赖“安装成功”提示，而是做了多层验证。

### 1. 系统登记信息

```bash
bm dump -n com.handwer.ohsidianhm -u 102
```

确认：

- 包名正确；
- `versionCode = 11404002`；
- `versionName = 1.14.4.2`。

当前用户为 `102`，所以应用数据和日志路径也使用 `102`。其他设备或账号环境下需要根据实际用户编号调整。

### 2. 进程检查

最终看到：

- `com.handwer.ohsidianhm`
- `com.handwer.ohsidianhm:Renderer`
- `com.handwer.ohsidianhm:GPU`
- `com.handwer.ohsidianhm:NetworkService`

说明主进程和主要运行组件都在持续运行。

### 3. Obsidian 主日志

日志位置：

```text
/data/app/el2/102/base/com.handwer.ohsidianhm/files/obsidian.log
```

其中确认：

- `obsidian.asar` 加载成功；
- 主窗口和渲染进程创建成功；
- `app.getVersion = 1.14.4`；
- 托盘初始化成功；
- 注入脚本执行成功；
- 华为云同步通道注册完成；
- 本机与远端 `versionCode` 都是 `11404002`，`hasUpdate=false`。

## 十、以后升级或重新安装时的推荐顺序

1. 从官方 Release 下载同版本源码和发布包。
2. 校验官方 SHA-256。
3. 检查本机无线调试状态，重新获取当前端口。
4. 确认 `devecocli auth status` 显示已登录。
5. 执行 `devecocli signature generate`。
6. 签名 HAP，并执行 `verify-app`、`verify-profile`。
7. 安装前检查同包名应用及签名冲突。
8. 安装后检查系统登记、进程和 `obsidian.log`。
9. 调试签名到期或版本升级时重新生成签名。

最终 HAP 的调试 Profile 绑定了本机 UDID，只适合本机调试使用，不应分享给其他设备安装。

## 相关链接

- [OHSidian 官方 GitCode 项目](https://gitcode.com/handwer/ObsidianHM)
- [OHSidian v1.14.4.2 Release](https://gitcode.com/handwer/ObsidianHM/releases/tag/v1.14.4.2)
- [DevEco CLI 官方快速入门](https://developer.huawei.com/consumer/cn/doc/doccenter-deveco-studio/ide-deveco-cli-install)
