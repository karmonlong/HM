# 鸿蒙 PC 开发环境搭建指南：OHSidian + Git 同步 + HarmonyBrew 工具链

> 适用对象：使用鸿蒙电脑（HarmonyOS 6/7，MateBook Pro 等）的同事
> 目标：在鸿蒙 PC 上运行 Obsidian 笔记（含 Git 多端同步）、MarkText，并搭建命令行开发环境
> 全程约 30-60 分钟，无需第二台电脑

---

## 0. 背景说明

- Obsidian **官方不支持鸿蒙**，OHSidian 是社区移植版（通过 Electron 兼容层在鸿蒙原生运行时上跑原版 Obsidian）
- 鸿蒙应用必须经过签名才能安装，因此采用"自己构建+自动签名"的方式，华为开发者账号免费即可
- 本指南基于 GitCode 上的 `handwer/ObsidianHM`（v1.14.4.2）验证通过

**合规提示**：OHSidian 代码为 BSD 3-Clause 许可，可自由分发；但 Obsidian 本体是商业软件，根据官方政策，**2 人以上的公司用于工作用途需购买商业许可**。公司内部推广前请先确认采购或许可事宜。

---

## 1. 前置准备

| 项目 | 要求 |
|------|------|
| 设备 | 鸿蒙 PC（MateBook Pro 等 2in1）或大屏平板，HarmonyOS 6 及以上 |
| 账号 | 华为账号（需完成开发者实名认证，免费） |
| 软件 | DevEco Studio 鸿蒙电脑版（应用市场安装） |
| 可选 | GitNext（鸿蒙版 git，用于 obsidian-git 插件同步） |

---

## 2. 第一步：完成华为开发者实名认证

1. 浏览器打开 `developer.huawei.com`，登录华为账号
2. 鼠标悬停右上角头像 → 点击"实名认证"→ 选择**个人开发者**
3. 推荐选**人脸识别认证**：用手机（华为浏览器或微信）扫页面二维码，按提示刷脸（二维码 15 分钟内有效）
4. 回到电脑端补充信息、签署协议，即时通过

> 也可选银行卡认证（即时）或人工审核（1-2 个工作日）。

---

## 3. 第二步：安装 DevEco Studio 鸿蒙电脑版

1. 打开**应用市场 → 我的 → 应用尝鲜**
2. 搜索 **DevEco Studio** 安装（公测版，6.1 预览版）
3. 安装包有几个 GB，"安装中"状态包含下载阶段，耐心等待；下载管理里可看真实进度

---

## 4. 第三步：准备项目文件

项目分两部分：**源码** + **大体积二进制**（Git LFS 指针，需从发布包提取）。两者**必须版本配套**。

### 4.1 下载

1. GitCode 仓库：`https://gitcode.com/handwer/ObsidianHM`
   - 下载源码 zip（仓库页 → 克隆/下载 → 下载 zip）
   - 下载发布版 zip（仓库 Releases，如 `OHSidian-v1.14.4.2.zip`）
2. 解压源码到工作目录（如 `文档/OHSidianHM`）
3. 解压发布版 zip，得到 `OHSidian-default-unsigned.app`（本质是 zip），再解压它取出 `electron-default.hap`

### 4.2 关键：补全二进制文件

源码里以下文件是 LFS 占位指针（仅百字节），必须用发布版 `electron-default.hap` 里的真实文件替换（hap 也是 zip，直接解压）：

| 源码内路径 | 发布包内路径 |
|---|---|
| `electron/libs/arm64-v8a/`（3 个 .so） | `libs/arm64-v8a/` |
| `web_engine/src/main/resources/resfile/icudtl.dat`、`resources.pak`、`chrome_*.pak`、`snapshot_blob.bin`、`v8_context_snapshot.bin`、`locales/*.pak` | `resources/resfile/` 同名文件 |
| `web_engine/.../resources/app/obsidian.asar` | `resources/resfile/resources/app/obsidian.asar` |

