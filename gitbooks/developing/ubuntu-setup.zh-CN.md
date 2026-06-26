---
description: 在 Ubuntu/Debian 上安装 OpenHuman（稳定版 AppImage/deb）或从源码编译打包 .deb。
icon: ubuntu
lang: zh-CN
---

# Ubuntu 安装与编译

面向 **Ubuntu / Debian**（22.04 / 24.04）。跨平台说明见[环境搭建](getting-set-up.zh-CN.md)，仅 Rust 核心见[构建 Rust 核心](building-rust-core.zh-CN.md)。

| 架构 | 稳定版 | 源码 deb |
|------|--------|----------|
| x86_64 | AppImage / `.deb` | 默认 target |
| aarch64 | AppImage / `.deb` | `--target aarch64-unknown-linux-gnu` |

---

## 路径一：安装稳定版

**AppImage（推荐，无需 sudo）**

```bash
curl -fsSL https://raw.githubusercontent.com/tinyhumansai/openhuman/main/scripts/install.sh | bash
```

AppImage 启动异常时，改用 [GitHub Releases](https://github.com/tinyhumansai/openhuman/releases/latest) 的 `.deb`：

```bash
sudo dpkg -i OpenHuman_*_amd64.deb
sudo apt-get install -f
```

覆盖升级：重新运行 `install.sh` 或 `sudo dpkg -i` 新版本。**用户数据在 `~/.openhuman`，不在安装包内**（见下文）。

---

## 路径二：从源码编译 deb

> 以下均在**仓库根目录**执行。`pnpm --filter openhuman-app exec` 会在 `app/` 下跑 `cargo tauri`，无需 `cd app`。

### 1. 系统依赖

```bash
sudo apt-get update
sudo apt-get install -y git curl build-essential cmake pkg-config \
  libgtk-3-dev libwebkit2gtk-4.1-dev libayatana-appindicator3-dev librsvg2-dev \
  patchelf libasound2-dev libxdo-dev libxtst-dev libx11-dev libxi-dev \
  libevdev-dev libssl-dev libclang-dev desktop-file-utils \
  libnss3 libnspr4 libatk1.0-0 libatk-bridge2.0-0 libcups2 libdrm2 \
  libxkbcommon0 libxcomposite1 libxdamage1 libxfixes3 libxrandr2 \
  libgbm1 libpango-1.0-0 libcairo2 libatspi2.0-0 libxshmfence1 libu2f-udev
```

### 2. Node 24 + pnpm 10.10.0

**国内推荐：npmmirror 二进制**（勿用 Node v22）

```bash
cd /tmp && NODE_VERSION=v24.11.0
curl -L "https://npmmirror.com/mirrors/node/${NODE_VERSION}/node-${NODE_VERSION}-linux-x64.tar.xz" -o node.tar.xz
mkdir -p ~/.local/node && tar -xf node.tar.xz -C ~/.local/node --strip-components=1
echo 'export PATH="$HOME/.local/node/bin:$PATH"' >> ~/.zshrc && source ~/.zshrc

which node && node -v   # 必须 v24.x.x
corepack enable && corepack prepare pnpm@10.10.0 --activate
```

arm64 将 URL 中 `linux-x64` 改为 `linux-arm64`。也可用 NodeSource / nvm 安装 Node 24，版本满足即可。

### 3. Rust 1.93.0

**国内推荐：rsproxy**

```bash
export RUSTUP_DIST_SERVER=https://rsproxy.cn RUSTUP_UPDATE_ROOT=https://rsproxy.cn/rustup
curl -L https://rsproxy.cn/rustup-init.sh -o /tmp/rustup-init.sh && bash /tmp/rustup-init.sh -y
source "$HOME/.cargo/env"
rustup toolchain install 1.93.0 --profile minimal && rustup default 1.93.0
rustup component add rustfmt clippy --toolchain 1.93.0
```

境外可用 `https://sh.rustup.rs`。建议把 `RUSTUP_*` 写入 `~/.zshrc`。

### 4. 克隆与子模块

两个子模块**缺一不可**：`vendor/tauri-cef`、`vendor/tauri-plugin-notification`。

```bash
git clone --recursive https://github.com/tinyhumansai/openhuman.git
cd openhuman
# 已 clone 未 --recursive 时：
git submodule update --init --recursive
```

检出后检查；若 notification 目录为空，`update` 无输出时加 **`--force`**：

```bash
test -f app/src-tauri/vendor/tauri-plugin-notification/Cargo.toml && echo OK \
  || git submodule update --init --force app/src-tauri/vendor/tauri-plugin-notification
```

### 5. 依赖、前端验证、打包

```bash
cd /path/to/openhuman
node -v   # v24.x.x

pnpm install
pnpm --filter openhuman-app tauri:ensure

export PATH="$(pwd)/.cache/cargo-install/bin:$HOME/.cargo/bin:$PATH"
export CEF_PATH="$HOME/.cache/tauri-cef"
export NODE_OPTIONS="--max-old-space-size=8192"

pnpm --filter openhuman-app exec cargo tauri --version   # 秒退说明 PATH 未设
pnpm --filter openhuman-app run build:app                # 先过前端再打包
pnpm --filter openhuman-app exec cargo tauri build --bundles deb -- --bin OpenHuman
```

- 曾用 Node 22 装过依赖：`rm -rf node_modules app/node_modules && pnpm install`
- Debug：`--debug`；AppImage：加 `--bundles deb appimage`
- ARM64：build 命令加 `--target aarch64-unknown-linux-gnu`
- 可选 env：`cp .env.example .env && cp app/.env.example app/.env.local && source scripts/load-dotenv.sh`

### 6. 安装 deb

```bash
sudo dpkg -i app/src-tauri/target/*/release/bundle/deb/OpenHuman_*.deb
sudo apt-get install -f
```

### 日常重新打包

```bash
git pull && git submodule update --init --recursive
pnpm install && pnpm --filter openhuman-app tauri:ensure
export PATH="$(pwd)/.cache/cargo-install/bin:$HOME/.cargo/bin:$PATH"
export CEF_PATH="$HOME/.cache/tauri-cef" NODE_OPTIONS="--max-old-space-size=8192"
pnpm --filter openhuman-app run build:app
pnpm --filter openhuman-app exec cargo tauri build --bundles deb -- --bin OpenHuman
sudo dpkg -i app/src-tauri/target/*/release/bundle/deb/OpenHuman_*.deb
```

---

## 用户数据

| 内容 | 路径 |
|------|------|
| 数据根目录 | `~/.openhuman` |
| 聊天记录 | `~/.openhuman/workspace/memory/conversations/` |
| 配置 | `~/.openhuman/config.toml` |

覆盖安装 deb **不会**丢数据。会丢数据的操作：删 `~/.openhuman`、改 `OPENHUMAN_WORKSPACE`、切 staging 环境、设置里「重置本地数据」。备份：`cp -a ~/.openhuman ~/.openhuman.backup.$(date +%Y%m%d)`。

---

## 故障排除

| 现象 | 处理 |
|------|------|
| `no such command: tauri` / 命令秒退 | `pnpm --filter openhuman-app tauri:ensure` + `export PATH="$(pwd)/.cache/cargo-install/bin:$PATH"` |
| `build:app failed` / Node 22 / `@rolldown/binding-linux-x64-gnu` | 换 Node 24 → `rm -rf node_modules app/node_modules && pnpm install` → 重跑 `build:app` |
| Vite 已过、缺 `tauri-plugin-notification/Cargo.toml` | `git submodule update --init --force app/src-tauri/vendor/tauri-plugin-notification` |
| `libcef.so not found` | `pnpm --filter openhuman-app exec cargo build -p cef-dll-sys --release`，设 `LD_LIBRARY_PATH` 指向 `~/.cache/tauri-cef` 下含 `libcef.so` 的目录 |
| Node OOM | `export NODE_OPTIONS="--max-old-space-size=8192"` |
| 端口 7788 占用 | `pkill -f OpenHuman; pkill -f openhuman-core` |
| ARM 托盘 GTK 未初始化 | 见[环境搭建 — ARM Linux](getting-set-up.zh-CN.md#arm-linux-构建aarch64) |

本地开发（不打 deb）：`pnpm dev` / `pnpm dev:app`。CI 参考：`.github/workflows/build-desktop.yml`。
