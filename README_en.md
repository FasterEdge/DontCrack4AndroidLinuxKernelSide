<div align="center">
<img src="./Logo.png" alt="logo" width="100"/>
<h2>DontCrack4AndroidLinuxKernelSide</h2>
<h3>Dedicated Process Manager for the Android Linux Kernel</h3>
</div>


### 1. Features

- Improves the robustness, availability and timing stability of background/daemon processes on the Android Linux Kernel
- Supports hosting any child process as a service from `init.rc`, avoiding boot failures caused by malformed init.rc entries
- Supports managing these process types: `binary executable`, `sh script`
- Maps processes to port numbers; RESTful API for getting logs, starting/stopping processes, etc.
- Each managed process can be configured independently with: program path, environment variables, startup arguments, pre-processing script, auto-restart toggle, max crash-retry count, start-now flag, port number, log cache line limit, per-line byte limit, local log storage path, log retention period, etc.
- Cross-architecture, no CGO required; supports any architecture that the Go compiler can target (arm64 / armv7 / x86_64 etc.)
- Auto-detects the Android default shell path (`/system/bin/sh` or `/bin/sh`), no longer hardcoding the OpenHarmony-specific path

### 2. Basic Usage

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

| Config | Type | Default | Description |
|--------|------|---------|-------------|
| path | string | "" | Path of the program to manage (supports executables, shell scripts etc.) |
| args | string | "" | Arguments passed to the program (optional) |
| pre | string | "" | Command to run before startup (executed in sh; multiple commands can be chained with && / ; / ||, default empty) |
| env | string | "" | Environment variables to append for the child process, e.g. "PATH=/system/bin:/system/xbin FOO=bar"; separated by spaces or semicolons |
| auto-restart | bool | false | Whether to auto-restart on crash |
| max-retries | int | 3 | Max retry count (-1 means unlimited, default 3) |
| start-now | bool | false | Whether to start immediately |
| port | int | 11883 | HTTP service port |
| password | string | "" | Password for managing the process (optional; no password protection if empty) |
| log-capacity | int | 200 | Max lines of cached logs (default 200) |
| log-max-line-bytes | int | 1048576 | Max bytes per log line (for bufio.Scanner, default 1 MiB) |
| file-log | bool | false | Whether to enable file logging (default false) |
| log-path | string | /data/logs/proc_manager/ | Local log file directory (default /data/logs/proc_manager/, subdirectories created per process name) |
| log-life-day | int | 7 | Log retention in days (default 7; expired logs cleaned when new logs are written) |

### 3. adb Deployment Example (From Cross-Compilation to On-Device Run)

#### A. Cross-compile a Linux arm64 binary on the host

```bash
cd DontCrack4AndroidLinuxKernelSide
GOOS=linux GOARCH=arm64 CGO_ENABLED=0 \
    go build -ldflags='-s -w' -o DontCrack .

# Also build childproc for the demo
cd example/childproc
GOOS=linux GOARCH=arm64 CGO_ENABLED=0 \
    go build -ldflags='-s -w' -o childproc .
```

> For older 32-bit devices, change to `GOARCH=arm`. For emulators or x86 devices, change to `GOARCH=amd64`.

#### B. Push to the device and start

```bash
# Push the binaries to /data/local/tmp/ (writable by regular users, no root needed)
adb push DontCrack /data/local/tmp/DontCrack
adb push childproc /data/local/tmp/childproc

# Key: adb shell does not grant execute permission automatically, you must chmod
adb shell chmod 755 /data/local/tmp/DontCrack /data/local/tmp/childproc

# Run in foreground (Ctrl+C termination also kills the child process)
adb shell /data/local/tmp/DontCrack \
  -path /data/local/tmp/childproc \
  -args "-mode normal -interval 2s" \
  -start-now \
  -auto-restart -max-retries 3 \
  -port 11883 \
  -file-log -log-path /data/local/tmp/logs/ -log-life-day 7
```

#### C. Run in background (survives disconnecting adb)