用命令行批量完成（在源码根目录执行）：

```bash
# 先解压发布版 hap
unzip -o -q electron-default.hap "resources/resfile/*" "libs/*" -d /tmp/hapx

# 填入资源文件
SRC=web_engine/src/main/resources/resfile
for f in chrome_100_percent.pak chrome_200_percent.pak icudtl.dat resources.pak snapshot_blob.bin v8_context_snapshot.bin; do
  cp "/tmp/hapx/resources/resfile/$f" "$SRC/$f"
done
cp /tmp/hapx/resources/resfile/locales/*.pak "$SRC/locales/"
cp /tmp/hapx/resources/resfile/resources/app/obsidian.asar "$SRC/resources/app/obsidian.asar"

# 填入原生库
cp /tmp/hapx/libs/arm64-v8a/*.so electron/libs/arm64-v8a/
```

### 4.3 两个必要的小改动

1. **复制构建配置**：把根目录的 `build-profile.example.json5` 复制一份命名为 `build-profile.json5`（缺它 DevEco 会提示"不是鸿蒙项目"）
2. **修改包名**（推荐）：编辑 `AppScope/app.json5`，把 `bundleName` 改成一个自己的反域名，如 `com.example.ohsidian`
   - 原因：避免与机器上可能存在的旧版发生签名冲突，也避免与他人构建的包名撞车

---

## 5. 第四步：构建与安装

1. DevEco Studio → **Open** → 选择项目目录（如 `文档/OHSidianHM`）
2. 等待右下角同步（hvigor/ohpm 依赖下载）完成
3. **File → Project Structure → Signing Configs** → 勾选 **Automatically generate signature** → 登录已实名认证的华为账号
   - IDE 会自动生成调试证书、注册本机、申请 profile，全自动
4. 右上角设备选择器选**本机**，点 ▶️ **Run 'electron'**
5. 首次构建约 1-2 分钟（打包约 250MB），完成后自动安装并启动

构建窗口里大量 `WARN`（deprecated、arkts-no-classes-as-obj 等）属正常现象，只有 `ERROR` 才需要处理。

---

## 6. 第五步（可选）：配置 obsidian-git 插件同步

前置：本机已装 GitNext（鸿蒙版 git，HNP 包，git 位于 `/data/service/hnp/bin/git`）。

### 6.1 插件内配置

OHSidian → 设置 → Git 插件 → **Advanced** 区域，三个框分别填：

**Custom Git binary path**
```
/data/service/hnp/bin/git
```

**Additional PATH environment variable paths**（纯路径）
```
/data/service/hnp/bin
```

**Additional environment variables**（KEY=VALUE，每行一个）
```
GIT_CONFIG_COUNT=1
GIT_CONFIG_KEY_0=safe.directory
GIT_CONFIG_VALUE_0=*
```

填完点 **Reload with new environment variables**。

> 说明：`safe.directory=*` 解决 git 的"dubious ownership"拒绝（vault 文件属主与 app 运行用户不同）；环境变量方式属 git 的"命令级"配置，可让该受保护项生效（需 git ≥ 2.36，GitNext 为 2.45）。

### 6.2 仓库初始化与署名

在系统终端（hmos-clt）对 vault 目录执行一次：

```bash
cd <你的vault目录>
git init                       # 若插件尚未初始化
git config user.name "你的名字"
git config user.email "你的邮箱"
```

### 6.3 GitHub 远程同步

**推荐：HTTPS + 访问令牌（PAT）**。实测 SSH 私钥放在公共存储时，插件内的 ssh 子进程因沙箱文件权限读取失败（报 "inaccessible or not found"），HTTPS 可完全绕过这类问题。

