---
title: 编译Chromium源码for Android
date: 2019-09-29 19:20:47
categories: Chromium
tags: [Android, Chromium]
---

## 编译环境
在Windows或者Mac下编译Android客户端是不支持的，官方推荐的是使用Ubuntu来进行编译，因此我们的编译采用的是Ubuntu服务器，版本为`14.04.2 LTS (GNU/Linux 3.16.0-30-generic x86_64)`。需提前安装Git（拉代码）和Python（GN中的所有外部脚本都在Python中执行）

## 安装depot_tools
Chromium使用`depot_tools`的脚本包来管理checkout和审查代码，因此拉代码前需要安装好此工具。首先拉取代码：
```bash
git clone https://chromium.googlesource.com/chromium/tools/depot_tools.git
```
添加路径到PATH环境变量中，或者直接添加到配置文件：`~/.bashrc` 或 `~/.zshrc`
```bash
export PATH="$PATH:/path/to/depot_tools"
```

## 拉取Chromium源码
新建目录然后拉取代码，这里过程非常长，看大家网速了。如果想快点拉完，可以在fetch的时候添加`--no-history`指令不拉取历史记录，这样会快很多。
```bash
mkdir ~/chromium && cd ~/chromium
fetch --nohooks android
```
>PS：通过`fetch android`指令拉取代码，会自动帮我们在`.gclient`文件下配置好`target_os = [ 'android' ]`属性，并自动执行`gclient sync`指令去拉取Android相关的依赖。

## 下载Android依赖
拉完源码后，执行以下指令安装编译Android APK所需的依赖及下载其他所需的二进制文件。
```
build/install-build-deps-android.sh
gclient runhooks
```

## 配置GN编译脚本
Chromium使用[Ninja][1]作为主要的构建工具，并通过[GN][2]来生成`.ninja`配置文件。对于Chromium这种航母级的源码编译，使用Ninja能大大的加快编译速度。通过`gn args out/Default`指令，会启动系统的默认编辑器进入配置文件编辑模式，并在`out/Default`目录下初始化此版本的编译，生成`args.gn`文件（可以同时配置多个构建版本，生成的产物互不影响）。这个时候输入以下的配置参数并保存，GN会帮我们初始化好所有的`.ninja`配置文件。
```bash
target_os = "android"
target_cpu = "arm"
```
当然你也可以使用以下指令快速完成配置文件的初始化：
```bash
gn gen --args='target_os="android" target_cpu="arm"' out/Default
```

关于`target_cpu`与本机代码指令集的对应关系，可以参照下列表格：

|  abi   | target_cpu |
| :----: | :----: |
| arm64-v8a | arm64 |
| armeabi-v7a | arm |
| x86 | x86 |
| x86_64 | x64 |

除了`target_os`和`target_cpu`之外，我们还可以根据编译类型（如debug、release）设置一些其他的参数：

- is_debug: 设置true时编译debug版本，false时编译release版本。
- is_component_build: 设置true时会把声明为`components`的目标作为共享库动态加载，一般在编译debug版本时，会设置为true，这样每次改动编译链接花费的时间就会减少很多。若为false，则组件将采用静态链接，详细介绍可参考官方文档：[component_build.md][3]。
- use_jumbo_build: 设置true时会启动jumbo编译模式，该模式下编译与链接速度都会快很多，在调试的时候可以提高效率。但是因为jumbo的原理是合并多文件一起进行编译，因此编译时可能会产生额外的symbols冲突，详细介绍可参考官方文档：[jumbo.md][4]。

## GN常用指令

### 查看构建环境的默认参数
通过`args --list`命令还可以查看当前构建环境的所有配置参数与详细说明，并且可以看到当前的默认值是多少，通过以下指令我们可以把参数输出到文件中进行查看。
```bash
gn args --list out/Default > env.txt
```

### 对历史构建进行清理
通过`clean`指令可以对编译环境进行清理，会删除所有编译的中间产物，仅保留`args.gn`、`build.ninja`等基础的配置文件，以便我们进行一次完整的重编译。
```bash
gn clean out/Default
```

