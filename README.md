# Deepin screenshot

## Description

This is a default screenshot app for Linux Deepin.

## Dependencies

- Qt (>= 5.15),
- cmake, qtbase5-dev, libqt5x11extras5-dev, qttools5-dev-tools,
- libdtkwidget-dev (>= 5.5, pulls in libdtkcore-dev / libdtkgui-dev)

## Installation

- On the Deepin operation system: the deepin screenshot is published within!
- On other operation system, you could download the source code of deepin screenshot:
    - firstly, install the dependencies rightly;
    - secondly, run `cd deepin-screenshot`;
    - thirdly, make a directory: build; run `cmake ../`; `make`; `make install`

## Deb packages (UOS / deepin)

Releases are built on GitHub Actions with native runners (amd64 + arm64) inside
a Debian bullseye container (glibc 2.31, matching the Kylin V20 target), then
packaged manually with `dpkg-deb` without strict `Depends` — a UOS/deepin
desktop already provides the Qt5/DTK runtime. Install with:

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

脚本逻辑（版本感知）：逐个检查 `deps/` 里的依赖包，**系统已安装且版本不低的直接用系统的；系统缺失或版本更旧的，用包内副本安装/升级**（合并成一次 `dpkg -i` 调用，由 dpkg 按依赖顺序配置；Qt5 小版本间 ABI 兼容，升级不影响系统里其他 Qt 应用）。`deps/` 是构建时按 Depends 递归展开的完整闭包——麒麟 V20（Ubuntu 20.04 基座）与 Debian 对部分库的包名不同（如 libicu63/libicu66），不同名的库可以共存，逐层展开才能保证离线 `dpkg -i` 不缺依赖。执行完再做一次 `ldd` 校验，明确报告"全部就绪"或仍缺失的库。

### 常见问题：运行报 `error while loading shared libraries: libdtkwidget.so.5`

这是目标机上没有 Qt5/DTK 运行库，说明装的是**常规包**，或装了离线包但依赖安装没有执行。处理：

1. 确认装的是 `-offline` 后缀的离线包，且架构与机器一致（`linux-aarch64` 对应 ARM64）；
2. 执行一次 `sudo /usr/share/deepin-screenshot/install-deps.sh`，看到 `OK: all runtime libraries resolved` 即可正常运行；
3. 若脚本报 `WARNING: these libraries are still missing`，把输出发回来排查。

注意 glibc 边界仍然适用：bullseye 构建产物要求目标机 glibc ≥ 2.31，glibc 更老的系统装了依赖库也无法运行。

两个硬边界：

- **glibc 不能打包替换**。所有产物在 Debian bullseye 容器（glibc 2.31）构建，要求目标机 glibc ≥ 2.31——麒麟 V20 / 优麒麟（Ubuntu 20.04 基座，glibc 2.31-0kylin）✓、UOS/deepin 桌面 ✓、Debian 11+ ✓；老 UOS 20（glibc 2.28）不在支持范围，需要目标机自编译或换更老的基础镜像（Qt/DTK 版本会相应变老，API 未必兼容本项目）。
- **postinst 里不能嵌套 `dpkg -i`**。dpkg 全程持有锁，维护脚本执行期间再调 dpkg 会报锁冲突，所以 postinst 只做检测，通过 systemd 延迟执行安装脚本（等同手动执行），systemd 不可用时提示手动运行。

## 构建踩坑记录（aarch64 deb）

在 bullseye 容器 + GitHub 原生 arm64 runner 上构建 deepin-screenshot 时遇到的问题与解决方法，供后续同类型 deepin 产品参考：