1. GitHub → Settings → Developer settings → Personal access tokens → Tokens (classic) → 生成令牌，勾 **repo** 权限
2. GitHub 上创建**空仓库**（不要勾 README）
3. 终端执行一次（把 `<令牌>` 换成刚生成的）：
   ```bash
   cd <你的vault目录>
   git remote add origin "https://<用户名>:<令牌>@github.com/<用户名>/<仓库名>.git"
   git push -u origin master
   ```
   令牌只保存在仓库本地的 `.git/config`，不会随同步上传。
4. 之后插件内的 commit/push/pull 开箱即用，无需配置 GIT_SSH_COMMAND

**备选：SSH 方式**（如果坚持用 SSH）：密钥生成后公钥加到 GitHub；注意私钥文件权限在鸿蒙公共存储上无法常规修改，插件子进程可能读不到，需把密钥放在 vault 的 `.git/` 目录内并在插件 Advanced → Additional environment variables 里配置 `GIT_SSH_COMMAND` 指向它，同时用 `UserKnownHostsFile` 指定一个应用可读的 known_hosts。

### 6.4 内网部署：GitLab 社区版 / 极狐 GitLab / Gitea

公司内部署时推荐换成自建 git 服务，obsidian-git 与任何标准 git 服务兼容，且内网有三大优势：**速度快、数据不出内网、不依赖外网访问**。

配置与 GitHub 几乎相同，仅地址不同：

1. 在 GitLab 上创建空项目（不勾 README）
2. 生成令牌：GitLab → 头像 → **Edit profile / Preferences → Access Tokens** → 创建 PAT，勾选 `read_repository` + `write_repository`
3. 终端执行一次：
   ```bash
   cd <你的vault目录>
   git remote add origin "https://<用户名>:<令牌>@gitlab.公司域名/<组>/<项目>.git"
   git push -u origin master
   ```
4. **自签名证书注意**：如果公司 GitLab 用内部 CA/自签名证书，HTTPS 会报证书校验失败，在插件 Advanced → Additional environment variables 加一行：
   ```
   GIT_SSL_NO_VERIFY=true
   ```
   （或将公司 CA 证书导入系统后去掉该项）

> 令牌按最小权限发放、建议设置过期时间并定期轮换；revoke 后更换 `.git/config` 中的令牌即可。

---

## 7. 命令行开发环境：HarmonyBrew（推荐）

HarmonyBrew 是 Homebrew 的鸿蒙移植版——鸿蒙 PC 上的命令行包管理器。装上它之后，git / node / python 以及 OpenCode 等 AI CLI 工具都是一条命令的事。

### 7.1 安装 HarmonyBrew

前置条件：
- 设置 → 系统 → 开发者选项 → 打开
- 设置 → 隐私和安全 → 高级 → 打开"运行来自非应用市场的扩展程序"
- **如装有 GitNext / DevBox 需先卸载**（路径冲突；卸载 GitNext 后 obsidian-git 的 git 路径需切换到 HarmonyBrew 的 git，见 7.3）

终端执行：

```bash
zsh -c "$(curl -fsSL https://harmonybrew.atomgit.com/install.sh)"
echo 'eval "$(/storage/Users/currentUser/.harmonybrew/bin/brew shellenv)"' >> ~/.zshrc
eval "$(/storage/Users/currentUser/.harmonybrew/bin/brew shellenv)"
```

### 7.2 常用包一览

```bash
brew install git       # git（卸载 GitNext 后的替代）
brew install node      # node + npm
brew install python    # python
brew install opencode  # OpenCode 终端 AI 编程
brew install codex     # OpenAI Codex CLI（可选）
```

npm 生态的工具（需先 `brew install node`）：

```bash
npm install -g @tencent-ai/codebuddy-code   # CodeBuddy CLI
npm install -g @moonshot-ai/kimi-code       # Kimi Code
```

> **注意**：HarmonyBrew 的 npm 全局包装到 `~/npm/bin`，该目录不在默认 PATH 中。装完后执行一次：
> ```bash
> echo 'export PATH="$HOME/npm/bin:$PATH"' >> ~/.zshrc
> source ~/.zshrc
> ```
> 否则运行时会报 `command not found`。之后所有 npm 全局工具都生效。

