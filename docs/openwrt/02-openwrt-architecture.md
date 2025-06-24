# OpenWRT Architecture

## Layered Architecture

OpenWRT’s architecture is designed for embedded systems, particularly routers, with a modular and lightweight structure. It can be conceptualized as a layered stack:

- **Hardware Layer**: System-on-Chip (SoC), flash storage, RAM, network interfaces, and wireless radios.
- **Kernel Layer**: Linux kernel (customized for embedded use), providing core OS functionality, drivers, and hardware abstraction.
- **System Layer**: Core utilities and libraries (e.g., BusyBox, musl/uClibc), forming the minimal user-space environment.
- **Middleware Layer**: OpenWRT-specific components like UCI (Unified Configuration Interface), netifd (network interface daemon), procd (process daemon), and ubus (inter-process communication bus).
- **Application Layer**: User-installed packages (e.g., firewall, VPN, web servers) and services managed via opkg.
- **User Interface Layer**: Configuration tools like LuCI (web-based GUI), command-line interfaces, or custom scripts.

This layered approach ensures modularity, resource efficiency, and flexibility for networking tasks.

## Control Plane vs Data Plane

In OpenWRT, networking tasks are divided into control plane and data plane operations:

### Control Plane:
- Manages network configuration, routing decisions, and policy enforcement.
- Operates in user space and kernel space, involving daemons like netifd, routing protocols (e.g., BGP, OSPF), and firewall rules.
- Examples: Configuring IP addresses, setting up VLANs, or managing QoS policies.
- Typically slower, as it involves software processing and decision-making.

### Data Plane:
- Handles packet forwarding and processing at high speed.
- Operates primarily in the kernel or hardware (e.g., switch chips or wireless drivers).
- Examples: Packet switching, NAT, or hardware-accelerated encryption.
- Optimized for performance, often leveraging SoC-specific hardware offloading.

OpenWRT balances these planes by offloading data plane tasks to hardware (when supported) while providing flexible control plane tools for customization.

## High-Level Software Stack

The OpenWRT software stack is tailored for embedded networking devices:

- **Linux Kernel**: Stripped-down kernel (e.g., Linux 5.x in recent versions) with patches for embedded hardware, networking, and drivers.
- **C Library**: musl or uClibc for lightweight standard library functions, replacing glibc to save resources.
- **BusyBox**: Provides a minimal set of Unix utilities (e.g., ls, cat, grep) in a single binary to reduce storage.
- **Core Daemons**:
  - **netifd**: Manages network interfaces and configurations.
  - **procd**: System init and process manager, replacing traditional init systems.
  - **ubus**: Lightweight IPC (inter-process communication) for system components.
- **Configuration Layer**:
  - **UCI**: Centralized configuration system for network, firewall, and services.
  - **LuCI**: Web-based GUI for user-friendly configuration.
- **Packages**: Extensible software (e.g., OpenVPN, dnsmasq) installed via opkg.
- **Filesystem**: Combines SquashFS (read-only) and OverlayFS (writable) for efficient storage.

This stack prioritizes minimal resource usage while supporting advanced networking features.

## Role of BusyBox, UCI, Netifd, Procd, UBUS

### BusyBox:
- A multi-tool binary providing essential Unix commands (e.g., sh, cp, tar) in a compact form.
- Reduces flash and RAM usage by combining tools into one executable.
- **Role**: Forms the core user-space environment for basic system operations.

### UCI (Unified Configuration Interface):
- A centralized configuration system for managing OpenWRT settings.
- Stores configurations in text files under `/etc/config/` (e.g., `/etc/config/network`).
- Provides a simple syntax and CLI (`uci` command) for scripting and automation.
- **Role**: Simplifies configuration of network, firewall, and services, ensuring consistency.

### Netifd (Network Interface Daemon):
- Manages network interfaces, protocols, and connectivity.
- Processes UCI network configurations and interacts with kernel networking stack.
- Supports dynamic changes (e.g., hotplugging interfaces) and protocols like DHCP, PPPoE.
- **Role**: Central hub for network setup and runtime management.

### Procd (Process Daemon):
- OpenWRT’s init system and process supervisor, replacing traditional SysVinit or systemd.
- Handles system boot, service management, and hotplug events.
- Lightweight and tailored for embedded systems.
- **Role**: Manages system startup, service lifecycle, and event-driven tasks.