### 查看给定target或config的详细信息
通过`desc`指令可以查看某个target的所有详细信息，包括所有包含的`sources`源码、`cflags`参数、依赖的`libs`等等...通过以下指令我们可以把结果输出到文件中进行查看。
```bash
gn desc out/Default //net:net > desc.txt
```

### 查看指令帮助信息
通过`help`指令可快速查看某个`command`的详细使用文档，包括支持参数、输出结果、Examples等：
```bash
gn help <command>
```

关于GN的更多使用方式可参考官方文档：[Quick Start guide][5]

## 编译Chromium APK
通过以下命令我们可以编译Chromium的APK：
```bash
autoninja -C out/Default chrome_public_apk
```
其中`autoninja`为`ninja`的包装器，会自动提供最佳的参数给到`ninja`；`out/Default`为我们生成配置文件的目录；最后的`chrome_public_apk`为`ninja`已经配置好的编译target，其中还包含了`chrome_modern_public_apk`、`monochrome_public_apk`等特性target。

经过漫长的等待后，看到下面的log，并且没有任何报错，就证明已经编译完成了，APK产物`ChromePublic.apk`在`out/Default/apks`目录下面：
```bash
ninja: Entering directory `out/Default'
[40164/40164] STAMP obj/chrome/android/chrome_public_apk.stamp
```

## 单独编译模块与使用
除了编译完整的APK，其实在`Chromium`中还有很多实用的模块与so库我们可以抽出来在项目中进行使用。单独编译模块的方式也很简单，假设我们需要编译net模块到项目中使用，可通过如下的命令进行编译：
```bash
autoninja -C out/Default net
```

此命令会编译net模块以及所依赖的其他模块，如：`base`、`boringssl`、`crcrypto`等模块。顺利编译完成后，在`out/Default/`目录下即可找到我们所需的so库文件：
```
ls -l | grep so$
-rwxr-xr-x  1 codezjx codezjx  1871676 Sep 30 11:18 libbase.cr.so
-rwxr-xr-x  1 codezjx codezjx   286620 Sep 30 11:18 libbase_i18n.cr.so
-rwxr-xr-x  1 codezjx codezjx   966296 Sep 30 11:18 libboringssl.cr.so
-rwxr-xr-x  1 codezjx codezjx   464328 Sep 30 11:18 libc++.cr.so
-rwxr-xr-x  1 codezjx codezjx    63448 Sep 30 11:18 libchrome_zlib.cr.so
-rwxr-xr-x  1 codezjx codezjx    53208 Sep 30 11:18 libcrcrypto.cr.so
-rwxr-xr-x  1 codezjx codezjx  1609404 Sep 30 11:18 libicui18n.cr.so
-rwxr-xr-x  1 codezjx codezjx  1092640 Sep 30 11:18 libicuuc.cr.so
-rwxr-xr-x  1 codezjx codezjx  5965140 Sep 30 11:19 libnet.cr.so
-rwxr-xr-x  1 codezjx codezjx   313324 Sep 30 11:18 libprotobuf_lite.cr.so
-rwxr-xr-x  1 codezjx codezjx   104628 Sep 30 11:18 liburl.cr.so
```

将上面编译生成的so拷贝到项目的`src/main/jniLibs/armeabi-v7a`目录下，并抽取相关头文件拷贝至`src/main/cpp/include`中（可通过`gn desc`中返回的相关信息进行查找）。最终在`CMakeLists.txt`中设置`include_directories`与`link_directories`后即可进行调用。
```
set(libs_dir src/main/jniLibs/${ANDROID_ABI})
set(libs_include_dir src/main/cpp/include)
include_directories(${libs_include_dir})
link_directories(${libs_dir})
```

[1]: https://ninja-build.org/
[2]: https://gn.googlesource.com/gn/+/master/docs/quick_start.md
[3]: https://chromium.googlesource.com/chromium/src/+/master/docs/component_build.md
[4]: https://chromium.googlesource.com/chromium/src/+/master/docs/jumbo.md
[5]: https://gn.googlesource.com/gn/+/master/docs/quick_start.md