### 7.3 与 obsidian-git 的联动

卸载 GitNext 后，把 obsidian-git 插件设置（Advanced）里的两处路径改为 HarmonyBrew 的 git：

- Custom Git binary path：`/storage/Users/currentUser/.harmonybrew/bin/git`
- Additional PATH：`/storage/Users/currentUser/.harmonybrew/bin`

改完点 Reload with new environment variables。仓库内的署名/远程/令牌配置不受影响。

---

## 8. 常见问题（FAQ）

| 现象 | 原因 | 解决 |
|------|------|------|
| 双击 `.app` 弹"打开方式" | `.app` 是未签名打包格式，系统不识别 | 不要双击，走本指南的构建安装流程 |
| 提示"不是鸿蒙项目" | 缺 `build-profile.json5` | 复制 `build-profile.example.json5` 改名 |
| 安装报 `code: 9568332 sign info inconsistent` | 机器上已有同包名、不同签名的旧版 | 卸载旧版；找不到旧版就改 bundleName 绕过 |
| 启动闪退，日志含 `icu_util.cc Invalid file descriptor to ICU data` | 源码与二进制版本混搭（如旧源码+新发布包） | 确保源码与发布版同一版本，重新同步全部 resfile 文件 |
| obsidian-git 提示 git 无效 | 找不到 git 可执行文件 | 按 6.1 配置 git 路径与 PATH |
| git 操作报 `dubious ownership` | 文件属主与 app 用户不一致 | 按 6.1 配置 `safe.directory` 三行环境变量 |
| 提交报缺 user.name/email | 未配置署名 | 按 6.2 在终端做仓库级配置 |
| 插件 push 报 `inaccessible or not found` | SSH 私钥权限问题（鸿蒙公共存储上 chmod 无效，插件子进程读不到私钥） | 改用 6.3 的 HTTPS + 令牌方案 |

**抓崩溃日志的方法**：DevEco Studio 底部 Log 窗口 → 复现问题 → 看 `Reason:` 行和 `#00~#08` 堆栈；崩溃日志文件也会存到系统 `下载/cppcrash-*.log`。

---

## 9. 附：安装 MarkText（可选）

MarkText 同样是 Electron 应用，官方不支持鸿蒙，但已有社区移植版，流程与本指南完全相同。

**项目地址**：`https://atomgit.com/OpenHarmonyPCDeveloper/ohos_marktext`（鸿蒙 PC 开发者社区出品，已完成原生模块 Mock、菜单栏重写等适配）

**安装步骤**：

1. 浏览器打开项目地址 → "克隆/下载" → **下载 ZIP**（该平台命令行下载需登录，用浏览器即可）；如有"发行版/Releases"安装包也一并下载
2. 解压源码到工作目录（如 `文档/ohos_marktext`）
3. 按本指南 **4.2** 的方式补齐 LFS 二进制（从配套的发布包中提取，版本必须一致）
4. 按 **4.3** 复制 `build-profile.example.json5` 为 `build-profile.json5`、修改 `bundleName`（如 `com.example.marktext`）
5. 按 **第 5 节** 在 DevEco Studio 中打开项目 → 自动签名 → Run

> 该移植版与 OHSidian 共用同一套 Electron 兼容层思路，FAQ（第 7 节）中的问题排查方法同样适用。

---

## 10. 已知局限

- 华为账号一键登录、华为云同步不可用（该移植版已移除 AGC 依赖）
- 应用显示名为"OHsidian"，随源码构建，官方 Obsidian 升级需要等待移植版跟进
- 这是社区项目，稳定性不如官方桌面版；重要笔记请保持 git 备份习惯

---

*指南整理自实际装机验证（MateBook Pro, HarmonyOS 7.0.0），2026-10*
