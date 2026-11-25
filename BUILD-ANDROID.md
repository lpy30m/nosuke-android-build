# Android Server / Gadget 编译

这份源码是修改过的 frida-core。以下命令从包含 `configure`、`meson.build`、
`releng/` 的 frida-core 目录运行。默认目标为 Android arm64，保留对 32 位
ARM 进程的支持。当前配置需要 Android NDK r29、Python 3、Node.js >= 18
和 npm，推荐 Node.js 22。

## Kali / Linux 本地编译

将整个 `nosuke-src`（包括隐藏的 `.git` 目录）复制到虚拟机本地磁盘，
例如 `~/nosuke-src`。以下步骤假设虚拟机是 x86_64，可用 `uname -m` 确认。

```bash
sudo apt update --error-on=any &&
sudo apt install -y build-essential git python3 nodejs npm \
  pkg-config ninja-build unzip xz-utils

python3 --version
node --version
npm --version
```

安装 Linux 版 NDK r29（如果已安装，跳过下载并使用实际路径）：

```bash
mkdir -p "$HOME/android"
cd "$HOME/android"
wget -c https://dl.google.com/android/repository/android-ndk-r29-linux.zip
echo '87e2bb7e9be5d6a1c6cdf5ec40dd4e0c6d07c30b  android-ndk-r29-linux.zip' | sha1sum -c - &&
unzip -q android-ndk-r29-linux.zip
export ANDROID_NDK_ROOT="$HOME/android/android-ndk-r29"
cat "$ANDROID_NDK_ROOT/source.properties"
```

该下载约 784 MB。也可以在其他电脑下载这个 **Linux 版** ZIP，再复制进
虚拟机解压。Linux 不能使用 Windows 版 NDK。

配置并编译：

```bash
cd "$HOME/nosuke-src/subprojects/frida-core"
export ANDROID_NDK_ROOT="$HOME/android/android-ndk-r29"
export FRIDA_DEPS="$HOME/.cache/nosuke-frida-deps-20260311"
set -o pipefail

bash ./configure --host=android-arm64 \
  --enable-server --enable-gadget \
  --disable-portal --disable-inject --disable-tests \
  --disable-compiler-backend \
  --with-compat=native 2>&1 | tee configure.log &&
make 2>&1 | tee build.log
```

默认下载匹配版本的 Frida 工具链（包含专用 Vala 编译器）、SDK、Gum 和
JavaScript 构建依赖，需要访问 GitHub、Frida 依赖下载站点和 npm。
`FRIDA_DEPS` 使用新的缓存目录，避免复用这份拷贝中版本不一致的 `deps/`。
配置错误时先看 `configure.log`；编译错误时看 `build.log`。

如果复制后遇到脚本 `Permission denied`，在 frida-core 根目录恢复 Git
记录为可执行的文件权限（不覆盖文件内容）：

```bash
python3 - <<'PY'
from pathlib import Path
import os
import subprocess

for repository in (Path('.'), Path('releng'), Path('releng/meson'), Path('releng/tomlkit')):
    output = subprocess.check_output(['git', '-C', str(repository), 'ls-files', '--stage', '-z'])
    for record in output.split(b'\0'):
        if not record:
            continue
        metadata, raw_path = record.split(b'\t', 1)
        if metadata.split()[0] != b'100755':
            continue
        path = repository / os.fsdecode(raw_path)
        if path.is_file() and not path.is_symlink():
            path.chmod(path.stat().st_mode | 0o111)
PY
```

成功后的主要产物：

```text
build/server/nosuke-server
build/lib/gadget/frida-gadget.so
```

这里 Gadget 最终文件名没有 `lib` 前缀。Android 二进制不能直接在 x86_64
Kali 上运行。将它们放到相应架构的 Android 设备上验证。

`--with-compat=native` 会额外构建 ARM 32 位支持。若只需要 64 位进程，
可在第一次配置时改为 `--with-compat=disabled`。默认关闭的是设备端
Compiler backend，仍然支持加载 JavaScript hook；TypeScript 可在电脑端
预先编译。需要设备端编译接口时，应安装匹配的 Go 工具链并启用该 backend。

如果已配置过且需要改参数，先将旧 `build/` 改名备份，再重新配置。

## GitHub Actions 云端编译

本目录提供 `.github/workflows/android-build.yml`，在 GitHub 的 Ubuntu
runner 上安装 NDK r29 并交叉编译。不需要在本机安装编译工具。

1. 以 **frida-core 本身**作为 GitHub 仓库根目录：仓库顶层应看到
   `configure`、`meson.build`、`.gitmodules`、`releng` 和 `.github`。
   不要只把外层 `nosuke-src` 当作新仓库后直接 `git add`：内层是已有的
   Git 仓库，可能只记录一个 gitlink，云端拿不到实际修改源码。
2. 保留修改过的源码、Git 历史和版本标签，将其提交并推送到自己的仓库。
   `releng` 是子模块，保持 `.gitmodules` 及其固定提交。`deps/`、`build/`
   不应上传。不要重新克隆官方 frida-core 来代替自己的修改版。
3. 本目录原有 `CI` 工作流会在推送时运行多个平台。首次推送前可在独立的
   上传副本中移走 `.github/workflows/ci.yml`，或在仓库 Actions 页面禁用
   原有 `CI`，避免无关构建。保留新增的 Android 工作流。
4. 确保新工作流已进入仓库默认分支，然后打开 **Actions → Build Android
   Server and Gadget → Run workflow**。一般选择 `arch=arm64`、
   `compat=native`。
5. 构建完成后打开这次运行，在 **Artifacts** 下载 `nosuke-android-arm64`。
   解开 GitHub 下载的 ZIP，再解开其中的 `.tar.gz`，得到服务器、Gadget
   和 `BUILD-INFO.txt`。tar 包会保留可执行权限。
6. 失败时查看运行日志，或下载 `build-logs-android-arm64`。

工作流仅由手动操作触发，不会发布 Release。公开仓库的标准托管 runner
通常免费；私有仓库消耗账号套餐的 Actions 分钟和存储额度，具体以 GitHub
账单页面为准。Codespaces 和云服务器也可编译；若只是生成二进制，Actions
通常更方便。

此配置已做静态校验，尚未在 GitHub runner 上实际完成编译；源码修改与
依赖的兼容性仍需实际构建及 Android 设备验证。
