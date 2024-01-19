@[toc]
## 背景 

使用Go进行跨平台编译通常是直接的：设置GOOS和GOARCH环境变量，然后执行go build命令。

不幸的是，**对于使用CGO依赖的项目来说，事情可能会更复杂**。根据目标架构的不同，可能需要安装C编译器，如gcc、clang或x86_64-w64-mingw64-gcc，并配置额外的环境变量，如CC，以及启用CGO_ENABLED=1。

在这篇文章中，我们将看到如何使用Zig来跨编译Paw项目，针对Linux、macOS和Windows，无需安装额外的编译器。

Paw是一个用于安全地管理密码和身份的跨平台应用程序，它是用Go编写的，并使用Fyne工具包作为GUI，以及golang.design/x/clipboard库作为CLI，这些都有CGO依赖。

Zig是一个通用编程语言和工具链，它提供了一个零依赖的C/C++编译器，并且支持开箱即用的跨平台编译。

## 运行环境
* debian/ubuntu based distro
* go v1.18.0
* zig v0.9.1

## 下载源代码
```shell
git clone https://github.com/lucor/paw.git
```

## 1. 为Linux amd64目标编译

```shell
# install the GUI and CLI dependencies
apt-get update
apt-get install -y -q --no-install-recommends \
    libgl-dev \
    libx11-dev \
    libxrandr-dev \
    libxxf86vm-dev \
    libxi-dev \
    libxcursor-dev \
    libxinerama-dev

# create dist dir
mkdir -p dist/linux-amd64

# build
CGO_ENABLED=1 \
GOOS=linux \
GOARCH=amd64 \
CC="zig cc -target x86_64-linux-gnu -isystem /usr/include -L/usr/lib/x86_64-linux-gnu" \
CXX="zig c++ -target x86_64-linux-gnu -isystem /usr/include -L/usr/lib/x86_64-linux-gnu" \
go build -trimpath -o dist/linux-amd64 ./cmd/...
```

## 2. 为Linux arm64目标编译
```shell
# install the GUI and CLI dependencies
dpkg --add-architecture arm64
apt-get update
apt-get install -y -q --no-install-recommends \
    libgl-dev:arm64 \
    libx11-dev:arm64 \
    libxrandr-dev:arm64 \
    libxxf86vm-dev:arm64 \
    libxi-dev:arm64 \
    libxcursor-dev:arm64 \
    libxinerama-dev:arm64

# create dist dir
mkdir -p dist/linux-arm64

# build
CGO_ENABLED=1 \
GOOS=linux \
GOARCH=arm64 \
PKG_CONFIG_LIBDIR=/usr/lib/aarch64-linux-gnu/pkgconfig \
CC="zig cc -target aarch64-linux-gnu -isystem /usr/include -L/usr/lib/aarch64-linux-gnu" \
CXX="zig c++ -target aarch64-linux-gnu -isystem /usr/include -L/usr/lib/aarch64-linux-gnu" \
go build -trimpath -o dist/linux-arm64 ./cmd/...
```

## 3. 为Windows amd64目标编译
```shell
# create dist dir
mkdir -p dist/windows-amd64

# build
CGO_ENABLED=1 \
GOOS=windows \
GOARCH=amd64 \
CC="zig cc -target x86_64-windows-gnu" \
CXX="zig c++ -target x86_64-windows-gnu" \
go build -trimpath -ldflags='-H=windowsgui' -o dist/windows-amd64 ./cmd/...
```
## 4. 为macOS amd64目标编译
```shell
export MACOS_MIN_VER=10.14
export MACOS_SDK_PATH="/path/to/macOS/sdk"

# create dist dir
mkdir -p dist/darwin-amd64

# build
CGO_ENABLED=1 \
GOOS=darwin \
GOARCH=amd64 \
CGO_LDFLAGS="-mmacosx-version-min=${MACOS_MIN_VER} --sysroot ${MACOS_SDK_PATH} -F/System/Library/Frameworks -L/usr/lib" \
CC="zig cc -mmacosx-version-min=${MACOS_MIN_VER} -target x86_64-macos-gnu -isysroot ${MACOS_SDK_PATH} -iwithsysroot /usr/include -iframeworkwithsysroot /System/Library/Frameworks" \
CXX="zig c++ -mmacosx-version-min=${MACOS_MIN_VER} -target x86_64-macos-gnu -isysroot ${MACOS_SDK_PATH} -iwithsysroot /usr/include -iframeworkwithsysroot /System/Library/Frameworks" \
go build -trimpath -buildmode=pie -o dist/darwin-amd64  ./cmd/...
```

## 5. 为macOS arm64目标编译
```shell
export MACOS_MIN_VER=11.1
export MACOS_SDK_PATH="/path/to/macOS/sdk"

# create dist dir
mkdir -p dist/darwin-arm64

# build
CGO_ENABLED=1 \
GOOS=darwin \
GOARCH=arm64 \
CGO_LDFLAGS="-mmacosx-version-min=${MACOS_MIN_VER} --sysroot ${MACOS_SDK_PATH} -F/System/Library/Frameworks -L/usr/lib" \
CC="zig cc -mmacosx-version-min=${MACOS_MIN_VER} -target aarch64-macos-gnu -isysroot ${MACOS_SDK_PATH} -iwithsysroot /usr/include -iframeworkwithsysroot /System/Library/Frameworks" \
CXX="zig c++ -mmacosx-version-min=${MACOS_MIN_VER} -target aarch64-macos-gnu -isysroot ${MACOS_SDK_PATH} -iwithsysroot /usr/include -iframeworkwithsysroot /System/Library/Frameworks" \
go build -ldflags "-s -w" -buildmode=pie -trimpath -o dist/darwin-arm64 ./cmd/...
```

## 参考内容
* https://dev.to/kristoff/zig-makes-go-cross-compilation-just-work-29ho
* https://github.com/ziglang/zig/issues/9050#issuecomment-859939664
