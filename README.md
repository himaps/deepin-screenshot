# Deepin screenshot

## Description

This is a default screenshot app for Linux Deepin.

## Dependencies

- Qt (>= 5.12) — build against the target distro's own Qt,
- cmake (>= 3.16), qtbase5-dev, libqt5x11extras5-dev, qttools5-dev-tools,
- DTK 5.2 (libdtkwidget-dev + libdtkcore-dev / libdtkgui-dev; the release
  workflow builds DTK 5.2 and dtkwm from source on distros without them)

## Installation

- On the Deepin operation system: the deepin screenshot is published within!
- On other operation system, you could download the source code of deepin screenshot:
    - firstly, install the dependencies rightly;
    - secondly, run `cd deepin-screenshot`;
    - thirdly, make a directory: build; run `cmake ../`; `make`; `make install`

## Deb packages (UOS / deepin)

Releases are built on GitHub Actions with native runners (amd64 + arm64) inside
an Ubuntu 20.04 container — the same distro base as the Kylin V20 target
(Qt 5.12.8, glibc 2.31, identical library package names) — then packaged
manually with `dpkg-deb` without strict `Depends`. Install with:

    sudo dpkg -i deepin-screenshot-*.deb

To build a deb by hand:

    cmake -B build -DCMAKE_BUILD_TYPE=Release -DCMAKE_INSTALL_PREFIX=/usr
    make -C build -j$(nproc)
    make -C build install DESTDIR="$PWD/staging"
    # copy staging/. into pkg/deepin-screenshot_<version>_<arch>/ with a DEBIAN/control,
    # then: dpkg-deb --build --root-owner-group pkg/deepin-screenshot_<version>_<arch>

Pushing a `v*` tag (e.g. `v4.2.2`) builds both architectures and publishes a
GitHub release with the deb and tarball assets.

## 离线环境安装（offline install）

Release 提供两个 deb 变体，按需二选一（文件名格式：`deepin-screenshot-<版本>-linux-<架构>[-offline].deb`，架构为 amd64 / aarch64）：

- `deepin-screenshot-<版本>-linux-<架构>.deb` — 常规包，适合已具备 Qt5/DTK 运行时的 UOS/deepin 桌面。
- `deepin-screenshot-<版本>-linux-<架构>-offline.deb` — 离线包，在常规内容之外额外打包了运行时依赖闭包（`deps/*.deb`）和安装脚本。

安装方式与常规包相同：

    sudo dpkg -i deepin-screenshot-<版本>-linux-<架构>-offline.deb

安装结束时 postinst 会用 `ldd` 检测缺失的运行库：**全部齐备则无事发生；有缺失则自动通过 systemd 延迟 10 秒执行依赖安装**（等 dpkg 锁释放，等同手动执行下述脚本，结果在 `journalctl -u deepin-screenshot-deps`）。systemd 不可用时回退为提示手动执行：

    sudo /usr/share/deepin-screenshot/install-deps.sh

脚本逻辑（只增不改）：逐个检查 `deps/` 里的依赖包，**系统已安装的一律保留系统副本，只安装缺失的**（合并成一次 `dpkg -i` 调用，由 dpkg 按依赖顺序配置）。`deps/` 是构建时按 Depends 递归展开的完整闭包，且在构建期就剔除了 glibc 与 systemd/dbus/PAM 等系统核心件——因为容器基线与麒麟同源（Ubuntu 20.04），目标机已有的库版本正好匹配，不需要也不会升级任何系统组件。执行完再做一次 `ldd` 校验，明确报告"全部就绪"或仍缺失的库。

### 常见问题：运行报 `error while loading shared libraries: libdtkwidget.so.5`

这是目标机上没有 Qt5/DTK 运行库，说明装的是**常规包**，或装了离线包但依赖安装没有执行。处理：