### UBUS (Micro Bus):
- A lightweight IPC mechanism for inter-process communication.
- Allows daemons (e.g., netifd, LuCI) to exchange messages and trigger actions.
- Supports publish/subscribe and remote procedure calls (RPC).
- **Role**: Enables modular communication between system components.

## Filesystem Structure (OverlayFS, SquashFS, etc.)

OpenWRT’s filesystem is optimized for embedded devices with limited flash storage:

### SquashFS:
- A compressed, read-only filesystem for the root filesystem (`/rom`).
- Stores the base system (kernel, core utilities, default configs).
- Ensures immutability and efficient storage.

### OverlayFS:
- A writable overlay filesystem mounted on `/overlay`.
- Combines with SquashFS to create a unified root (`/`) where changes are stored in `/overlay`.
- Persists user configurations, installed packages, and logs.

### JFFS2 (Journaling Flash File System 2):
- Used in older devices or specific partitions for writable storage.
- Less common in modern OpenWRT due to OverlayFS adoption.

### Key Directories:
- `/rom`: Read-only base system (SquashFS).
- `/overlay`: Writable user data.
- `/etc/config`: UCI configuration files.
- `/usr/lib/opkg`: Package metadata for opkg.
- `/tmp`: RAM-based temporary storage (volatile).

This structure allows firmware upgrades while preserving user settings and supports devices with as little as 4-8 MB of flash.

## Configuration Flow and Boot Lifecycle

### Configuration Flow:
- Configurations are stored in `/etc/config/` as UCI files (e.g., network, firewall).
- UCI parses these files, and tools like netifd or firewall apply settings.
- Users modify settings via CLI (`uci` command), LuCI, or scripts.
- Changes are applied dynamically or after a service restart (e.g., `/etc/init.d/network restart`).

### Boot Lifecycle:
- **Bootloader (U-Boot)**: Loads the kernel and initial ramdisk (initramfs) from flash.
- **Kernel Initialization**: Mounts SquashFS (`/rom`) and OverlayFS (`/`) filesystems.
- **Procd Startup**: Starts as PID 1, initializes system services.
- **Ubus Initialization**: Sets up IPC for daemons to communicate.
- **Netifd Startup**: Applies network configurations from `/etc/config/network`.
- **Service Initialization**: Starts enabled services (e.g., firewall, dnsmasq) via `/etc/init.d/`.
- **System Ready**: System is fully operational, accepting user input via SSH, LuCI, or CLI.

The boot process is optimized to complete in seconds, even on low-power hardware.

## Integration Points with SoC SDK (Board Support Packages)

OpenWRT relies on SoC SDKs and Board Support Packages (BSPs) to support diverse hardware:

### SoC SDK:
- Provided by chip vendors (e.g., Qualcomm, MediaTek, Broadcom).
- Includes kernel patches, drivers, and tools for SoC-specific features (e.g., Wi-Fi, switch chips, hardware NAT).
- OpenWRT integrates these into its build system to create device-specific firmware.

### Board Support Packages (BSPs):
- Define hardware-specific configurations (e.g., GPIO mappings, flash layout, LED controls).
- Stored in OpenWRT’s source tree under `target/linux/<platform>/`.
- Example: `target/linux/ath79` for Qualcomm Atheros-based routers.

### Integration Points:
- **Kernel Drivers**: OpenWRT customizes the Linux kernel with SoC drivers for Ethernet, Wi-Fi, or crypto acceleration.
- **Device Tree (DTS)**: Describes hardware layout (e.g., CPU, memory, peripherals) for modern SoCs.
- **Firmware Build**: OpenWRT’s buildroot system compiles firmware images tailored to specific boards, incorporating SDK/BSP components.
- **Hardware Offloading**: Leverages SoC features (e.g., switch chips for VLANs) to optimize data plane performance.
- **Proprietary Blobs**: Some SoCs require closed-source firmware (e.g., for Wi-Fi radios), which OpenWRT includes when necessary.

### Challenges:
- Limited vendor support for open-source drivers.
- Variability in SDK quality, requiring community patches.
- Ensuring compatibility across diverse hardware.

OpenWRT’s build system abstracts these complexities, allowing developers to target new devices by adding BSPs.