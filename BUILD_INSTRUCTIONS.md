# Steps to build for vscode debugger

## macOS

First need to install emscripten and clone emsdk:

```shell
brew install emscripten
git clone https://github.com/emscripten-core/emsdk.git ~/emsdk
```

Init emsdk env and build with cmake:

```shell
source ~/emsdk/emsdk_env.sh
emcmake cmake -S . -B build
cd build
cmake --build . -j8
```

Copy files to plugin dir:

```shell
cp vAmiga.js ../../vscode-vamiga-debugger/vamiga/
cp vAmiga.wasm ../../vscode-vamiga-debugger/vamiga/
```

## Windows 11

**Prerequisites** — install via winget if not already present:

```cmd
winget install Ninja-build.Ninja
winget install Kitware.CMake
```

Python 3 is also required (https://www.python.org/downloads/).

**Install emsdk** (one-time setup):

```cmd
git clone https://github.com/emscripten-core/emsdk.git C:\emsdk
C:\emsdk\emsdk.bat install latest
C:\emsdk\emsdk.bat activate latest
```

**Configure** (one-time, from the repo root — open a fresh terminal after installing the prerequisites so PATH is up to date):

```cmd
set "EMSDK_QUIET=1" && call C:\emsdk\emsdk_env.bat && emcmake cmake -S . -B build -G Ninja
```

**Build:**

```cmd
set "EMSDK_QUIET=1" && call C:\emsdk\emsdk_env.bat && cmake --build build -j8
```

**Copy files to plugin dir:**

```cmd
copy /Y build\vAmiga.js ..\..\vscode-vamiga-debugger\vamiga\
copy /Y build\vAmiga.wasm ..\..\vscode-vamiga-debugger\vamiga\
```
