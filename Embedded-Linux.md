# Embedded Linux: Beginner to Advanced

Embedded Linux is Linux adapted to a dedicated device such as a router, camera,
industrial controller, vehicle computer, or IoT product. This guide moves from
first principles to production engineering for embedded systems.

## Table of Contents

- [1. Mental Model](#1-mental-model)
- [2. Hardware and Architecture](#2-hardware-and-architecture)
- [3. Host Setup and Serial Console](#3-host-setup-and-serial-console)
- [4. Cross-Compilation](#4-cross-compilation)
- [5. Boot Process](#5-boot-process)
- [6. U-Boot](#6-u-boot)
- [7. Linux Kernel](#7-linux-kernel)
- [8. Device Tree](#8-device-tree)
- [9. Root Filesystem](#9-root-filesystem)
- [10. Init and Services](#10-init-and-services)
- [11. Buildroot and Yocto](#11-buildroot-and-yocto)
- [12. Applications and IPC](#12-applications-and-ipc)
- [13. Networking](#13-networking)
- [14. Storage and Filesystems](#14-storage-and-filesystems)
- [15. Device Drivers](#15-device-drivers)
- [16. Real-Time Systems](#16-real-time-systems)
- [17. Power and Resources](#17-power-and-resources)
- [18. Debugging](#18-debugging)
- [19. Security](#19-security)
- [20. Reliability and Updates](#20-reliability-and-updates)
- [21. Production Workflow](#21-production-workflow)
- [22. Learning Projects](#22-learning-projects)
- [23. Quick Reference](#23-quick-reference)
- [24. Is This Sufficient for Placement?](#24-is-this-sufficient-for-placement)
- [25. Recommended Learning Resources](#25-recommended-learning-resources)

---

# 1. Mental Model

**Host** means the development computer. **Target** means the board. A native
build runs and produces code for the same machine; a cross-build runs on the
host and produces code for the target.

```text
Application and business logic
	|
Libraries, IPC, services, update agent
	|
Init system and root filesystem
	|
Linux kernel, drivers, device tree
	|
Bootloader (often U-Boot)
	|
Boot ROM and hardware
```

| Artifact | Responsibility | Typical output |
| --- | --- | --- |
| Boot ROM | Starts from a fixed hardware location | Vendor code |
| Bootloader | Initializes RAM and loads images | `u-boot.bin` |
| Kernel | Scheduling, memory, drivers, networking | `Image` |
| Device tree | Describes board hardware | `.dtb` |
| Root filesystem | Programs, libraries, configuration | `ext4`, `squashfs`, `cpio` |
| Application | Product behavior | Executable and assets |

The kernel is not the complete operating system. A board needs these artifacts,
plus a deployment and recovery process.

# 2. Hardware and Architecture

Before writing code, record the CPU family, cores, RAM, physical address map,
boot source, flash layout, UART voltage, network interfaces, GPIO/I2C/SPI/UART
connections, interrupts, watchdog, regulators, thermal limits, and reset
behavior. Read the schematic and SoC reference manual with the software docs.

```bash
uname -m
cat /proc/cpuinfo
getconf LONG_BIT
cat /proc/meminfo
```

Common architectures are ARM (`arm`, `aarch64`), MIPS, RISC-V (`riscv64`), and
x86. Architecture alone does not identify a board: SoC, peripherals, memory
map, bootloader configuration, and device tree matter too.

# 3. Host Setup and Serial Console

On a Debian-like host, install a compiler, build tools, serial terminal,
device-tree compiler, image tools, and target debugger:

```bash
sudo apt update
sudo apt install git build-essential bc bison flex libssl-dev libelf-dev device-tree-compiler ncurses-dev cpio rsync u-boot-tools wget unzip minicom gdb-multiarch
mkdir -p ~/embedded/{src,downloads,toolchains,images}
gcc --version
make --version
```

A serial console is usually the first debug interface. Check the voltage first;
many boards use 3.3 V TTL, not RS-232. Connect ground, target TX to adapter RX,
and target RX to adapter TX.

```bash
ls /dev/ttyUSB* /dev/ttyACM* 2>/dev/null
sudo usermod -aG dialout "$USER"
screen /dev/ttyUSB0 115200
```

`115200 8N1` with hardware flow control disabled is common. Save boot logs.

# 4. Cross-Compilation

A cross-toolchain contains binutils, a compiler, target C libraries, a sysroot
with target headers and libraries, and a target-aware debugger. A tuple such as
`aarch64-linux-gnu` identifies the target.

```bash
export ARCH=arm64
export CROSS_COMPILE=aarch64-linux-gnu-
$CROSS_COMPILE-gcc --version
```

```c
#include <stdio.h>
int main(void) { puts("hello from embedded Linux"); return 0; }
```

```bash
$CROSS_COMPILE-gcc -Wall -Wextra -O2 hello.c -o hello
file hello
$CROSS_COMPILE-readelf -h hello
```

`file` and `readelf` reveal architecture, interpreter, and dynamic library
requirements. A target binary needs matching loader and libraries unless it is
intentionally built static.

# 5. Boot Process

1. The SoC Boot ROM selects a boot device and loads a first-stage image.
2. The loader initializes clocks, pins, and external RAM.
3. A bootloader loads the kernel, device tree, and optional initramfs.
4. The kernel initializes drivers, mounts the root filesystem, and starts PID 1.
5. PID 1 starts services and the product application.

Failure before the kernel banner usually points to hardware or bootloader
configuration. A kernel panic after the banner commonly involves the kernel,
device tree, or root filesystem. A failure after init starts is usually user
space.

```text
console=ttyS0,115200 root=/dev/mmcblk0p2 rootwait rw loglevel=4
```

Important parameters include `console=`, `root=`, `rootwait`, `ro`, `rw`,
`init=`, `earlycon`, and `loglevel=`.

# 6. U-Boot

U-Boot initializes hardware, accesses storage and networks, loads images, and
provides recovery commands.

```text
=> printenv
=> mmc list
=> fatls mmc 0:1
=> printenv bootargs
=> setenv bootargs 'console=ttyS0,115200 root=/dev/mmcblk0p2 rootwait rw'
```

Do not save experimental environment changes until understood; a bad `bootcmd`
can make a healthy board appear dead. Development often uses TFTP:

```text
=> setenv serverip 192.168.1.10
=> setenv ipaddr 192.168.1.20
=> tftpboot ${loadaddr} Image
=> tftpboot ${fdt_addr_r} board.dtb
=> booti ${loadaddr} - ${fdt_addr_r}
```

FIT images can package a kernel, device tree, ramdisk, hashes, and signatures.

# 7. Linux Kernel

Start with board support or a close upstream defconfig:

```bash
make ARCH=arm64 CROSS_COMPILE=aarch64-linux-gnu- defconfig
make ARCH=arm64 CROSS_COMPILE=aarch64-linux-gnu- menuconfig
make -j"$(nproc)" ARCH=arm64 CROSS_COMPILE=aarch64-linux-gnu- Image modules dtbs
```

Configure CPU support, buses, storage, filesystems, networking, cryptography,
security, namespaces, cgroups, tracing, and debug facilities. Keep `.config`
under version control and prefer upstream drivers and bindings.

```bash
make ARCH=arm64 CROSS_COMPILE=aarch64-linux-gnu- modules
make ARCH=arm64 CROSS_COMPILE=aarch64-linux-gnu- INSTALL_MOD_PATH="$PWD/rootfs" modules_install
lsmod
modinfo example_driver.ko
modprobe example_driver
dmesg | tail -n 50
```

Build modules against the exact kernel release and configuration that runs.

# 8. Device Tree

Device tree describes addresses, interrupts, clocks, GPIOs, regulators, and
driver bindings for hardware that cannot be discovered reliably.

```dts
&i2c1 {
	status = "okay";
	sensor@48 {
		compatible = "vendor,temperature-sensor";
		reg = <0x48>;
	};
};
```

```bash
dtc -I dts -O dtb -o board.dtb board.dts
dtc -I dtb -O dts -o - board.dtb | less
```

`compatible` connects a node to a driver. For a missing device, check bus,
`status`, address, pinctrl, clocks, regulators, and interrupt before changing
the driver. Do not describe hardware that is not physically wired.

# 9. Root Filesystem

A root filesystem commonly contains `/bin`, `/sbin`, `/etc`, `/dev`, `/proc`,
`/sys`, `/run`, `/usr`, `/var`, `/tmp`, and `/lib`. The kernel needs a valid init
program. BusyBox is useful for a minimal system; production images also need
libraries, certificates, users, permissions, configuration, and updates.

```bash
mkdir -p rootfs/{bin,sbin,etc,proc,sys,dev,run,tmp}
mount -t proc proc /proc
mount -t sysfs sysfs /sys
mount -t devtmpfs devtmpfs /dev
```

Choose `mdev`, `eudev`, or another device manager based on size and requirements.
Build images with `mkfs.ext4`, `mksquashfs`, or `cpio`.

# 10. Init and Services

PID 1 must reap orphaned children and handle shutdown signals.

```bash
ps -p 1 -o pid,comm,args
cat /proc/1/comm
```

BusyBox `init` is small and uses `/etc/inittab`. systemd provides dependencies,
logging, sandboxing, and device integration. OpenRC and runit are alternatives.
Keep dependencies explicit and run services as unprivileged users.

```text
::sysinit:/etc/init.d/rcS
ttyS0::askfirst:-/bin/sh
::restart:/sbin/init
```

# 11. Buildroot and Yocto

**Buildroot** is a good fit for a compact product and quick board bring-up. It
builds the toolchain, bootloader, kernel, rootfs, and packages from one config.

```bash
make <board>_defconfig
make menuconfig
make -j"$(nproc)"
ls output/images/
```

Use a `br2-external` tree for product packages and board files.

**Yocto/OpenEmbedded** builds layered, customizable distributions. Recipes
describe fetching, configuring, compiling, packaging, and deploying software.

```bash
source oe-init-build-env build
bitbake-layers show-layers
bitbake core-image-minimal
```

Learn recipes, tasks, layers, machine and distro configuration, `IMAGE_INSTALL`,
`DISTRO_FEATURES`, shared-state caching, and `devtool`. Choose Buildroot for
simplicity; choose Yocto for multiple machines, feeds, compliance, or a
long-lived distribution.

# 12. Applications and IPC

Applications must handle missing hardware, startup order, timeouts, reconnects,
corrupt input, full storage, and clean shutdown. IPC choices include pipes,
Unix sockets, TCP/UDP sockets, POSIX queues, shared memory, signals, `epoll`,
and D-Bus. Use versioned protocols, bounded messages, and timeouts.

Log timestamps, severity, component, and correlation IDs. Treat every network
or peripheral input as untrusted. Use `epoll` for many file descriptors and
shared memory only with explicit synchronization.

# 13. Networking

```bash
ip link
ip addr
ip route
ss -tulpen
cat /etc/resolv.conf
ping -c 3 192.168.1.1
ethtool eth0
```

Debug in layers: link, interface configuration, route, DNS, then application
protocol. Use `tcpdump` on the host when packet behavior is unclear. Production
protocols need authentication, encryption, replay protection, and bounded use.

# 14. Storage and Filesystems

| Filesystem | Typical use |
| --- | --- |
| `ext4` | General writable storage |
| `squashfs` | Compressed read-only system image |
| `UBIFS` | Raw NAND through UBI |
| `JFFS2` | Older or smaller raw-flash partitions |
| `tmpfs` | Temporary RAM-backed data |
| `F2FS` | Some managed-flash workloads |

Raw NAND has bad blocks and erase constraints; do not treat it like a normal
disk. For power-loss safety, write a new config, `fsync` it, rename it, and
consider syncing the directory. Rotate logs and limit flash writes.

```bash
lsblk -f
mount
df -h
dmesg | grep -iE 'mmc|nand|ubi|ext4|error'
sync
```

# 15. Device Drivers

Prefer kernel subsystems: GPIO, IIO, PWM, RTC, LED, watchdog, regulator, SPI,
I2C, UART, DRM, V4L2, ALSA, and networking. A driver probes hardware, acquires
resources, configures it, handles interrupts or DMA, exposes operations, and
cleans up. Keep interrupt handlers short; defer work when possible. Never sleep
in atomic context and validate user input.

User interfaces include `/dev`, `/sys`, `/proc`, and Netlink. `debugfs` is for
development, not a stable product API. For simple control, userspace GPIO/I2C/SPI
may be safer than custom kernel code.

# 16. Real-Time Systems

Real-time means bounded response under a defined workload, not merely fast
average performance. Define the deadline and measure worst-case latency.

```bash
chrt -p 1
taskset -cp 0 "$PID"
cyclictest -S -p 80 -m -n -D 60s
```

PREEMPT_RT, CPU isolation, interrupt affinity, priority inheritance, and
careful locking can improve determinism. Measure with storage, network, and
thermal load active.

# 17. Power and Resources

```bash
free -h
vmstat 1
cat /proc/interrupts
cat /sys/class/thermal/thermal_zone0/temp
```

Use runtime power management, suspend states, clock scaling, regulator control,
and device low-power modes. Check wake sources and resume latency. Avoid swap
when flash endurance or determinism matters. Use cgroups for resource limits
and test low-memory and thermal behavior.

# 18. Debugging

1. Reproduce the failure and record hardware, image, configuration, and steps.
2. Capture serial, kernel, service, and application logs.
3. Reduce it to the smallest failing component.
4. Change one variable and repeat the same test.
5. Preserve the failing image and logs before rebuilding.

```bash
dmesg -w
journalctl -b -p warning
top
ps aux
lsof
strace -f -tt -o trace.log ./application
gdb-multiarch ./application core
```

For kernel work, use dynamic debug, ftrace, trace-cmd, perf, lockdep, KASAN,
UBSAN, and kgdb as appropriate. Keep debug symbols for crash analysis.

# 19. Security

- Minimize packages, services, capabilities, and open ports.
- Use unique device identity and protected key storage.
- Verify boot components with a hardware root of trust where available.
- Sign updates and verify them before installation.
- Encrypt sensitive data in transit and at rest.
- Apply least privilege, permissions, seccomp, namespaces, and MAC.
- Disable debug interfaces and lock production bootloader access.
- Track CVEs, generate an SBOM, and define a patch support period.

Security is a system property. Signed applications do not compensate for an
unlocked bootloader, exposed root shell, shared credentials, or insecure update
rollback behavior.

# 20. Reliability and Updates

Design for interrupted power, network loss, full disks, corrupt input, clock
errors, sensor failure, and repeated reboot. A watchdog should detect a real
liveness failure and have a documented recovery action.

- **A/B images**: boot the new slot, verify health, mark it good, or revert.
- **Recovery partition**: retain a known-good rescue environment.
- **Package updates**: update individual packages with dependency planning.
- **Read-only system plus writable data**: reduce system corruption.

Updates must be authenticated, power-loss safe, observable, and tested with an
interrupted installation. Define factory reset, downgrade prevention, key
rotation, and failed-boot recovery.

# 21. Production Workflow

Pin source revisions, toolchain versions, configuration, patches, and downloads.
Use CI, shared download caches, and reproducible builds. Record image metadata:
Git revision, machine, toolchain, configuration hash, and signing identity
without exposing private keys.

Test unit logic, host services, hardware smoke paths, hardware-in-the-loop,
upgrades, recovery, stress, soak, power cuts, thermal behavior, and security.
An automated fixture should control power, serial capture, networking, and
flashing. Before release, verify credentials, debug services, signatures,
compatibility, bounded logs, watchdog behavior, recovery, and documentation.

# 22. Learning Projects

1. Cross-compile hello-world and run it over serial or SSH.
2. Build a BusyBox rootfs and boot it with QEMU.
3. Add an init script that starts an application and handles shutdown.
4. Build an image with Buildroot and add a custom package.
5. Add an I2C sensor through device tree and an existing subsystem.
6. Expose sensor data through a Unix or TCP service.
7. Write a kernel module, then assess whether a standard subsystem is better.
8. Simulate A/B updates with interrupted writes and rollback.
9. Measure boot time, memory, power, and latency under load.
10. Produce a signed image and test recovery from corrupted media.

Document each project's target, toolchain, build, deployment, expected logs, and
limitations. That turns experiments into repeatable engineering.

# 23. Quick Reference

```bash
# Identity and boot
uname -a
cat /proc/cmdline
cat /proc/meminfo
# Processes and services
ps aux
top
systemctl status service-name
journalctl -u service-name -f
# Devices and storage
ls -l /dev
lsblk -f
df -h
# Network
ip addr
ip route
ss -lntup
# Build artifacts
file output/application
readelf -h output/application
readelf -d output/application
```

## Questions for any new board

1. What executes first, and where does it load the next image?
2. Which exact kernel, device tree, rootfs, and command line are running?
3. Is the failure in power, pins, clocks, bus, interrupt, storage, or software?
4. Which process owns the behavior, and how can it be observed?
5. What happens after power loss, update failure, reboot, or malformed input?

Use the Linux kernel, U-Boot, Device Tree Specification, Buildroot, Yocto, POSIX,
and exact SoC manuals as primary references. Embedded Linux becomes manageable
when hardware facts, build artifacts, boot logs, and test results are recorded
as carefully as the source code.

# 24. Is This Sufficient for Placement?

This note is sufficient as a **roadmap and revision reference**, but reading it
alone is not sufficient for an embedded Linux placement. Employers normally look
for evidence that you can build, boot, debug, and explain a working system.

## Placement-ready checklist

- Strong C: pointers, arrays, structures, function pointers, memory lifetime,
	bitwise operations, concurrency, undefined behavior, and debugging.
- Linux fundamentals: processes, threads, virtual memory, file descriptors,
	signals, sockets, IPC, permissions, `select`/`poll`/`epoll`, and system calls.
- Embedded fundamentals: C startup, linker scripts, memory-mapped I/O,
	interrupts, DMA, buses, GPIO, UART, I2C, SPI, watchdogs, and boot flow.
- Practical Linux: cross-compilation, Git, Make or CMake, shell scripting,
	GDB, `strace`, `dmesg`, serial consoles, and network troubleshooting.
- One build system used deeply: Buildroot or Yocto, including customization.
- One complete project on QEMU or real hardware with source, build steps,
	logs, tests, and a short explanation of design decisions.
- Interview preparation: explain a crash, deadlock, boot failure, driver probe
	failure, memory leak, stack overflow, and power-loss-safe update design.

## A practical placement portfolio

Create one repository containing a cross-compiled C service, a Buildroot or
Yocto image, a device-tree change, a systemd or BusyBox service, a test script,
and a README showing how to reproduce the result. Include a serial boot log,
architecture diagram, resource measurements, and one deliberately diagnosed
failure. This is more persuasive than a list of commands or certificates.

## Suggested study order

1. C, data structures, Linux shell, Git, Make, and debugging.
2. Processes, threads, synchronization, IPC, sockets, and filesystems.
3. ARM or RISC-V architecture, boot flow, cross-compilation, and memory-mapped I/O.
4. QEMU and BusyBox, then a real board with serial, GPIO, and one sensor.
5. Kernel configuration, device tree, drivers, and Buildroot.
6. Yocto, security, real-time measurement, updates, and production testing.

# 25. Recommended Learning Resources

Prefer current official documentation for commands and APIs. Books provide
conceptual depth; hands-on labs and a real board provide the evidence needed for
placement.

## Books

| Resource | Best use |
| --- | --- |
| *Mastering Embedded Linux Programming*, Chris Simmonds | End-to-end embedded Linux, Buildroot, Yocto, kernel, and board work |
| *Embedded Linux Primer*, Christopher Hallinan | Beginner-to-intermediate system architecture and bring-up |
| *Linux Device Drivers*, Jonathan Corbet, Alessandro Rubini, and Greg Kroah-Hartman | Driver concepts; use it with current kernel documentation because the third edition is old |
| *Linux Kernel Development*, Robert Love | Processes, scheduling, memory, and kernel internals |
| *The Linux Programming Interface*, Michael Kerrisk | Excellent Linux system-call and userspace reference |
| *Making Embedded Systems*, Elecia White | Embedded design, testing, reliability, and engineering practice |
| *Embedded Systems Architecture*, Daniele Lacamera | Hardware/software architecture and practical system design |

## Official documentation and free training

- [Linux kernel documentation](https://docs.kernel.org/): current driver APIs,
	device model, filesystems, tracing, locking, and real-time topics.
- [U-Boot documentation](https://docs.u-boot.org/): commands, environment,
	boot flows, FIT images, and board porting.
- [Device Tree Specification](https://devicetree-specification.readthedocs.io/):
	standard device-tree structure and terminology.
- [Buildroot manual](https://buildroot.org/docs.html): reproducible image
	configuration, packages, board support, and external trees.
- [Yocto Project documentation](https://docs.yoctoproject.org/): BitBake,
	recipes, layers, machine configuration, and image creation.
- [Bootlin training materials](https://bootlin.com/training/): free slides and
	labs for embedded Linux, kernel, drivers, Buildroot, Yocto, and debugging.
- [Linux man-pages project](https://man7.org/linux/man-pages/): authoritative
	references for system calls, POSIX APIs, commands, and filesystems.

## Video and conference resources

- [Bootlin on YouTube](https://www.youtube.com/@bootlin): practical kernel,
	driver, Buildroot, Yocto, and embedded Linux training.
- [Embedded Linux Conference](https://www.youtube.com/@LinuxFoundationEvents):
	real project talks covering architecture, debugging, security, and updates.
- [Linux Foundation training](https://training.linuxfoundation.org/): structured
	Linux, kernel, real-time, and embedded courses, including paid options.
- [Digi-Key TechForum](https://www.youtube.com/@DigiKey): accessible hardware,
	buses, microcontrollers, and embedded design explanations.

## Practice platforms and tools

- [QEMU](https://www.qemu.org/): practice booting kernels and root filesystems
	without risking physical hardware.
- [BeagleBoard](https://www.beagleboard.org/), Raspberry Pi, or an affordable
	RISC-V board: practice serial access, GPIO, buses, and image deployment.
- [Kernel Newbies](https://kernelnewbies.org/): approachable kernel learning,
	development guides, and first-contribution material.
- [Bootlin labs](https://bootlin.com/labs/): guided exercises for building and
	debugging embedded Linux components.
- [Linux From Scratch](https://www.linuxfromscratch.org/): useful for learning
	how a Linux system is assembled, although it is not an embedded build system.



## How to use these resources

Read one chapter, implement one small experiment, and write down the observed
boot log or measurement. For every new topic, answer: what problem does it
solve, which layer owns it, how is it configured, how can it fail, and how can
the failure be observed? This turns passive study into placement evidence.

Also refer to proper study plan.
