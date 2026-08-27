<div align="center">
<img src="./Logo.png" alt="logo" width="100"/>
<h2>DontCrack4AndroidLinuxKernelSide</h2>
<h3>安卓(Android)Linux Kernel侧专用进程管理器</h3>
</div>


### 一、功能简介

- 用于提高安卓(Android)Linux Kernel上后台/守护进程的健壮性、可用性、时序稳定性
- 支持从 `init.rc` 中以 service 形式托管任意子进程，规避 init.rc 格式错误导致的开机失败
- 支持管理这些类型的进程：`二进制可执行程序`、`sh脚本`
- 实现将进程对应到端口号，可通过 Restful API 实现获取日志、开关进程等操作
- 启动的进程可以配置独立的：程序路径、环境变量、启动参数、预处理脚本、是否自动重启、崩溃自动重启次数、是否立即启动、端口号、日志最大缓存行数、单行日志最大字节数、日志本地存储路径、日志本地存储周期等
- 支持跨架构，免 CGO，支持任何可被 GO 编译器编译程序的架构使用（arm64 / armv7 / x86_64 等）
- 自动识别 Android 默认 shell 路径（`/system/bin/sh` 或 `/bin/sh`），不再硬编码 OpenHarmony 专属路径

### 二、基础用法

```
./DontCrack \
  -path "/Volumes/老固态/Projects/FasterEdge/DontCrack4AndroidLinuxKernelSide/example/childproc/childproc" \
  -args "-mode normal -interval 500ms -lifetime 5s" \
  -env "EXTRA_INFO=from_manager RESTART_ENV_COUNT=0" \
  -file-log -log-path ./example/logs/ -log-life-day 7 \
  -auto-restart -max-retries 2 \
  -start-now \
  -password 123456
```

| 配置项                | 类型     | 默认值                      | 说明                                                            |
| ------------------ | ------ | ------------------------ | ------------------------------------------------------------- |
| path               | string | ""                       | 要管理的程序路径（支持可执行文件、shell脚本等）                                    |
| args               | string | ""                       | 传递给程序的参数（可选）                                                  |
| pre                | string | ""                       | 启动前要执行的命令（在 sh 中执行，可用&&/;/||连接多条命令，默认为空）                      |
| env                | string | ""                       | 为子进程追加环境变量，如: "PATH=/system/bin:/system/xbin FOO=bar"；用空格或分号分隔 |
| auto-restart       | bool   | false                    | 是否自动重启                                                        |
| max-retries        | int    | 3                        | 最大重试次数（-1表示无限次，默认3次）                                          |
| start-now          | bool   | false                    | 是否立即启动                                                        |
| port               | int    | 11883                    | HTTP服务端口                                                      |
| password           | string | ""                       | 管理进程的密码（可选，默认为空且不开启密码保护）                                      |
| log-capacity       | int    | 200                      | 日志缓存的最大行数（默认200）                                              |
| log-max-line-bytes | int    | 1048576                  | 单行日志的最大字节数（用于bufio.Scanner，默认1MiB）                            |
| file-log           | bool   | false                    | 是否启用文件日志（默认false）                                             |
| log-path           | string | /data/logs/proc_manager/ | 本地日志文件目录（默认/data/logs/proc_manager/，按进程名创建子目录）                |
| log-life-day       | int    | 7                        | 本地日志文件保存天数（默认7天，新日志写入时会清理过期文件）                                |

### 三、adb 部署示例（从交叉编译到设备运行）

#### A. 在主机上交叉编译出 Linux arm64 二进制

```bash
cd DontCrack4AndroidLinuxKernelSide
GOOS=linux GOARCH=arm64 CGO_ENABLED=0 \
    go build -ldflags='-s -w' -o DontCrack .

# 顺带把 childproc 也编一份用于演示
cd example/childproc
GOOS=linux GOARCH=arm64 CGO_ENABLED=0 \
    go build -ldflags='-s -w' -o childproc .
```

> 旧设备是 32 位的改成 `GOARCH=arm`。模拟器或 x86 设备改成 `GOARCH=amd64`。

#### B. 推到设备并启动

```bash
# 推二进制到 /data/local/tmp/（普通用户可写，无需 root）
adb push DontCrack /data/local/tmp/DontCrack
adb push childproc /data/local/tmp/childproc

# 关键：adb shell 默认不会自动给你可执行权限，必须 chmod
adb shell chmod 755 /data/local/tmp/DontCrack /data/local/tmp/childproc

# 前台运行（Ctrl+C 终止会顺手 kill 子进程）
adb shell /data/local/tmp/DontCrack \
  -path /data/local/tmp/childproc \
  -args "-mode normal -interval 2s" \
  -start-now \
  -auto-restart -max-retries 3 \
  -port 11883 \
  -file-log -log-path /data/local/tmp/logs/ -log-life-day 7
```

#### C. 后台运行（断 adb 也不退出）

```bash
# nohup + & 让 DontCrack 不挂回 adb session
adb shell "cd /data/local/tmp && nohup ./DontCrack -path /data/local/tmp/childproc -start-now -auto-restart -port 11883 >/data/local/tmp/dontcrack.log 2>&1 &"

# 查进程
adb shell "ps -ef | grep DontCrack"

# 关掉
adb shell "pkill -f /data/local/tmp/DontCrack"
```

#### D. 远程调用 HTTP 控制接口

```bash
# 通过 USB 反向代理 (不需要设备 IP 可达)
adb reverse tcp:11883 tcp:11883
curl http://127.0.0.1:11883/heartbeat

# 或直连设备 IP（设备和电脑在同一网段时）
curl http://<device-ip>:11883/heartbeat?password=xxx
```