1. **glibc 基线必须匹配目标机**。最初用 bookworm（glibc 2.36）容器构建，产物引用 GLIBC_2.34+ 符号，在麒麟 V20（Ubuntu 20.04 基座，glibc 2.31-0kylin）上即使装齐依赖也起不来。解决：容器换成 bullseye（glibc 2.31，Qt 5.15 + DTK 5.2 齐全），构建产物向下兼容一切 ≥ 2.31 的系统。选容器先问"目标机 glibc 多少"。
2. **EOL 基线的 apt 源要留意时效**。截至 2026-10，bullseye 仍在 deb.debian.org 上正常服务，直接用镜像默认源即可。日后它移入 archive.debian.org 时，需把 apt 源改指 archive 并关闭 `Acquire::Check-Valid-Until`；但 archive 对云厂商 runner IP 限流严重（本次 arm64 构建连续在 apt 步骤失败就是这个原因），届时优先找一个仍在同步 debian-archive 的镜像站。
3. **包名随 Debian 版本变化**。如 `libxcb-util0-dev`（bullseye）→ `libxcb-util-dev`（bookworm），安装步骤按 `apt-cache show` 探测兼容两种命名。
4. **deepin 专用库已从 Debian 移除**。`dscreenwindowsutil.h`（`DScreenWindowsUtil`，多屏窗口选择）来自 dtkwm，bullseye 之后的 Debian 已无此包；且它的 qmake 构建依赖 dtkcore 的 `dtk_build.prf` 等 feature 文件，Debian 的 `libdtkcore-dev` 同样不提供（deepin 后期已转 cmake）。解决：CI 里用 `g++` + `moc` 直接编译 dtkwm 2.0.12（源码在 `martyr-deepin/dtkwm`，只有 2 个 cpp），产出的 `libdtkwm.so.2` 打进 deb 的 `/usr/lib/<三元组>/deepin-screenshot/` 私有目录，主程序设置 `INSTALL_RPATH` 指向该目录。CMakeLists 在 `find_package(DtkWm)` 失败时回退到 `find_path`/`find_library`。
5. **Qt 5.15 头文件包含更严格**。`QPainterPath` 不再被其他 Qt 头间接带入，使用处需显式 `#include <QPainterPath>`。
6. **deb 版本号必须以数字开头**。非 tag 构建 `git describe` 返回纯 git SHA，`dpkg-deb` 直接拒绝；统一加 `0.0.0-` 前缀兜底。
7. **arm64 用原生 runner，不做交叉编译**。`ubuntu-24.04-arm` 对公共仓库免费，容器内直接原生构建，避免交叉工具链和 sysroot 的所有坑。唯一要注意：debian-slim 容器里的 pkg-config 不会自动搜索多架构目录，需要手动设 `PKG_CONFIG_PATH=/usr/lib/aarch64-linux-gnu/pkgconfig:/usr/lib/pkgconfig:/usr/share/pkgconfig`。
8. **lrelease 在 cmake configure 阶段被调用**（`generate_translations.sh`），必须先装 `qttools5-dev-tools`，否则 Release 构建会静默丢失翻译。
9. **匿名身份读不到 Actions 日志**（API 返回 403）。调试办法：让工作流用自带的 `GITHUB_TOKEN` 把 `*.log` 推到临时分支（如 `ci-logs`），CI 结束后匿名可读，定位完把步骤和分支一起删掉。
10. **dpkg 全程持锁** → `postinst` 无法嵌套 `dpkg -i`（见"离线环境安装"）。
11. **离线依赖闭包必须递归展开 + 版本感知安装**。目标发行版对同一库可能用不同包名（麒麟/focal 的 libicu66 对 Debian 的 libicu63），只按 `ldd` 收一层依赖在离线机上 `dpkg -i` 会因 Depends 缺名而失败——构建时按 Depends 逐轮展开成完整闭包（不同名的库共存不冲突）；安装时按版本比较，系统库较新则保留、较旧或缺失才装包内的（Qt5 小版本间 ABI 兼容，升级安全）。

## 新产品接入清单（deepin 同类型产品）

复制 `.github/workflows/release.yml` 后按产品调整：

- 顶层 `name`、打包步骤里的 `Package`/`Description`、发布说明文本、产物文件名前缀
- "Build" 步骤换成项目自己的构建命令（cmake/qmake/meson 均可，安装前缀统一 `/usr`）
- "Install build dependencies" 换成项目真实的构建依赖；先在 packages.debian.org 按容器基线（当前 bullseye）确认是否有对应 `-dev` 包，没有的走踩坑记录第 4 条的自编译方案
- 项目若也需要打包额外库，参照 "Build dtkwm from source" 与 "Build offline deb package" 两个步骤的模式
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
