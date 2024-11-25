## N-API

Node-API（以前称为 N-API）是一个用于和 JavaScript 引擎交互并独立于 JavaScript 运行时的 API 接口层，旨在将 C 原生代码和 JavaScript 引擎隔离开来。API 通常用于创建和操作
JavaScript 值，概念和操作通常映射到 ECMA-262 语言规范，API 具有以下特点：

1. 所有节点API调用都返回类型为 NAPI{Common|Error|Exception}Status 的状态代码。此状态指示 API 调用是成功还是失败。
2. NAPICommonStatus 类型的接口一般情况下不需要检查返回值，NAPIErrorStatus 接口可能抛出内存分配失败错误，NAPIExceptionStatus 代表可能抛出 JavaScript 异常
3. API 的返回值通过 out 参数传递。
4. 所有 JavaScript 值都抽象在一个名为 NAPIValue 的不透明类型后面。
5. 如果出现 NAPIExceptionPendingException 说明出现 JS 异常，可以通过 napi_get_and_clear_last_exception 获取，或通过 NAPIClearLastException 清除

## 代码静态分析

### JavaScriptCore

1. `clang --analyze src/js_native_api_jsc.c --analyzer-output html -o ./static_analyze_report -I./include`

### QuickJS

1. `clang --analyze src/js_native_api_qjs.c --analyzer-output html -o ./static_analyze_report -I./include -I./third_party/quickjs`

### Hermes

1. `clang++ --analyze src/js_native_api_hermes.cpp --analyzer-output html -o ./static_analyze_report -I./include -I./third_party/hermes/include -I./third_party/hermes/public -DHERMESVM_GC_NONCONTIG_GENERATIONAL -DHERMESVM_ALLOW_COMPRESSED_POINTERS -DHERMES_ENABLE_DEBUGGER -std=c++17 -I./third_party/include -I./third_party/hermes/external/llvh/include -I./third_party/hermes/external/llvh/gen/include -I./third_party/hermes/API -I./third_party/hermes/API/jsi -I./third_party/react-native/ReactCommon`

## 单元测试

1. `gn gen out --args="debug=true asan=true ubsan=true"`（Release 模式下 assert 失效，会隐藏很多问题，特殊情况下启用 asan + ubsan）
2. `ninja -C out test_{qjs|jsc|hermes}`
3. `./out/test_{qjs|jsc|hermes}`

### QuickJS 单元测试修改源代码部分

1. ubsan/asan 对于 QuickJS 开启，需要注释掉 CONFIG_STACK_CHECK，否则会报告 InternalError: stack overflow
2. QuickJS 单元测试建议开启 DUMP_LEAKS

### Hermes 单元测试修改源代码部分

1. include/hermes/VM/HandleRootOwner.h 修改 HERMESVM_DEBUG_MAX_GCSCOPE_HANDLES 为 2^16-1 -> 65535

## 编辑器配置

1. 推荐使用 CLion
2. VSCode

### CLion

1. `gn gen clion --ide=json --json-ide-script=../gn_to_cmake.py --script-executable=python3`

### VSCode

1. C/C++ 插件
2. 自行配置头文件搜索路径

## 交叉编译

1. `rm -rf napi`
2. `mkdir napi`
3. `cp -r include napi`
4. `cp -r third_party/react-native/ReactCommon/cxxreact/MessageQueueThread.h include/napi`
5. `tar czvf napi.tar.gz -C napi .` 压缩产物

### iOS 静态库

1. `gn gen x86_64 --args="build_ios=true"`
2. `gn gen armv7 --args="build_ios=true cross_compile_target=\"armv7\""`
3. `gn gen arm64 --args="build_ios=true cross_compile_target=\"arm64\""`
4. `ninja -C x86_64 quickjs jsc && ninja -C armv7 quickjs jsc && ninja -C arm64 quickjs jsc`
5. `gn gen x86_64 --args="build_ios=true debug_info=false lto=false bitcode=false"`
6. `gn gen armv7 --args="build_ios=true cross_compile_target=\"armv7\" debug_info=false lto=false bitcode=false"`
7. `gn gen arm64 --args="build_ios=true cross_compile_target=\"arm64\" debug_info=false lto=false bitcode=false"`
8. `ninja -C x86_64 hermes && ninja -C armv7 hermes && ninja -C hermes`
9. `libtool -static x86_64/obj/libquickjs.a armv7/obj/libquickjs.a arm64/obj/libquickjs.a -o napi/libquickjs.a && libtool -static x86_64/obj/libhermes.a armv7/obj/libhermes.a arm64/obj/libhermes.a -o napi/libhermes.a && libtool -static x86_64/obj/libjsc.a armv7/obj/libjsc.a arm64/obj/libjsc.a -o napi/libjsc.a`

### Android 动态库

首先，需要将BUILDCONFIG.gn文件所配置的ndk_path修改为你本机上Andoird NDK安装路径

1. `gn gen armv7 --args="build_android=true cross_compile_target=\"armv7\""`
2. `gn gen arm64 --args="build_android=true cross_compile_target=\"arm64\""`
3. `gn gen i386 --args="build_android=true cross_compile_target=\"i386\""`
4. `gn gen x86_64 --args="build_android=true cross_compile_target=\"x86_64\""`
5. `ninja -C armv7 qjs hermes && ninja -C arm64 qjs hermes && ninja -C i386 qjs hermes && ninja -C x86_64 qjs hermes`
6. `mkdir -p napi/libs/armeabi-v7a && mkdir -p napi/libs/arm64-v8a && mkdir -p napi/libs/x86 && mkdir -p napi/libs/x86_64`
7. `cp armv7/obj/lib{hermes,qjs}.so napi/libs/armeabi-v7a && cp arm64/obj/lib{hermes,qjs}.so napi/libs/arm64-v8a && cp i386/obj/lib{hermes,qjs}.so napi/libs/x86 && cp x86_64/obj/lib{hermes,qjs}.so napi/libs/x86_64`

#### 交叉编译注意

1. macOS shell 对文件描述符有限制，默认限制 256，会导致链接出问题，可以通过 `ulimit -a` 查看，可以通过 `ulimit -S -n 4096` 临时修改

#### 注意

1. 建议使用 BUILDCONFIG.gn 中定义的 LTS NDK 版本
2. Hermes 引擎需要先 `cd third_party/hermes && git apply ../hermes_patch.diff`
3. Android 版本 libhermes.so 包括 fbjni 库，内含 OnLoad.cpp，需要使用 System.load("hermes") 显式加载，不能依赖 Linux 内核的动态库隐式加载
4. Hermes 引擎 0.8.1 版本内置字节码，需要先编译主机 hermesc，指令 `./utils/build/configure.py`，后输入如下命令
```
> cd lib/InternalBytecode
> cat 00-header.js 01-Promise.js 02-AsyncFn.js 99-footer.js > InternalBytecode.js
> ../../build/bin/hermesc -O -Wno-undefined-variable -fno-enable-tdz -emit-binary -out=./InternalBytecode.hbc ./InternalBytecode.js
> ./xxd.py ./InternalBytecode.hbc > ./InternalBytecode.inc
```

## 注意事项

1. 只能在单一线程执行，不跨多线程执行，JavaScriptCore 通过锁机制保证同步，但是其他引擎很可能没有该保证
