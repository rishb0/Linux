# Linux Init Systems

---
## What is an Init System?

The **init system** is the **first process started by the Linux kernel** after boot. It always has **PID 1**. Every other process on the system is either started directly by init or is a descendant of a process init started.

Its responsibilities:

- Starting and stopping system services (daemons)
- Managing runlevels / targets (system states)
- Handling service dependencies (start A before B)
- Reaping orphaned processes (cleaning up zombie processes)
- Responding to shutdown, reboot, and crash events

Verify PID 1:

```bash
ps -p 1
```

Check which init system is running:

```bash
stat /proc/1/exe
```
- **`stat` command** → Displays detailed metadata (permissions, size, owner, timestamps, inode) of a file.
	- **Inode** → A unique identifier for a file in a filesystem that stores its metadata (permissions, owner, size, timestamps, disk block locations) but **not the filename itself**.
		- **Links** → The number of directory entries (hard links) that point to the same inode (i.e., how many filenames reference the same file data).
- **`/proc` directory** → Virtual filesystem that exposes live kernel and process information.
- **`1` (PID 1)** → The first process started by the kernel (usually init/system manager).
- **`exe`** → Symbolic link pointing to the actual executable binary of that process.

```bash
ls -l /proc/1/exe
```

---
## The Boot Process — Where Init Fits

```
BIOS / UEFI
    ↓
Bootloader (GRUB)
    ↓
Linux Kernel loads into memory
    ↓
Kernel mounts root filesystem
    ↓
Kernel starts /sbin/init (PID 1)
    ↓
Init starts all services and processes
    ↓
Login prompt / Desktop
```

---

## Types of Init Systems

| Init System   | Used In                                                    | Style                        |
| ------------- | ---------------------------------------------------------- | ---------------------------- |
| **SysV init** | CentOS 6, RHEL 6, older Debian/Ubuntu                      | Sequential, script-based     |
| **systemd**   | CentOS 7+, RHEL 7+, Ubuntu 15.04+, Fedora, Arch, Debian 8+ | Parallel, unit-based         |
| **Upstart**   | Ubuntu 9.10–14.10, CentOS 6 (parallel)                     | Event-driven                 |
| **OpenRC**    | Gentoo, Alpine Linux                                       | Sequential, dependency-aware |
| **runit**     | Void Linux, used in some Devuan setups                     | Simple, supervision-based    |
| **s6**        | Used in some containers and minimal systems                | Supervision-tree based       |


---

## 1. SysV Init (System V Init)

The original Unix init system, inherited from UNIX System V.

### How It Works

- Uses **shell scripts** stored in `/etc/init.d/` to start and stop services.
- Uses **runlevels** (0–6) to define system states.
- Services are started **sequentially** — one after another, which makes booting slow.
- Symlinks in `/etc/rc<N>.d/` directories define which scripts run at each runlevel.

### Key Directories and Files

|Path|Purpose|
|---|---|
|`/etc/init.d/`|All service scripts|
|`/etc/rc0.d/` to `/etc/rc6.d/`|Symlinks to scripts for each runlevel|
|`/etc/inittab`|Defines default runlevel and core init behaviour|
|`/etc/rc.local`|Commands to run after all services at boot|

Symlink naming convention in `/etc/rcN.d/`:

- `S` prefix = **Start** the service at this runlevel (e.g., `S20nginx`)
- `K` prefix = **Kill** (stop) the service at this runlevel (e.g., `K80nginx`)
- The number controls the **order** of execution.

### Service Management Commands (SysV)

Start a service:

```bash
service <name> start
```

Stop a service:

```bash
service <name> stop
```

Restart a service:

```bash
service <name> restart
```

Check service status:

```bash
service <name> status
```

Enable a service at a runlevel (using `chkconfig` — RHEL/CentOS):

```bash
chkconfig <name> on
```

Disable a service at a runlevel:

```bash
chkconfig <name> off
```

List all services and their runlevel status:

```bash
chkconfig --list
```

---

## 2. systemd

The dominant init system on modern Linux. Introduced to fix the slowness and complexity of SysV.

### How It Works

- Starts services **in parallel** — dramatically faster boot times.
- Uses **unit files** (plain text config files) instead of shell scripts.
- Tracks services using **cgroups** — every service's processes are grouped together.
- Uses **targets** instead of runlevels (but maps to them for compatibility).
- Has a **dependency system** — units declare what they need before/after them.
- Manages not just services but also mounts, timers, sockets, devices, and more.

### Unit Types
- **“Units”** refers to components of the systemd init system.
- **Units** → Configuration objects in systemd that define and manage system resources like services, mounts, devices, and scheduled tasks.

| Unit Type | Extension  | Purpose                               |
| --------- | ---------- | ------------------------------------- |
| Service   | `.service` | A daemon or background process        |
| Target    | `.target`  | A group of units (replaces runlevels) |
| Timer     | `.timer`   | Scheduled tasks (replaces `cron`)     |
| Mount     | `.mount`   | Filesystem mount points               |
| Socket    | `.socket`  | Socket-based service activation       |
| Device    | `.device`  | Kernel device                         |
| Path      | `.path`    | File/directory watch trigger          |
| Swap      | `.swap`    | Swap space                            |



### Unit File Location

|Path|Purpose|
|---|---|
|`/lib/systemd/system/`|Default unit files (shipped by packages)|
|`/etc/systemd/system/`|Admin-created or overriding unit files (takes priority)|
|`/run/systemd/system/`|Runtime unit files (temporary)|

