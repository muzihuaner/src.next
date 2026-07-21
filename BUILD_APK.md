# Kiwi Browser (src.next) 构建 APK 指南

本文档介绍如何在 **Linux** 环境下，从本仓库（`src.next`，Kiwi 补丁层）构建出可安装的 Android APK。

当前版本号（见 [CHROMIUM_VERSION](CHROMIUM_VERSION) / [KIWI_VERSION](KIWI_VERSION) / [VERSION](VERSION)）：**150.0.7871.128**

---

## 0. 必读：先搞清楚这个仓库是什么

- **本仓库不是完整的 Chromium 源码**，而是叠加在 Chromium 之上的 **Kiwi 补丁层（overlay）**。你看到的 `base/`、`chrome/`、`components/`、`content/` 等目录，是 Kiwi 修改过的文件，需要**覆盖**到一份完整的 Chromium 源码上才能编译。
- **必须在 Linux 上构建。** Chromium 的 Android 目标官方只支持 Linux（推荐 Ubuntu）。macOS / Windows **无法**编译 Android 版 Chromium。
- **APK 是完整编译 Chromium 得到的**，不是简单打包。首次构建需要下载约 100+ GB 源码/依赖，编译数小时。

### ⚠️ 版本一致性警告（非常重要）

改动 `CHROMIUM_VERSION` / `KIWI_VERSION` / `VERSION` 这三个文件**只改变最终 APK 上显示的版本号**，**不会真正升级内核代码**。

本仓库的补丁代码是针对 **Chromium 105.0.5195** 编写的。因此：

| 你的目标 | 应 checkout 的 Chromium 源码 | 能否编译 |
| --- | --- | --- |
| **A. 就地构建当前仓库代码** | Chromium **105.0.5195.x**（补丁对应的基线） | ✅ 能编译 |
| **B. 真正做出 150 内核** | Chromium **150.0.7871.128** | ❌ 需先把 Kiwi 补丁「变基」到 150，解决大量冲突后才能编译 |

> 换句话说：把版本号写成 150、却拿 105 时代的补丁去覆盖 Chromium 150 源码，会因为跨 45 个大版本的 API 变化而**编译失败**。要得到真正的 150 内核，必须先完成补丁变基（见文末「真正升级内核」一节），这是一项独立的大工程。
>
> **下面的步骤以「方案 A：先跑通构建流程」为主线**，用 `$CHROMIUM_VERSION` 占位，你可按需替换为 105 基线或已变基好的版本。

---

## 1. 硬件与系统要求

| 项目 | 最低 | 推荐 |
| --- | --- | --- |
| 操作系统 | Ubuntu 20.04 / 22.04 64-bit | Ubuntu 22.04 |
| 内存 | 16 GB | 32 GB+（链接阶段吃内存） |
| 磁盘 | 100 GB 可用 | 200 GB+ SSD |
| CPU | 4 核 | 8 核+ |
| 网络 | 能稳定访问 Google 源（`chromium.googlesource.com`） | 建议配好代理 |

首次完整构建通常耗时 **2–6 小时**（取决于 CPU）。

---

## 2. 安装 depot_tools

`depot_tools` 是 Chromium 的构建工具集（含 `gclient`、`gn`、`autoninja` 等）。

```bash
git clone https://chromium.googlesource.com/chromium/tools/depot_tools.git ~/depot_tools

# 加入 PATH（建议写进 ~/.bashrc）
echo 'export PATH="$HOME/depot_tools:$PATH"' >> ~/.bashrc
source ~/.bashrc

# 验证
gclient --version
```

---

## 3. 获取 Chromium 源码

```bash
# 设置目标版本（按上文「版本一致性」选择）
export CHROMIUM_VERSION=105.0.5195.24   # 方案A：与本仓库补丁匹配的基线
# export CHROMIUM_VERSION=150.0.7871.128  # 方案B：需先完成补丁变基

mkdir -p ~/chromium && cd ~/chromium

# 拉取 Android 版源码骨架（--nohooks 先不跑钩子，加快首次拉取）
fetch --nohooks android chromium

cd src

# 切换到指定版本 tag
git fetch --tags
git checkout -b build_$CHROMIUM_VERSION tags/$CHROMIUM_VERSION

# 同步该版本对应的全部第三方依赖
gclient sync -D --with_branch_heads --with_tags --reset
```

> 💡 若只想构建单一版本、节省空间，可在 `gclient sync` 后加 `--no-history` 拉取浅历史。

---

## 4. 安装系统构建依赖

```bash
cd ~/chromium/src

# 一次性安装编译 Android 所需的系统包（需要 sudo）
./build/install-build-deps.sh --android

# 让 gclient 拉取 Android SDK/NDK 等钩子产物
gclient runhooks

# args.gn 里用到了 ccache 加速二次编译，装一下
sudo apt-get install -y ccache
```

---

## 5. 叠加 Kiwi 补丁层

把本仓库（`src.next`）的文件**覆盖**到 `~/chromium/src` 上。假设本仓库已克隆到 `~/src.next`：

```bash
cd ~/src.next
git checkout kiwi          # 确认在 kiwi 分支

# 用 rsync 覆盖，排除 .git 与构建产物
rsync -a --exclude='.git' --exclude='out' ~/src.next/ ~/chromium/src/
```

> ⚠️ 这一步会用 Kiwi 版文件替换掉同名的 Chromium 原版文件。方案 A 下补丁与源码同为 105，覆盖后可正常编译；方案 B（150 源码 + 105 补丁）覆盖后会编译报错。

---

