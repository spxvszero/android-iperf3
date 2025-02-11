该 libs 文件构建修改源代码内容：

* 增加了安卓下自动刷新缓冲区的功能，避免任务执行完才会有输出
* 增加了安卓下的环境变量 `IPERF_ANDROID_TEMP_DIR` 通过设置 `Os.setenv("IPERF_ANDROID_TEMP_DIR", app_cache_dir, true)` 即可避免运行权限问题