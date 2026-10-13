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
a Debian bookworm container, then packaged manually with `dpkg-deb` without
strict `Depends` — a UOS/deepin desktop already provides the Qt5/DTK runtime.
Install with:

    sudo dpkg -i deepin-screenshot_*.deb

To build a deb by hand:

    cmake -B build -DCMAKE_BUILD_TYPE=Release -DCMAKE_INSTALL_PREFIX=/usr
    make -C build -j$(nproc)
    make -C build install DESTDIR="$PWD/staging"
    # copy staging/. into pkg/deepin-screenshot_<version>_<arch>/ with a DEBIAN/control,
    # then: dpkg-deb --build --root-owner-group pkg/deepin-screenshot_<version>_<arch>

Pushing a `v*` tag (e.g. `v4.2.2`) builds both architectures and publishes a
GitHub release with the deb and tarball assets.

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
