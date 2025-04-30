# How To Build Perfetto Components



## Perfetto UI

**The UI module must be compiled on Linux.**



### Config Environments

```
sudo apt-get install build-essential libc6-dev curl
```



### Build Commands

```
tools/install-build-deps --ui
ui/build
// run http://localhost:10000
ui/run-dev-server
```



***



## Windows Perfetto Executable File

### Config Environment

#### LLVM

[**Install llvm from github.**](https://github.com/llvm/llvm-project/releases)

If you can execute this command blow, it means that LLVM has been successfully installed.

```
C:\Users\chao.xu5>clang --version
clang version 19.1.6
Target: x86_64-pc-windows-msvc
Thread model: posix
InstalledDir: C:\Program Files\LLVM\bin
```

#### Build Tools For Visual Studio

## Build tools for visual studio

1. Select C++ desktop development.
2. You must select msvc and MFC.
3. You can install Windows SDK online. (Optional)


#### Windows SDK

https://developer.microsoft.com/en-us/windows/downloads/windows-sdk/

You can download Windows SDK from here or from Visual Studio 2022. 

#### Python3

You must install python3 from Microsoft Store. I don't know why the Python environment variables I configured myself cannot be recognized.

#### Build Commands
```
python tools/install-build-deps
python tools/gn gen out/win
// add below content to out/win/args.gn
is_debug = false
is_clang = true
python tools/ninja -C out/win perfetto traced trace_processor_shell traceconv
```

453a8a40196bea0033d205a6d0c1a0075aee65b1