## 6. 配置 args.gn

本仓库在 [.build/production_build_reference/args.gn](.build/production_build_reference/args.gn) 提供了官方生产构建参考参数（`arm64`、official build）。

```bash
cd ~/chromium/src

# 选择输出目录与架构：android_arm64 / android_arm / android_x86 / android_x64
export OUT_DIR=out/android_arm64
mkdir -p $OUT_DIR

# 复制参考 args.gn
cp .build/production_build_reference/args.gn $OUT_DIR/args.gn
```

`args.gn` 关键项说明：

```gn
target_os   = "android"
target_cpu  = "arm64"          # 改这里切换架构：arm / arm64 / x86 / x64
is_debug    = false
is_official_build = true        # 正式优化构建
is_component_build = false

android_default_version_name = "Git"   # 显示的版本名，可改
android_default_version_code = "1"      # 版本号，商店/升级用

android_keystore_name     = "dev"
android_keystore_password = "public_password"
android_keystore_path     = "../../keystore.jks"   # 相对 out 目录 → 即 src/keystore.jks

proprietary_codecs = true       # H.264 等专利编解码
enable_widevine    = true       # DRM
enable_extensions  = true       # Kiwi 的核心卖点：扩展支持
cc_wrapper         = "ccache"   # 二次编译加速；没装 ccache 就删掉这行
```

> 若切换架构，记得同时把 `target_cpu` 改成对应值，并使用独立的 `out/android_<arch>` 目录。

---

## 7. 生成 keystore（签名密钥）

`args.gn` 里指定了签名密钥路径 `../../keystore.jks`（即 `src/keystore.jks`）。编译前必须先生成它，否则打包签名会失败。

以下命令与 CI 一致，生成一个自签名开发密钥：

```bash
cd ~/chromium/src

keytool -genkey -v -keystore keystore.jks -alias dev \
  -keyalg RSA -keysize 2048 -validity 10000 \
  -storepass public_password -keypass public_password \
  -dname "cn=Kiwi Browser (unverified), ou=Actions, o=Kiwi Browser, c=GitHub"
```

> 正式发布应使用你自己的私有密钥并妥善保管；此处密钥仅用于本地测试安装。

---

## 8. 生成构建配置并编译

```bash
cd ~/chromium/src

# 生成 ninja 构建文件
gn gen $OUT_DIR

# 查看/校验参数（可选）
gn args $OUT_DIR --list | head

# 开始编译 APK 目标（chrome_public_apk → 产物 ChromePublic.apk）
autoninja -C $OUT_DIR chrome_public_apk
```

编译成功后，APK 位于：

```
~/chromium/src/out/android_arm64/apks/ChromePublic.apk
```

---

## 9. 安装到设备

```bash
# 用 Chromium 自带脚本安装（自动匹配架构、处理签名）
$OUT_DIR/bin/chrome_public_apk install

# 或直接用 adb
adb install -r $OUT_DIR/apks/ChromePublic.apk
```

---

## 10. 常见问题

- **磁盘不够 / `No space left`**：Chromium 源码 + 构建产物很大，务必预留 150 GB+；`out/` 目录尤其占空间。
- **链接阶段 OOM 被 kill**：内存不足。降低并发 `autoninja -j 4 ...`，或增大 swap。
- **`install-build-deps.sh` 报缺包**：确认是 64 位 Ubuntu，且已 `sudo apt-get update`。
- **方案 B 大量编译错误**：属预期——105 补丁与 150 源码不兼容，须先完成补丁变基。
- **网络拉取失败**：`chromium.googlesource.com` 需稳定网络，建议给 `git`/`gclient` 配置代理。
- **二次编译很慢**：确认 `ccache` 已安装且 `args.gn` 保留 `cc_wrapper = "ccache"`。

---

## 11. 附：用 GitHub Actions 远端构建（可选）

本仓库自带 CI 工作流，把编译外包给 Kiwi 的远端构建服务器，无需本地环境：

- [.github/workflows/build_apk.yml](.github/workflows/build_apk.yml) —— 任意分支手动触发（`workflow_dispatch`），产出测试 APK（arm64）。
- [.github/workflows/build_and_sign_release_apk.yml](.github/workflows/build_and_sign_release_apk.yml) —— 正式发布，四架构（arm/arm64/x86/x64）+ 签名 + 上传 Release。

> ⚠️ 这些工作流依赖 Kiwi 私有基础设施与仓库 Secrets（`BUILD_KEY`、`STORAGE_HOST`、`STORAGE_KEY` 等），且构建服务器 `longbuild.find.kiwi` 属于 Kiwi 官方。Kiwi 项目已于 2025 年 1 月归档，fork 仓库若未配置这些 Secret、或服务器已下线，CI 路径可能不可用。自建构建请以上文 Linux 本地流程为准。

---

## 12. 真正升级内核到 150（进阶）

如果目标是**真正的 Chromium 150 内核**（而非仅改版本号），需要把 Kiwi 补丁从 105 变基到 150：

1. 参考 [.github/workflows/rebase_kiwi_on_top_of_chromium.yml](.github/workflows/rebase_kiwi_on_top_of_chromium.yml) 的流程：将 `kiwi` 分支 rebase 到新的 Chromium 基线分支之上。
2. 逐个解决跨版本的 API/构建冲突（工作量很大，Chromium 每个大版本都有接口变化）。
3. 变基通过、能编译后，再按本文档「方案 B」用 Chromium 150 源码构建。

这是一项持续性的维护工程，通常在专用构建机上进行，不适合在开发机上一次性完成。