#### E. SELinux 受限的设备

部分 ROM 默认 enforce 模式，直接 `/data/local/tmp/DontCrack` 可能被 SELinux 阻止 exec。两种解法：

1. 推到 `/system/bin/` (需 root + remount + 修 sepolicy，麻烦)
2. 放到 selinux 允许的可执行目录：`/vendor/bin/` 或 `/data/local/tmp/` 通常默认允许
3. 临时宽容：`adb shell setenforce 0`（调试用）

#### F. 开机自启（可选）

参照 `example/example_init.rc`，把 `service dontcrack-edgecore /system/bin/DontCrack` 加进设备的 init.rc 后 `setprop ctl.restart dontcrack-edgecore` 即可触发。需要设备已 root 且 init.rc 可写。

### 四、接口文档

> /startup

- 接口说明：启动进程，同时会重置重启次数
- 请求方式：get、post
- 请求参数
  ```
  password: 密钥（可选params参数）
  ```
- 返回类型：文本
- 返回示例：
    ```
    ok
    ```

> /heartbeat

- 接口说明：获得心跳信息，会输出启动情况和缓存中的日志（同时会清除缓存）
- 请求方式：get、post
- 请求参数
  ```
  password: 密钥（可选params参数）
  ```
- 返回类型：JSON
- 返回示例：
     ```
	{
	"version": "1.0.20260826",
	"state": "stopped",
	"info": "进程管理器正常运行",
	"timestamp": "2026-08-25 15:28:04",
	"logs": [
	"[STDERR] 2026/08/25 15:27:55.647714 env restart count -> 1",
	"[STDERR] 2026/08/25 15:27:55.648316 childproc start | pid=32054 | mode=normal | interval=1s | lifetime=5s | msg=",
	"[STDERR] 2026/08/25 15:27:55.648345 args: /data/childproc -mode normal -interval 1s -lifetime 5s",
	"[STDERR] 2026/08/25 15:27:55.648352 env EXTRA_INFO=from_manager",
	"[STDERR] 2026/08/25 15:27:55.648353 env RESTART_ENV_COUNT=0",
	"[STDERR] 2026/08/25 15:27:56.668982 tick at 2026-08-25T15:27:56.64879725+08:00",
	"[STDERR] 2026/08/25 15:27:56.64879725+08:00 stderr burst at 2026-08-25T15:27:56.64879725+08:00",
	"[STDERR] 2026/08/25 15:28:00.686300 lifetime reached, exiting normally"
	],
	"process_pid": 0,
	"process_path": "/data/childproc",
	"restart_count": 0,
	"file_type": "binary_executable",
	"last_exit_time": "2026-08-25 15:28:00",
	"program_args": "-mode normal -interval 1s -lifetime 5s",
	"extra_env_raw": "PATH=/system/bin EXTRA_INFO=from_manager RESTART_ENV_COUNT=0"
	}
	```

> /shutdown

- 接口说明：终止进程
- 请求方式：get、post
- 请求参数
  ```
  password: 密钥（可选params参数）
  ```
- 返回类型：文本
- 返回示例：
  ```
  ok
  ```

### 五、细节说明

- 目标管理的进程的 Path 尽量使用全路径，例如 `/system/bin/edgecore`
- Android 默认 shell 一般位于 `/system/bin/sh`，部分定制 ROM 在 `/bin/sh`，DontCrack 会自动按顺序探测；两者都不存在时回退到 `sh`（依赖 `PATH`）
- 运行的文件使用 .sh 结尾、首行包含 `#!` 都将被识别为脚本文件，由探测到的 sh 执行
- 开启密码时，接口请求需要在 URL 参数中携带 `password` 参数，例如 `xxx/startup?password=123456`
- 受 SELinux 限制的设备，可使用 `seclabel u:r:init:s0` 或在 init.rc 中给 service 单独打标签
- init.rc 中参数顺序无要求，DontCrack 自带的 flag 包对顺序不敏感

### 六、与 Android init.rc 集成

将 DontCrack 放进 `/system/bin/`，然后在 init.rc 中以 `service` 的形式托管目标进程：

```rc
service dontcrack-edgecore /system/bin/DontCrack
    class core
    user root
    group root
    -env PATH=/system/bin:/system/xbin
    -path /system/bin/edgecore
    -pre /data/pres/edgecore-pre.sh
    -start-now
    -auto-restart
    -max-retries 3
    -port 10000
    -file-log
    -log-path /data/logs/proc_manager/
    -log-life-day 7

on boot
    start dontcrack-edgecore
```

完整示例见 `example/example_init.rc`。

### 七、使用技巧

- 单独使用时如果在会话中使用 & 作为命令结尾时一般会话结束的时候这个操作也会被杀死
- Go 语言程序的 log.Printf 默认将数据写到 os.Stderr，所以子进程中日志类型会显示为 [STDERR]，可以换成 fmt.Println 得到非 [STDERR] 的消息
- 因为可以通过 HTTP 带加密的形式操作进程，你可以根据此文档将进程操作结合 Android 设备的角色，再结合 AI + MCP（或 Skills）完成各种操作
- 除了可以使用直接功能保证进程健壮性，还可以利用反复重启机制实现进程轮询，预先脚本也能实现延迟启动、等待依赖进程等操作
- 与 OpenHarmony 版的差异:
  - 启动横幅、根路径消息改为 `DontCrack_arm64-android`
  - 自动探测 Android 默认 shell 路径，不再硬编码 `/bin/sh`
  - 默认示例配置改用 Android Init Language (`init.rc`) 写法
  - 心跳/启动参数中针对 Android 常见目录（`/system/bin`、`/data/`）提供默认建议