```bash
# nohup + & keeps DontCrack from hanging on the adb session
adb shell "cd /data/local/tmp && nohup ./DontCrack -path /data/local/tmp/childproc -start-now -auto-restart -port 11883 >/data/local/tmp/dontcrack.log 2>&1 &"

# Check the process
adb shell "ps -ef | grep DontCrack"

# Stop it
adb shell "pkill -f /data/local/tmp/DontCrack"
```

#### D. Call the HTTP control API remotely

```bash
# Via USB reverse proxy (no need for the device IP to be reachable)
adb reverse tcp:11883 tcp:11883
curl http://127.0.0.1:11883/heartbeat

# Or connect directly to the device IP (when the device and PC are on the same subnet)
curl http://<device-ip>:11883/heartbeat?password=xxx
```

#### E. SELinux-restricted devices

Some ROMs default to enforcing mode, and directly running `/data/local/tmp/DontCrack` may be blocked by SELinux. Two solutions:

1. Push to `/system/bin/` (requires root + remount + sepolicy changes, cumbersome)
2. Place it in a SELinux-permitted executable directory: `/vendor/bin/` or `/data/local/tmp/` is usually allowed by default
3. Temporarily set permissive: `adb shell setenforce 0` (for debugging)

#### F. Boot auto-start (optional)

Refer to `example/example_init.rc`; add `service dontcrack-edgecore /system/bin/DontCrack` to the device's init.rc, then trigger with `setprop ctl.restart dontcrack-edgecore`. Requires a rooted device with a writable init.rc.

### 4. API Documentation

> /startup

- Description: Starts the process and resets the retry count
- Method: GET, POST
- Request parameters:
  ```
  password: secret key (optional params parameter)
  ```
- Response type: text
- Example response:
    ```
    ok
    ```

> /heartbeat

- Description: Returns heartbeat information, including startup status and cached logs (logs are cleared after reading)
- Method: GET, POST
- Request parameters:
  ```
  password: secret key (optional params parameter)
  ```
- Response type: JSON
- Example response:
     ```
	{
	"version": "1.0.20260831",
	"state": "stopped",
	"info": "Process manager running normally",
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

- Description: Terminates the process
- Method: GET, POST
- Request parameters:
  ```
  password: secret key (optional params parameter)
  ```
- Response type: text
- Example response:
  ```
  ok
  ```

### 5. Details

- Use full paths for the managed process's Path whenever possible, e.g. `/system/bin/edgecore`
- The Android default shell is usually at `/system/bin/sh`; some custom ROMs use `/bin/sh`. DontCrack probes both in order automatically; if neither exists it falls back to `sh` (relying on `PATH`)
- Files ending with `.sh` or containing `#!` on the first line are recognized as scripts and executed by the probed sh
- When password protection is enabled, API requests must include the `password` parameter in the URL, e.g. `xxx/startup?password=123456`
- On SELinux-restricted devices, you can use `seclabel u:r:init:s0` or tag the service separately in init.rc
- Argument order in init.rc does not matter; DontCrack's flag package is order-insensitive

### 6. Integration with Android init.rc

Put DontCrack into `/system/bin/`, then host the target process as a `service` in init.rc:

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

See the full example at `example/example_init.rc`.

### 7. Tips

- When used standalone, if `&` is used at the end of a command in a session, the operation may be killed when the session ends
- Go's `log.Printf` writes to `os.Stderr` by default, so log entries from child processes will show as `[STDERR]`; switch to `fmt.Println` to get non-`[STDERR]` messages
- Since processes can be controlled via encrypted HTTP, you can combine this with the Android device's role, AI + MCP (or Skills) to implement various automation operations
- Besides using direct functionality for process robustness, the repeated restart mechanism can also be used for process polling, and the pre-script can implement delayed startup, waiting for dependent processes, etc.
- Differences from the OpenHarmony version:
  - Startup banner and root path messages changed to `DontCrack_arm64-android`
  - Auto-detects the Android default shell path, no longer hardcoding `/bin/sh`
  - Default example configuration uses the Android Init Language (`init.rc`) syntax
  - Heartbeat/startup arguments provide default suggestions for common Android directories (`/system/bin`, `/data/`)