1. 确认装的是 `-offline` 后缀的离线包，且架构与机器一致（`linux-aarch64` 对应 ARM64）；
2. 执行一次 `sudo /usr/share/deepin-screenshot/install-deps.sh`，看到 `OK: all runtime libraries resolved` 即可正常运行；
3. 若脚本报 `WARNING: these libraries are still missing`，把输出发回来排查。

注意 glibc 边界仍然适用：产物要求目标机 glibc ≥ 2.31（focal 基线），glibc 更老的系统装了依赖库也无法运行。

两个硬边界：

- **glibc 不能打包替换，系统核心组件不能被第三方包升级**。所有产物在 Ubuntu 20.04 容器（glibc 2.31）构建——与麒麟 V20 / 优麒麟（glibc 2.31-0kylin）同源同版本。曾有版本用 Debian bullseye 容器构建 + "系统库旧则升级"策略，结果在麒麟上把 systemd 245 升成了 Debian 247，导致系统崩溃——这就是现在改为 focal 同源基线 + 只装缺失、构建期黑名单（systemd/dbus/PAM/udev/glibc 永不入包）三重防护的原因。老 UOS 20（glibc 2.28）不在支持范围，需要目标机自编译或换更老的基础镜像（Qt/DTK 版本会相应变老，API 未必兼容本项目）。
- **postinst 里不能嵌套 `dpkg -i`**。dpkg 全程持有锁，维护脚本执行期间再调 dpkg 会报锁冲突，所以 postinst 只做 `ldd` 检测，有缺失就通过 systemd 延迟 10 秒自动执行安装脚本（等同手动执行），systemd 不可用时提示手动运行。

## 构建踩坑记录（aarch64 deb）

在 Ubuntu 20.04 容器 + GitHub 原生 arm64 runner 上构建 deepin-screenshot 时遇到的问题与解决方法，供后续同类型 deepin 产品参考：

