# Twilight Echo Linux/Fedora 适配

> 状态：进行中（基础链路已通，ALSA/FFmpeg 待系统依赖）
> 目标机器：Fedora 44 / x86_64 / dnf
> 对应分支：`feat/linux-fedora-adaptation`

## 现状

- 仓库已经包含 Linux 侧基础：ALSA 后端（`output/alsa`）、`after-pack-linux.cjs`、`linux` 的 electron-builder 目标（AppImage/deb/rpm/tar.gz）。
- 官方 README 明确：Linux/macOS 后端尚未达到发布级验证，需要本机适配。
- 关键产物路径（`after-pack-linux.cjs` 要求）：
  - `audio-engine/build/default/libtwilight-audio-engine.so`
  - `audio-engine/build/default/twilight_audio_node.node`

## 本机环境

```bash
cat /etc/os-release
# Fedora release 44 (Forty Four)
uname -r
# 7.1.5-201.fc44.x86_64
```

## 已安装工具链

```bash
node --version   # v22.22.2
pnpm --version   # 11.7.0（需使用可写 XDG 目录，见下）
cmake --version  # 4.3.0
ninja --version  # 1.13.2
g++ --version    # GCC 16.1.1
```

## 系统依赖（Fedora 需要安装）

```bash
sudo dnf install -y \
  cmake ninja-build gcc-c++ pkgconf-pkg-config \
  alsa-lib-devel ffmpeg-devel libebur128-devel
```

- `alsa-lib-devel`：启用 ALSA 后端
- `ffmpeg-devel`：解码大多数音频格式
- `libebur128-devel`：响度测量（缺失时使用内置 fallback，但建议安装）

## 适配步骤

### 1. 安装 JS 依赖

当前 home 目录在只读沙箱中，pnpm 需要可写的 XDG 目录：

```bash
export XDG_CACHE_HOME=/tmp/xdg-cache
export XDG_DATA_HOME=/tmp/xdg-data
export XDG_STATE_HOME=/tmp/xdg-state
export PNPM_HOME=/tmp/pnpm-home

cd Twilight_Echo
pnpm install --frozen-lockfile
```

### 2. 配置原生音频引擎（Linux/ALSA）

不通过 vcpkg，使用系统 FFmpeg/ALSA/ebur128，产物输出到 `audio-engine/build/default`：

```bash
pnpm run configure:audio-engine:linux
```

> 注意：`build/default` 与其他 preset 共用目录。若此前用 vcpkg 的 `default`
> preset 配置过，请先删除 `audio-engine/build/default` 再执行，避免残留
> `CMAKE_TOOLCHAIN_FILE`/`VCPKG_ROOT` 缓存。

### 3. 构建并测试

```bash
pnpm run build:audio-engine
pnpm run test:audio-engine:linux
```

### 4. 暂存原生库到应用资源

```bash
pnpm run stage:audio-engine
```

### 5. 前端类型检查 / 构建 / 运行

```bash
pnpm run typecheck
pnpm run build
pnpm run dev
```

## 预期要处理的差异点

- `default` CMake preset 目前依赖 `VCPKG_ROOT`；本机适配应使用“系统库”配置而不是 vcpkg。
- `after-pack-linux.cjs` 已就绪，但还没经过真实 Fedora 打包验证。
- ALSA 独占/bit-perfect：需用 `hw:` 设备做真实设备 smoke（`TAE_RUN_REAL_AUDIO_BACKEND_TESTS=1`）。
- Electron 43 在 Wayland 下的输入法降级逻辑已有，但需在 Fedora KDE/GNOME 实际验证。

## 验证进展（2026-08-31 实测）

- [x] `pnpm install --frozen-lockfile` 通过（39.3s，658 包）
- [x] `configure:audio-engine:linux` 配置成功（system libs 缺 dev 包时 ALSA/FFmpeg/ebur128 自动 OFF）
- [x] `cmake --build audio-engine/build/default` 构建成功，产物齐全（见上）
- [x] `pnpm run stage:audio-engine` 成功 → `resources/audio-engine/libtwilight-audio-engine.so`(18.9MiB) + `twilight_audio_node.node`(0.5MiB)
- [x] `pnpm run typecheck` 通过（node+web）
- [x] `pnpm run build` 通过（electron-vite 224+29+567 模块，renderer budgets 校验通过）
- [x] `pnpm run test:cue` 通过（44 条 CUE 用例全绿）
- [ ] `ctest` 全绿（当前 16/22；6 个 Not Run 因缺系统 dev 包：platform_backend_smoke / runtime_queue_reroute / audio_performance_gate / dst_decoder / coreaudio(无关) / alsa_backend）
- [ ] 安装系统依赖后重配重建启用 ALSA+FFmpeg，再跑 ctest 与真实设备 smoke
- [ ] `pnpm run dev` 启动并加载原生引擎
- [ ] ALSA `hw:` 独占/bit-perfect 输出验证
- [ ] `pnpm run build:linux` 生成 AppImage/rpm/deb

## 验证清单（后续逐项打勾）

- [ ] `pnpm install --frozen-lockfile` 通过
- [ ] `cmake -S audio-engine -B audio-engine/build/default ...` 通过
- [ ] `ctest --test-dir audio-engine/build/default` 全部通过
- [ ] `pnpm run stage:audio-engine` 生成 `resources/audio-engine/libtwilight-audio-engine.so` 与 `twilight_audio_node.node`
- [ ] `pnpm run dev` 能启动并加载原生音频引擎
- [ ] ALSA `hw:` 独占/bit-perfect 输出验证
- [ ] CUE 分轨、本地库扫描、主题功能在 Linux 下通过
- [ ] `pnpm run build:linux` 生成 AppImage/rpm/deb
