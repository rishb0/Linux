# Runlevels in Linux

---
## What is a Runlevel ?

\- A **runlevel** is a predefined state of the operating system that determines which **services** and **processes** are running.
\- Linux uses **runlevels** to control what the system is doing — booting, running normally, rebooting, etc. Each runlevel starts and stops a specific set of services.
\- Runlevels are a concept from **SysV init** (the traditional init system). Modern Linux systems use **systemd**, which replaces runlevels with **targets** — but the old runlevel numbers are still mapped to systemd targets for compatibility.

---
## SysV Runlevels (Traditional)

| Runlevel | Name                       | Description                                                 |
| -------- | -------------------------- | ----------------------------------------------------------- |
| **0**    | Halt                       | Shuts down the system completely                            |
| **1**    | Single-user mode           | Only root; no networking; used for maintenance and recovery |
| **2**    | Multi-user (no NFS)        | Multi-user without network file system; varies by distro    |
| **3**    | Multi-user with networking | Full multi-user, CLI only — standard for servers            |
| **4**    | Undefined / Custom         | Not used by default; reserved for custom use                |
| **5**    | Multi-user with GUI        | Full multi-user with networking and graphical desktop       |
| **6**    | Reboot                     | Restarts the system                                         |

> **Note:** Runlevels 2, 3, and 4 vary slightly between distros. On **Debian/Ubuntu**, runlevels 2–5 are all full multi-user with networking (no distinction). On **RHEL/CentOS**, runlevel 3 = CLI only and runlevel 5 = GUI.

---
## Systemd Targets (Modern Equivalent)

Systemd replaced runlevels with **targets**. Each target is a group of units (services, mounts, etc.) that must be active to reach that system state.

| Runlevel | Systemd Target      | Description                        |
| -------- | ------------------- | ---------------------------------- |
| 0        | `poweroff.target`   | Halt / power off                   |
| 1        | `rescue.target`     | Single-user / rescue mode          |
| 2        | `multi-user.target` | Multi-user, no GUI                 |
| 3        | `multi-user.target` | Multi-user with networking, no GUI |
| 4        | `multi-user.target` | Same as above                      |
| 5        | `graphical.target`  | Multi-user with GUI                |
| 6        | `reboot.target`     | Reboot                             |

---
## Checking and Changing Runlevels / Targets

### SysV init (legacy — CentOS 6, older systems)

Check the current runlevel:

```bash
runlevel
```


Switch to a different runlevel immediately (e.g., runlevel 3):

```bash
init 3
```


Shut down the system:

```bash
init 0
```


Reboot the system:

```bash
init 6
```


Set the default runlevel at boot by editing `/etc/inittab`:

```bash
nano /etc/inittab
```


The default runlevel line looks like:

```
id:5:initdefault:
```

---
### Systemd (modern — CentOS 7+, RHEL, Ubuntu, Fedora, Arch, etc.)

**`systemctl`** is the **command-line interface (CLI) tool used to interact with `systemd`**, meaning it is how you **control, manage, and query system services, targets (runlevels), and the overall system state**—for example, starting/stopping services, enabling them at boot, switching system modes, or rebooting/shutting down the machine; in simple terms, if **`systemd` is the engine**, then **`systemctl` is the control panel you use to operate it**.


A **target** in `systemd` is **not a single service**, but a **logical grouping (collection) of units** that represents a _system state_. Think of it as a **bundle of services, mounts, sockets, etc.** that must be active together. This replaces the old SysV _runlevel concept_, but instead of one fixed number (like runlevel 5 = GUI), systemd uses **multiple granular targets** that can exist simultaneously.

In your output, many targets show `active` because **systemd is dependency-driven, not exclusive like runlevels**. For example, `graphical.target` depends on `multi-user.target`, which depends on `basic.target`, and so on. So when your system boots into GUI, it **activates a chain of targets**, not just one. That’s why you see multiple targets active—they are **stacked layers of system state**, not alternatives.

“Enabled” vs “active” is important: your command shows **active targets (currently reached)**, not necessarily enabled ones. Only the **default target** (e.g., `graphical.target`) is _enabled for boot_, but systemd automatically activates all **dependent targets** required to reach that state.

Difference from services: a **service** does actual work (e.g., `ssh`, `apache`), while a **target just groups and orchestrates them**. Targets don’t run processes—they act like **milestones/checkpoints**, telling systemd: _“to reach this state, all these services must be up.”_


Check the current target (equivalent to current runlevel):

```bash
systemctl get-default
```

List all available targets:

```bash
systemctl list-units --type=target
```

Switch to a target immediately (e.g., multi-user / runlevel 3):

```bash
sudo systemctl isolate multi-user.target
```

Switch to graphical target (runlevel 5):

```bash
sudo systemctl isolate graphical.target
```

Switch to rescue mode (runlevel 1):

```bash
sudo systemctl isolate rescue.target
```

Set the default target that boots into (persists across reboots):

```bash
sudo systemctl set-default multi-user.target
```

Set default to boot into GUI:

```bash
sudo systemctl set-default graphical.target
```

Shut down immediately:

```bash
sudo systemctl poweroff
```

Reboot immediately:

```bash
sudo systemctl reboot
```

---
## Viewing Default Target (Symlink)

The default target is stored as a symlink at `/etc/systemd/system/default.target`. You can inspect it directly:
- `/etc/systemd/system/default.target` exists **only if you have explicitly set or overridden the default target**.
```bash
ls -l /etc/systemd/system/default.target
```

The real default is stored at:

```php
/lib/systemd/system/default.target
```
(or sometimes `/usr/lib/systemd/system/default.target` depending on distro)

---
## Runlevel vs Target — Quick Mapping

```
Runlevel 0  →  poweroff.target
Runlevel 1  →  rescue.target
Runlevel 3  →  multi-user.target
Runlevel 5  →  graphical.target
Runlevel 6  →  reboot.target
```

---
## Special Modes

### Single-User Mode (Runlevel 1 / rescue.target)

- Only the root user can log in.
- Networking is disabled.
- Most services are stopped.
- Used for: password recovery, filesystem repair, critical maintenance.
- Accessed at boot by editing the GRUB entry and appending `single` or `1` to the kernel line.

### Emergency Mode (systemd only — below rescue)

- Even more minimal than rescue mode.
- Only the root filesystem is mounted (read-only).
- No services started at all.
- Used when the system is too broken to even reach rescue mode.

Switch to emergency mode:

```bash
sudo systemctl isolate emergency.target
```

---
## Quick Reference

| Task                          | SysV Command                              | Systemd Command                           |
| ----------------------------- | ----------------------------------------- | ----------------------------------------- |
| Check current runlevel/target | `runlevel`                                | `systemctl get-default`                   |
| Switch to CLI only (3)        | `init 3`                                  | `systemctl isolate multi-user.target`     |
| Switch to GUI (5)             | `init 5`                                  | `systemctl isolate graphical.target`      |
| Set default to CLI            | edit `/etc/inittab` → `id:3:initdefault:` | `systemctl set-default multi-user.target` |
| Set default to GUI            | edit `/etc/inittab` → `id:5:initdefault:` | `systemctl set-default graphical.target`  |
| Reboot                        | `init 6`                                  | `systemctl reboot`                        |
| Shutdown                      | `init 0`                                  | `systemctl poweroff`                      |

----
----