1. **构建基线必须与目标机同源，且第三方离线包永远不要升级系统组件**。第一版用 bookworm（glibc 2.36）构建，产物引用 GLIBC_2.34+ 符号，麒麟 V20 装齐依赖也起不来；第二版换 bullseye（glibc 2.31 + Qt 5.15）并用"系统库较旧则升级"策略，结果把麒麟的 systemd 245 升成了 Debian 247（含 /sbin/init），**直接把系统搞崩**。最终方案：容器换成与目标机同源的 Ubuntu 20.04（Qt 5.12.8 / glibc 2.31 / 包名一致），DTK 5.2 在 CI 里从源码构建，安装策略改为"只装缺失的，绝不升级已装的"，并在构建期把 glibc 和 systemd/dbus/PAM/udev 等核心件从闭包里剔除。教训：给整机 Linux 发第三方离线包，升级系统库的红线一次都不能碰。
2. **EOL 基线的 apt 源处于"半归档"状态**（bullseye 教训，选旧基线时会再遇到）。过 LTS 的发行版，镜像上 `bullseye-security` 这类套件的索引还在（`apt-get update` 正常），但 pool 里的 .deb 已被移走（安装时成片 404）。解决：只把失效套件指到 archive 镜像，main 保持快速通道；不要把整个源都切到 archive——它对云厂商 runner IP 限流严重，会造成更难排查的间歇性失败。
3. **包名随发行版版本变化**。如 `libxcb-util0-dev`（bullseye）→ `libxcb-util-dev`（bookworm），安装步骤按 `apt-cache show` 探测兼容两种命名。与目标机同源（focal 对麒麟）可从根上避开此类错配。
4. **deepin 专用库已从 Debian 移除**。`dscreenwindowsutil.h`（`DScreenWindowsUtil`，多屏窗口选择）来自 dtkwm，bullseye 之后的 Debian 已无此包；且它的 qmake 构建依赖 dtkcore 的 `dtk_build.prf` 等 feature 文件，Debian 的 `libdtkcore-dev` 同样不提供（deepin 后期已转 cmake）。解决：CI 里用 `g++` + `moc` 直接编译 dtkwm 2.0.12（源码在 `martyr-deepin/dtkwm`，只有 2 个 cpp），产出的 `libdtkwm.so.2` 打进 deb 的 `/usr/lib/<三元组>/deepin-screenshot/` 私有目录，主程序设置 `INSTALL_RPATH` 指向该目录。CMakeLists 在 `find_package(DtkWm)` 失败时回退到 `find_path`/`find_library`。
5. **Qt 5.15 头文件包含更严格**。`QPainterPath` 不再被其他 Qt 头间接带入，使用处需显式 `#include <QPainterPath>`。
6. **deb 版本号必须以数字开头**。非 tag 构建 `git describe` 返回纯 git SHA，`dpkg-deb` 直接拒绝；统一加 `0.0.0-` 前缀兜底。
7. **arm64 用原生 runner，不做交叉编译**。`ubuntu-24.04-arm` 对公共仓库免费，容器内直接原生构建，避免交叉工具链和 sysroot 的所有坑。唯一要注意：debian-slim 容器里的 pkg-config 不会自动搜索多架构目录，需要手动设 `PKG_CONFIG_PATH=/usr/lib/aarch64-linux-gnu/pkgconfig:/usr/lib/pkgconfig:/usr/share/pkgconfig`。
8. **lrelease 在 cmake configure 阶段被调用**（`generate_translations.sh`），必须先装 `qttools5-dev-tools`，否则 Release 构建会静默丢失翻译。
9. **匿名身份读不到 Actions 日志**（API 返回 403）。调试办法：让工作流用自带的 `GITHUB_TOKEN` 把 `*.log` 推到临时分支（如 `ci-logs`），CI 结束后匿名可读，定位完把步骤和分支一起删掉。
10. **dpkg 全程持锁** → `postinst` 无法嵌套 `dpkg -i`（见"离线环境安装"）。
11. **离线依赖闭包必须递归展开 + 只装缺失**。目标机可能缺某个间接依赖，只按 `ldd` 收一层在离线机上 `dpkg -i` 会因 Depends 缺名而失败——构建时按 Depends 逐轮展开成完整闭包；安装时只装系统缺失的包、绝不升级已装的（前提是构建基线与目标机同源，见第 1 条）。

## 新产品接入清单（deepin 同类型产品）

复制 `.github/workflows/release.yml` 后按产品调整：

- 顶层 `name`、打包步骤里的 `Package`/`Description`、发布说明文本、产物文件名前缀
- "Build" 步骤换成项目自己的构建命令（cmake/qmake/meson 均可，安装前缀统一 `/usr`）
- "Install build dependencies" 换成项目真实的构建依赖；基线与目标发行版同源，目标机自带 Qt 的就直接用它的版本构建，deepin 专用库（DTK 等）参照 "Build DTK 5.2 and dtkwm from source" 步骤从源码构建
- 推送 master 做双架构预检；打 `v*` tag 自动构建并发布 Release
- 离线包构建是通用的：它按 `ldd` 结果反查包名、下载依赖闭包，不绑定本项目，换产品只需改安装路径里的产品名

## Usage
Run the command: `deepin-screenshot`

## Getting help

Any usage issues can ask for help via

* [Official Forum](https://bbs.deepin.org/)
* [Developer Center](https://github.com/linuxdeepin/developer-center)
* [WiKi](https://wiki.deepin.org/)
* [Gitter](https://gitter.im/orgs/linuxdeepin/rooms)
* [IRC channel](https://webchat.freenode.net/?channels=deepin)

## Getting involved

We encourage you to report issues and contribute changes

* [Contribution guide for developers](https://github.com/linuxdeepin/developer-center/wiki/Contribution-Guidelines-for-Developers-en). (English)
* [开发者代码贡献指南](https://github.com/linuxdeepin/developer-center/wiki/Contribution-Guidelines-for-Developers) (中文)

## License

deepin-screenshot is licensed under [GPLv3](LICENSE).