### Structure of a `.service` Unit File

```ini
[Unit]
Description=My Example Service
After=network.target

[Service]
Type=simple
ExecStart=/usr/bin/myapp
Restart=on-failure
User=myuser

[Install]
WantedBy=multi-user.target
```

|Section|Purpose|
|---|---|
|`[Unit]`|Description and dependency ordering|
|`[Service]`|How to start, stop, and manage the process|
|`[Install]`|Which target should pull this unit in|

### systemctl — Main Command

Start a service:

```bash
sudo systemctl start <service>
```

Stop a service:

```bash
sudo systemctl stop <service>
```

Restart a service:

```bash
sudo systemctl restart <service>
```

Reload config without full restart (if supported):

```bash
sudo systemctl reload <service>
```

Check service status:

```bash
sudo systemctl status <service>
```

Enable service to start at boot:

```bash
sudo systemctl enable <service>
```

Disable service from starting at boot:

```bash
sudo systemctl disable <service>
```

Enable and start in one command:

```bash
sudo systemctl enable --now <service>
```

Check if a service is enabled:

```bash
systemctl is-enabled <service>
```

Check if a service is active (running):

```bash
systemctl is-active <service>
```

List all running services:

```bash
systemctl list-units --type=service
```

List all services including inactive:

```bash
systemctl list-units --type=service --all
```

List failed units:

```bash
systemctl --failed
```

Reload systemd after editing unit files:

```bash
sudo systemctl daemon-reload
```

View a unit file:

```bash
systemctl cat <service>
```

Edit a unit file (opens override):

```bash
sudo systemctl edit <service>
```

### journalctl — Log Viewer

View all system logs:

```bash
journalctl
```

View logs for a specific service:

```bash
journalctl -u <service>
```

Follow logs in real time (like `tail -f`):

```bash
journalctl -u <service> -f
```

View logs since last boot:

```bash
journalctl -b
```

View logs from the previous boot:

```bash
journalctl -b -1
```

View logs since a specific time:

```bash
journalctl --since "2024-01-01 10:00:00"
```

View last N lines:

```bash
journalctl -n 50
```

View kernel messages only:

```bash
journalctl -k
```

### System State Commands

Reboot:

```bash
sudo systemctl reboot
```

Shut down:

```bash
sudo systemctl poweroff
```

Suspend:

```bash
sudo systemctl suspend
```

Switch to a target immediately:

```bash
sudo systemctl isolate multi-user.target
```

Get default boot target:

```bash
systemctl get-default
```

Set default boot target:

```bash
sudo systemctl set-default graphical.target
```

---

## 3. Upstart

- Developed by Canonical for Ubuntu (used Ubuntu 9.10 to 14.10).
- **Event-driven** — services start in response to events (e.g., "network is up") rather than runlevel changes.
- Faster than SysV but superseded by systemd.
- Still present in CentOS 6 alongside SysV.
- Job configs stored in `/etc/init/` as `.conf` files.
- Now effectively obsolete — Ubuntu and CentOS both moved to systemd.

---

## 4. OpenRC

- Used in **Gentoo** and **Alpine Linux**.
- Dependency-aware and faster than SysV but still script-based (not as complex as systemd).
- Does **not** run as PID 1 directly — works alongside a minimal init (like `sysvinit` or `busybox init`).
- Service scripts stored in `/etc/init.d/`.

Start a service:

```bash
rc-service <name> start
```

Enable service at boot:

```bash
rc-update add <name> default
```

List services and their runlevel status:

```bash
rc-status
```

---

## 5. runit

- Used in **Void Linux** and optionally in some minimal/container setups.
- Three-stage init: stage 1 (boot), stage 2 (supervise services), stage 3 (shutdown).
- Services are directories in `/etc/sv/` — each contains a `run` script.
- Very fast, simple, and reliable.
- Each service is continuously supervised — auto-restarted on crash.

Start a service:

```bash
sv start <service>
```

Stop a service:

```bash
sv stop <service>
```

Check service status:

```bash
sv status <service>
```

---

## Key Concepts

### PID 1 and Why It Matters

PID 1 is special — the kernel treats it differently from all other processes. If PID 1 dies, the kernel panics and the system crashes. PID 1 is also responsible for **reaping zombie processes** (collecting exit status of orphaned child processes).

### Daemon

A **daemon** is a background service process that runs continuously without user interaction. Examples: `sshd`, `httpd`, `cron`, `NetworkManager`. The `d` at the end of a process name conventionally means daemon.

### Service vs Daemon

- A **daemon** is the actual running process.
- A **service** is the managed unit (the config + the daemon together). In systemd terms, a `.service` unit manages a daemon.

### cgroups (Control Groups)

Systemd uses **cgroups** to track and manage every process belonging to a service. Even if a service forks child processes, systemd knows about all of them and can stop them all cleanly. This was a major improvement over SysV where orphaned child processes were a problem.

---

## Quick Comparison

|Feature|SysV init|systemd|OpenRC|runit|
|---|---|---|---|---|
|Boot speed|Slow (sequential)|Fast (parallel)|Medium|Fast|
|Config format|Shell scripts|Unit files (INI)|Shell scripts|Shell scripts|
|Dependency handling|Manual ordering|Automatic|Dependency-aware|Simple supervision|
|Logging|Separate syslog|Built-in journald|Separate syslog|Separate|
|Used in|Legacy RHEL/CentOS|Most modern distros|Gentoo, Alpine|Void Linux|
|Complexity|Simple|Complex but powerful|Medium|Very simple|


----
-----
