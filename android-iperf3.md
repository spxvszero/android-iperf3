# android-iperf3

> iPerf3 is a tool for active measurements of the maximum achievable bandwidth on IP networks. It supports tuning of various parameters related to timing, buffers and protocols (TCP, UDP, SCTP with IPv4 and IPv6). For each test it reports the bandwidth, loss, and other parameters.
>
> iPerf3 是一种主动测量IP网络上可实现的最大带宽的工具。它支持调整与时序、缓冲区和协议（TCP、UDP、带有IPv4和IPv6的SCTP）相关的各种参数。对于每次测试，它都会报告带宽、损耗和其他参数。
>
> [GitHub - davidBar-On/android-iperf3: Pre-compiled iperf3 binaries for Android + Dockerfile with SDK and NDK for manual build](https://github.com/davidBar-On/android-iperf3/tree/master)

想要在 Android 平台上使用 iperf3，就需要编译 cmake 或者 ndk 编译。

这个仓库包含了一个 Dockerfile 和 Android.mk 文件，所以它的用法就是在构建 docker 镜像的过程中，基于`ubuntu:22.04`，在镜像中下载 android ndk、jdk、 [iperf](https://github.com/esnet/iperf) 的源码，并拷贝 Android.mk 文件，使用`ndk-build`构建产生二进制文件。

```bash
git clone https://github.com/davidBar-On/android-iperf3.git
# 使用Dockerfile构建镜像
docker build -t android-ndk:latest .

## 如果不需要修改代码可直接运行镜像容器，取出编译制品
docker run -d --name ndk android-ndk

## 如果需要修改代码则在这里跟随 `使用容器再次编译` 章节

#取出编译结果
mkdir -p binaries
docker cp -a ndk:/tmp/libs binaries
# 删除容器
docker stop ndk
docker rm ndk
```

> 1. `-a` 或 `--archive`：这个选项用于递归地复制文件和目录，并保留它们的元数据（如修改时间、访问时间、所有权等）。这相当于使用了 `-r`（递归）和 `-t`（保留时间戳）选项。**用于目录拷贝**

很顺利执行完成，拿到制品。

# 使用容器再次编译

如果想要**运行一个容器并持续查看其日志输出，而不是让容器在命令执行完毕后立即退出**，可以使用 `-dit` 标志来启动容器：

```bash
docker run -dit --name ndk android-ndk
```

> - `-d`：让容器在后台运行。
>- `-i`：保持容器的标准输入（STDIN）打开，即使没有附加到容器终端。
> - `-t`：分配一个伪终端。

这样我们就可以进入容器，对代码进行修改和再次编译。

```bash
# 进入容器
docker exec -it ndk /bin/bash
# 安装vim编辑器
apt-get -y update && apt-get -y install -q vim
```

> `-q`：这个选项使`apt-get`在安装过程中减少输出的信息量，即减少日志的详细程度。

可以通过查看 [android-iperf3](https://github.com/davidBar-On/android-iperf3) 项目的 Dockerfile 文件了解源码的位置和编译命令。它首先拷贝了 jni 目录下的 Android.mk 和 Application.mk，进入容器，然后下载了 [esnet/iperf: iperf3: A TCP, UDP, and SCTP network bandwidth measurement tool](https://github.com/esnet/iperf) Release 中的 3.17.1 源码，解压到 /tmp 目录下进行编译。

```bash
# 修改源码
cd /tmp/iperf-3.17.1/src
vim iperf_api.c
# 编译
ndk-build clean
ndk-build
```

# Android 集成

### iperf3 二进制文件位置及执行权限问题

二进制文件作为应用资源文件，是无法执行的，必须要在安装时或执行前拷贝到某个目录，可选的目录：

- `/tmp`：Android 下应用无写入权限
- `/data/local/tmp`：应用无写入权限，进 ADB 有写入、执行权限
- `files`：有写入权限，但高版本下无执行权限

```java
// 内部存储
// /data/data/<应用包名>/files
context.getFilesDir();
// 外部存储
// /storage/emulated/0/Android/data/<应用包名>/files
context.getExternalFilesDir(null);
```

- `cache`：有写入权限，但高版本下无执行权限

```java
// 内部存储
// /data/data/<应用包名>/cache
context.getCacheDir();
// 外部存储
// /storage/emulated/0/Android/data/<应用包名>/cache
context.getExternalCacheDir();
```

- 非应用目录的外部存储：有写入权限，似乎也没有执行权限

> [!NOTE]
>
> `files`和`cache`目录的访问不需要额外的权限，并且随着应用的卸载，这些目录下的数据也会被系统自动清除。对于外部存储目录的访问，从Android 6.0（API 级别 23）开始，写入外部存储需要用户授权（`WRITE_EXTERNAL_STORAGE`）。
>
> 非应用目录的外部存储需要申请存储权限`MANAGE_EXTERNAL_STORAGE`或`WRITE_INTERNAL_STORAGE（API 级别 31 引入）`。

似乎以上目录都无解了，直到看到了一片神贴：

> Android brought another big change to how the applications execute pre-compiled native binaries or shell scripts in their device, after android API level 28, the application will no longer have permission to execute anything in its */data/data* directory!
>
> Android API level 28（Android P）之后（Android Q），应用目录（`/data/data/<应用包名>`）内不再拥有执行权限。
>
> *Using the hacky trick that the android installer while installing can extract all the libraries to the \*/data/app\* folder and this place is executable (even after Android Q)*!
>
> 但是，应用安装时，会将库文件拷贝到`/data/app`，并且拥有执行权限，即使在 Android Q 之后。
>
> [Execute Native Binaries Android Q+ (No Root) – WithMe](https://withme.skullzbones.com/blog/programming/execute-native-binaries-android-q-no-root/)

也就是写入权限和执行权限二选一，似乎谷歌更多是防范应用下载脚本动态执行，并不防范一开始带进去的🥰。

在 app/src/main 下创建 libs 文件夹，在里面按照架构放好库文件

```
libs
├─arm64-v8a
├───libIperf3.so
├─armeabi-v7a
├───libIperf3.so
├─x86
├───libIperf3.so
├─x86_64
└───libIperf3.so
```

在 app build.gradle 中配置 jniLibs

```nginx
android {
    sourceSets {
        main {
            jniLibs.srcDirs = ['src/main/libs']
        }
    }
}
```

应用安装时，对应设备架构的库文件会被拷贝到`nativeLibraryDir`目录下，测试可以愉快地执行了且无需申请任何权限。

```java
// 库文件
// /data/app/~~-1ODQxxxxxxhJOT-TGxxxA==/com.limelight.debug-rxxxxxxeM-Lm-gzpwon3Q==/lib/arm64
context.getApplicationInfo().nativeLibraryDir;
```

> [!ATTENTION]
>
> 测试发现库文件的命名非常讲究：
>
> - 非 so 后缀：不会被拷贝
> - so 后缀：部分设备会拷贝，部分不会
> - libxxx.so：终于都正常拷贝了



# iperf3 问题修改

用 master 分支修改，比 Release 3.17.1 版本略新一点

### Android 下缓冲区刷新问题

测试发现打包进 Android 工程，使用`Process.exec`执行时，命令不会逐行输出到，而是等待执行完毕后一次性输出，导致不能知道实时的查询结果。而通过 ADB 调用执行时，能够正常输出。可能 Android 对于缓冲区有特殊的处理吧:dog:

iperf_api.c 5104行，`int iperf_printf(struct iperf_test *test, const char* format, ...)`：

这是一个线程安全的打印方法，将测试结果以一定的格式（format）输出，由于只用到了客户端模式，所以只修改`test->role == 'c'`的情况， 在`fprintf`和`vfprintf`后手动刷新缓冲区，即增加了 `fflush(test->outfile);` 代码。

```c
if (test->role == 'c') {
    if (ct) {
        r0 = fprintf(test->outfile, "%s", ct);
        fflush(test->outfile);
        if (r0 < 0) {
            r = r0;
            goto bottom;
        }
        r += r0;
    }
    if (test->title) {
        r0 = fprintf(test->outfile, "%s:  ", test->title);
        fflush(test->outfile);
        if (r0 < 0) {
            r = r0;
            goto bottom;
        }
        r += r0;
    }
    va_start(argp, format);
    r0 = vfprintf(test->outfile, format, argp);
    va_end(argp);
    fflush(test->outfile);
    if (r0 < 0) {
        r = r0;
        goto bottom;
    }
    r += r0;
}
```

### iperf3 运行过程中写入临时文件问题

执行过程报错，大意是创建新的 stream 时，无法创建临时文件，定位代码：

iperf_api.c 4470行：`struct iperf_stream * iperf_new_stream(struct iperf_test *test, int s, int sender)`：

获取一个临时目录，首选`test->tmp_template`，次选环境变量，再次如果是 Android，为`/data/local/tmp`，否则为`/tmp`，最后构造一个临时文件`<tempDir>/iperf3.XXXXXX`的临时文件路径。上文才说过，这两 tmp 目录在高版本 Android 应用都不具有写入权限了，大佬们只考虑我们在 ADB 上用吗！而这个`test->tmp_template`，默认是 null，在 iperf_api.c 中提供了 set 方法，但是竟然没有加入命令行参数，大佬们又偷懒了:dog:。

解决方法就很简单了，把这个`tmp_template`命令行参数提供出来，执行出来的时候传入一个有写入权限的目录进去。

iperf_api.h 104行：

```c
# 添加临时目录选项定义，不可设置太大，某则与其他定义相等就会冲突
#define OPT_TMP_TAMPLATE 40
```

iperf_api.c 1152行：`int iperf_parse_arguments(struct iperf_test *test, int argc, char **argv)` :

参数与选项定义的转换：`static struct option longopts[]`中添加：

```json
{"tmp-template", required_argument, NULL, OPT_TMP_TAMPLATE},
```

同一个方法 1650 行中参数转换加上

```c
case OPT_TMP_TAMPLATE:
iperf_set_test_template(test, optarg);
break;
```



> [!TIP]
>
> 注意 c 编译的条件`#if defined(HAVE_SSL) xxxx #endif`，不要写在这些条件中去，某则可能导致这个代码没有被编译。



目前 3.18版本的 iperf 增加了多个配置条件

```
struct iperf_stream *
iperf_new_stream(struct iperf_test *test, int s, int sender)
{
    struct iperf_stream *sp;
    int ret = 0;

    char template[1024];
    if (test->tmp_template) {
        snprintf(template, sizeof(template) / sizeof(char), "%s", test->tmp_template);
    } else {
        //find the system temporary dir *unix, windows, cygwin support
        char* tempdir = getenv("TMPDIR");
        if (tempdir == 0){
            tempdir = getenv("TEMP");
        }
        if (tempdir == 0){
            tempdir = getenv("TMP");
        }
        if (tempdir == 0){
#if defined(__ANDROID__)
            tempdir = "/data/local/tmp";
#else
            tempdir = "/tmp";
#endif
        }
        snprintf(template, sizeof(template) / sizeof(char), "%s/iperf3.XXXXXX", tempdir);
    }
....

```

根据这篇 https://stackoverflow.com/questions/58574682/cant-run-iperf3-on-android-without-root-permission 文章的回答，我们可以通过 `Os.setenv("TEMP", CACHE_DIR, true) `  的方式设置环境变量，来让 iperf 选择不同的临时路径，不过直接设置 TEMP 的环境变量可能会出问题，所以保险起见，我们会修改上述代码中读取的环境变量：

```
#if defined(__ANDROID__)
            tempdir = getenv("CUSTOMIPERFTEMPDIR");
#else
            tempdir = "/tmp";
#endif
```

然后在 android 应用中 `Os.setenv("CUSTOMIPERFTEMPDIR", CACHE_DIR, true) ` 执行 iperf 命令前设置下就好了。

# 附

### android-iperf3/Dockerfile

```dockerfile
FROM ubuntu:22.04
LABEL maintainer="david.cdb004@gmail.com"

RUN echo 'Acquire::Retries "9";' > /etc/apt/apt.conf.d/80-retries
RUN ls -l /etc/apt/apt.conf.d

RUN env
RUN echo $http_proxy
RUN echo $HTTP_PROXY
RUN cat /etc/hosts
RUN cat /etc/resolv.conf
#RUN update-alternatives --config java
#RUN KUKU

# In case previous build failed
RUN apt-get clean
RUN    rm -rf /var/lib/apt/lists/*

RUN apt-get -y update -qq
RUN    apt-get -y upgrade -qq
#RUN    apt-get -y install -q make bash git unzip wget curl openjdk-17-jdk build-essential autoconf nano tree
RUN    apt-get -y update && apt-get -y install -q make
RUN    apt-get -y update && apt-get -y install -q bash
RUN    apt-get -y update && apt-get -y install -q git
RUN    apt-get -y update && apt-get -y install -q unzip
RUN    apt-get -y update && apt-get -y install -q wget
RUN    apt-get -y update && apt-get -y install -q curl
RUN    apt-get -y update && apt-get -y install -q build-essential
RUN    apt-get -y update && apt-get -y install -q autoconf
RUN    apt-get -y update && apt-get -y install -q nano
RUN    apt-get -y update && apt-get -y install -q tree
RUN    apt-get -y update && apt-get -y install -q openjdk-17-jdk
#RUN    ls -l /var/cache/apt/archives
#RUN    apt-get -y update && apt-get -y install --download-only -q openjdk-17-jdk
#RUN    cd /var/cache/apt/archives && ls -l && dpkg -i *
RUN    apt-get clean
RUN    rm -rf /var/lib/apt/lists/*

ARG API_VERSION=26
ARG SDK_VERSION=9477386_latest
ARG NDK_VERSION=r22

ENV ANDROID_SDK_VERSION ${SDK_VERSION}
ENV ANDROID_SDK_HOME /opt/android-sdk
ENV ANDROID_SDK_FILENAME commandlinetools-linux-${ANDROID_SDK_VERSION}
ENV ANDROID_SDK_URL https://dl.google.com/android/repository/${ANDROID_SDK_FILENAME}.zip

RUN wget --no-check-certificate -q ${ANDROID_SDK_URL} && \
    mkdir -p ${ANDROID_SDK_HOME} && \
    unzip -q ${ANDROID_SDK_FILENAME}.zip -d ${ANDROID_SDK_HOME} && \
    rm -f ${ANDROID_SDK_FILENAME}.zip

ENV PATH=${PATH}:${ANDROID_SDK_HOME}/cmdline-tools/bin

RUN yes | sdkmanager --sdk_root=${ANDROID_SDK_HOME} --licenses > /dev/null
RUN    yes | sdkmanager --sdk_root=${ANDROID_SDK_HOME} "platforms;android-${API_VERSION}" > /dev/null

ENV ANDROID_NDK_VERSION ${NDK_VERSION}
ENV ANDROID_NDK_HOME /opt/android-ndk
ENV ANDROID_NDK_FILENAME android-ndk-${ANDROID_NDK_VERSION}-linux-x86_64
ENV ANDROID_NDK_URL https://dl.google.com/android/repository/${ANDROID_NDK_FILENAME}.zip

RUN wget --no-check-certificate -q ${ANDROID_NDK_URL} && \
    mkdir -p ${ANDROID_NDK_HOME} && \
    unzip -q ${ANDROID_NDK_FILENAME}.zip && \
    mv ./android-ndk-${ANDROID_NDK_VERSION}/* ${ANDROID_NDK_HOME} && \
    rm -f ${ANDROID_NDK_FILENAME}.zip

ENV PATH=${PATH}:${ANDROID_NDK_HOME}

ENV NDK_PROJECT_PATH=/tmp

# Config files

RUN mkdir -p /tmp/jni
COPY /jni/Android.mk /tmp/jni
COPY /jni/Application.mk /tmp/jni

##### OLD ########
# iPerf 3.14
# iPerf 3.15
# iPerf 3.16-beta1
##################

#############
# iPerf 3.16
#############

RUN cd /tmp && \
    wget --no-check-certificate -q https://downloads.es.net/pub/iperf/iperf-3.16.tar.gz && \
    tar -zxvf iperf-3.16.tar.gz && \
    rm -f iperf-3.16.tar.gz

COPY /iperf-3.16/* /tmp/iperf-3.16/

# Workaround for pthread_cancel() as it is not supported by Android NDK
COPY /src/iperf3-pthread.h /tmp/iperf-3.16/src/pthread.h
COPY /src/iperf3-pthread.c /tmp/iperf-3.16/src/
RUN cd /tmp/iperf-3.16 && \
    sed 's/#include <pthread.h>/#include \"pthread.h\"/' src/iperf.h > src/tmp_iperf.h && \
    cp src/tmp_iperf.h src/iperf.h

RUN cd /tmp/iperf-3.16 && \
    ./configure


############################################################################################
# iPerf 3.17.1
# 
# First version with no need for `sed` to include pthread_cancel().
############################################################################################

RUN cd /tmp && \
    wget --no-check-certificate -q https://downloads.es.net/pub/iperf/iperf-3.17.1.tar.gz && \
    tar -zxvf iperf-3.17.1.tar.gz && \
    rm -f iperf-3.17.1.tar.gz

COPY /iperf-3.17.1/* /tmp/iperf-3.17.1/

RUN cd /tmp/iperf-3.17.1 && \
    ./configure

##############
# Compile
##############

RUN ndk-build clean

RUN ndk-build NDK_APPLICATION_MK=/tmp/jni/Application.mk

RUN tree /tmp/libs
```

### Process.exec 用法

```java
public void exec(String[] args) {
    new Thread(() -> {
        String[] cmdAndArgs = new String[args.length + 1];
        cmdAndArgs[0] = getCmdPath();
        System.arraycopy(args, 0, cmdAndArgs, 1, args.length);

        try {
            Process process = Runtime.getRuntime().exec(cmdAndArgs);
            Log.i(TAG, "command: " + String.join(" ", cmdAndArgs));
            parseArgs(cmdAndArgs);

            try (
                InputStreamReader in = new InputStreamReader(process.getInputStream());
                BufferedReader outReader = new BufferedReader(in);
                BufferedReader errReader = new BufferedReader(new InputStreamReader(process.getErrorStream()));
            ) {
                String line;
                // 正常输出
                while ((line = outReader.readLine()) != null) {
                    parseToCallback(line);
                }
                // 异常输出
                while ((line = errReader.readLine()) != null) {
                    parseToCallback(line);
                }

                // 等待进程结束
                process.waitFor();
                Log.i(TAG, "exitValue: " + process.waitFor());
            }

        } catch (Exception e) {
            Log.e(TAG, e.getMessage(), e);
            callback.onError(e.getMessage());
        }
    }).start();
}
```

