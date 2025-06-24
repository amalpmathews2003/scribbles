
# Code Flow and System Interaction

This guide examines the end-to-end data flow in OpenWRT, from user input to hardware interaction, covering network, firewall, boot processes, and a sample ubus application.

## From CLI/API to SoC Layer: End-to-End Data Flow

OpenWRT’s architecture facilitates a seamless flow from user input (CLI or API) to hardware-level operations on the System-on-Chip (SoC).

### Overview:
- User inputs via CLI (`uci`, `ubus`), LuCI (web GUI), or JSON-RPC API trigger system actions.
- These inputs interact with middleware (UCI, ubus, netifd) and propagate to the kernel and SoC.

### Data Flow:
1. **User Input**:
   - **CLI**: Commands like `uci set network.lan.ipaddr='192.168.1.100'` or `ubus call network reload`.
   - **LuCI**: Web interface sends JSON-RPC requests via rpcd.
   - **API**: External tools send JSON-RPC requests (e.g., `curl` to `/ubus`).
2. **Configuration Layer (UCI)**:
   - CLI/API modifies UCI files (e.g., `/etc/config/network`).
   - Example: `uci commit network` saves changes.
3. **Middleware (ubus, netifd)**:
   - Ubus routes requests to daemons (e.g., `ubus call network reload` notifies netifd).
   - Netifd parses UCI configs and applies changes (e.g., sets IP addresses).
4. **Kernel Interaction**:
   - Netifd uses netlink to configure kernel networking stack (e.g., `ip link`, `ip addr`).
   - Kernel drivers (e.g., Ethernet, Wi-Fi) handle low-level operations.
5. **SoC Layer**:
   - Kernel drivers interact with SoC hardware (e.g., Ethernet PHY, switch chip).
   - Hardware offloading (e.g., NAT, VLANs) is enabled if supported by the SoC.

### Example (Setting LAN IP):
- Command: `uci set network.lan.ipaddr='192.168.1.100'; uci commit network; ubus call network reload`.
- Flow:
  1. UCI updates `/etc/config/network`.
  2. Ubus notifies netifd via `network.reload`.
  3. Netifd uses netlink to set IP on `br-lan` (kernel).
  4. Kernel configures the SoC’s Ethernet controller.

### Key Components:
- UCI: Configuration storage.
- Ubus: IPC for signaling.
- Netifd: Network management.
- Kernel/SoC: Hardware execution.

## Network Interface Bring-Up Flow (ifup, hotplug, netifd)

Bringing up a network interface in OpenWRT involves **ifup**, **hotplug**, and **netifd**.

### Components:
- **ifup**: Shell script (`/sbin/ifup`) that triggers interface activation via UCI and netifd.
- **Hotplug**: Event-driven system for dynamic interface changes (e.g., USB modem insertion).
- **Netifd**: Central daemon for network configuration and management.

### Flow:
1. **Trigger**:
   - Manual: `ifup lan` or `ubus call network.interface up '{"interface":"lan"}'`.
   - Automatic: Hotplug event (e.g., plugging an Ethernet cable).
2. **UCI Configuration**:
   - Netifd reads `/etc/config/network` for the interface (e.g., `config interface 'lan'`).
   - Example:
     ```bash
     config interface 'lan'
         option proto 'static'
         option ipaddr '192.168.1.1'
         option netmask '255.255.255.0'
         option device 'br-lan'
     ```
3. **Netifd Processing**:
   - Netifd resolves the device (e.g., `br-lan` → `eth0.1`).
   - Configures protocol (e.g., static IP, DHCP) using protocol handlers.
   - Uses netlink to set kernel parameters (e.g., `ip link set br-lan up`).
4. **Hotplug Integration**:
   - Kernel events (e.g., device detected) trigger `/etc/hotplug.d/` scripts.
   - Example: `/etc/hotplug.d/iface/00-netstate` notifies netifd of interface changes.
5. **Ubus Notification**:
   - Netifd publishes events (e.g., `interface.up`) to ubus.
   - Other components (e.g., firewall, LuCI) react to these events.

### Example (Bringing up LAN):
- Command: `ifup lan`.
- Flow:
  1. `ifup` calls `ubus call network.interface up '{"interface":"lan"}'`.
  2. Netifd reads UCI config and configures `br-lan`.
  3. Kernel activates the interface, and SoC hardware handles traffic.
  4. Ubus event `interface.up` notifies subscribers.

### Hotplug Example:
- Plugging a USB modem triggers `/etc/hotplug.d/usb/` scripts.
- Netifd detects the new interface (e.g., `wwan0`) and applies UCI config.

## Firewall and Packet Filtering (nftables integration)

OpenWRT uses **nftables** (since version 22.03) for firewall and packet filtering, replacing iptables.

### Role:
- Manages packet filtering, NAT, and traffic shaping.
- Configured via UCI (`/etc/config/firewall`) and applied by the `firewall4` daemon.

### Internals:
- **Firewall4**: Daemon that translates UCI configs to nftables rules.
- **Nftables**: Kernel-based framework for packet processing, using netlink to interact with the kernel.
- **Ubus Integration**: Firewall4 exposes status and control via ubus (e.g., `ubus call firewall reload`).

### Configuration:
- Stored in `/etc/config/firewall`.
- Example (default LAN zone):
  ```bash
  config zone
      option name 'lan'
      option input 'ACCEPT'
      option output 'ACCEPT'
      option forward 'ACCEPT'
      list network 'lan'
  ```
- Applied via:
  ```bash
  /etc/init.d/firewall reload
  ```

### Flow:
1. **UCI Config**: User modifies `/etc/config/firewall` (e.g., add NAT rule).
2. **Firewall4**: Parses UCI and generates nftables rules.
3. **Kernel**: Nftables applies rules to the network stack.
4. **SoC**: Hardware offloading (if supported) accelerates packet processing.

### Example (Adding a port forward):
- UCI config:
  ```bash
  config redirect
      option name 'webserver'
      option src 'wan'
      option src_dport '80'
      option dest 'lan'
      option dest_ip '192.168.1.100'
      option dest_port '80'
  ```
- Apply: `/etc/init.d/firewall reload`.
- Firewall4 generates nftables rule:
  ```bash
  nft add rule ip nat PREROUTING iifname "wan" tcp dport 80 dnat to 192.168.1.100:80
  ```

### Key Features:
- Zones: Group interfaces (e.g., `lan`, `wan`) for policy application.
- NAT: Source/destination NAT for port forwarding or masquerading.
- Ubus Events: Firewall4 notifies changes (e.g., `firewall.reload`).

## Boot Sequence and Init Scripts (Procd rc.d)

The boot sequence in OpenWRT is managed by **procd**, using init scripts in `/etc/rc.d/`.

### Boot Sequence:
1. **Bootloader (U-Boot)**: Loads kernel and initramfs from flash.
2. **Kernel Initialization**: Mounts filesystems (SquashFS, OverlayFS).
3. **Procd (PID 1)**: Starts as the init process.
4. **Early Init**: Executes `/etc/preinit` for low-level setup (e.g., mounting `/tmp`).
5. **Ubus Startup**: Starts `ubusd` for IPC.
6. **Init Scripts**: Procd runs scripts in `/etc/rc.d/` in order:
   - Scripts prefixed with `S` (start) or `K` (stop) and a number (e.g., `S10network`).
   - Example: `/etc/rc.d/S10network start` starts netifd.
7. **Service Initialization**: Daemons like netifd, firewall4, and dnsmasq start.
8. **System Ready**: OpenWRT is fully operational.

### Init Scripts:
- Located in `/etc/init.d/` and symlinked to `/etc/rc.d/`.
- Use procd-specific functions (`USE_PROCD=1`).
- Example:
  ```bash
  #!/bin/sh /etc/rc.common
  USE_PROCD=1
  START=50
  start_service() {
      procd_open_instance
      procd_set_param command /usr/sbin/myservice
      procd_close_instance
  }
  ```
- Commands: `start`, `stop`, `reload`, `enable`, `disable`.

### Execution:
- Procd runs scripts in numerical order (e.g., `S10network` before `S20firewall`).
- Hotplug events trigger additional scripts post-boot.

## Code Walkthrough: A Sample UBUS Application

Below is a sample ubus application in C, implementing a server and client for a custom service.

### Server (custom_service.c):
```c
#include <libubus.h>
#include <libubox/blobmsg_json.h>
#include <stdio.h>

static int greet_method(struct ubus_context *ctx, struct ubus_object *obj,
                        struct ubus_request_data *req, const char *method,
                        struct blob_attr *msg) {
    struct blob_buf b = {};
    blob_buf_init(&b, 0);
    blobmsg_add_string(&b, "greeting", "Hello from ubus!");
    ubus_send_reply(ctx, req, b.head);
    blob_buf_free(&b);
    return 0;
}

static const struct ubus_method methods[] = {
    UBUS_METHOD("greet", greet_method, NULL),
};

static struct ubus_object_type obj_type = UBUS_OBJECT_TYPE("custom", methods);

static struct ubus_object obj = {
    .name = "custom",
    .type = &obj_type,
    .methods = methods,
    .n_methods = ARRAY_SIZE(methods),
};

int main() {
    struct ubus_context *ctx;
    ctx = ubus_connect(NULL);
    if (!ctx) {
        fprintf(stderr, "Failed to connect to ubus\n");
        return -1;
    }

    if (ubus_add_object(ctx, &obj)) {
        fprintf(stderr, "Failed to add object\n");
        ubus_free(ctx);
        return -1;
    }

    uloop_init();
    ubus_add_uloop(ctx);
    uloop_run();
    ubus_free(ctx);
    uloop_done();
    return 0;
}
```
- **Explanation**:
  - Registers a `custom` object with a `greet` method.
  - Responds with a JSON message: `{"greeting": "Hello from ubus!"}`.
  - Uses `uloop` for event handling.

### Client (custom_client.c):
```c
#include <libubus.h>
#include <libubox/blobmsg_json.h>
#include <stdio.h>

static void receive_result(struct ubus_request *req, int type, struct blob_attr *msg) {
    char *json = blobmsg_format_json(msg, true);
    printf("Response: %s\n", json);
    free(json);
}

int main() {
    struct ubus_context *ctx;
    uint32_t id;

    ctx = ubus_connect(NULL);
    if (!ctx) {
        fprintf(stderr, "Failed to connect to ubus\n");
        return -1;
    }

    if (ubus_lookup_id(ctx, "custom", &id)) {
        fprintf(stderr, "Failed to lookup custom object\n");
        ubus_free(ctx);
        return -1;
    }

    struct blob_buf b = {};
    blob_buf_init(&b, 0);
    ubus_invoke(ctx, id, "greet", b.head, receive_result, NULL, 1000);

    uloop_init();
    ubus_add_uloop(ctx);
    uloop_run();
    blob_buf_free(&b);
    ubus_free(ctx);
    uloop_done();
    return 0;
}
```
- **Explanation**:
  - Connects to ubus and calls `custom.greet`.
  - Prints the response from the server.

### Compilation:
```bash
gcc -o custom_service custom_service.c -lubus -lubox
gcc -o custom_client custom_client.c -lubus -lubox
```

### Running:
- Start server: `./custom_service &`.
- Run client: `./custom_client`.
- Output: `Response: {"greeting":"Hello from ubus!"}`.

## Service Startup, Watchdog, and Daemonization

OpenWRT uses **procd** to manage service startup, monitoring, and daemonization.

### Service Startup:
- Services are defined in `/etc/init.d/` with procd-compatible scripts.
- Enabled services are symlinked to `/etc/rc.d/` (e.g., `S10network`).
- Procd executes `start_service()` during boot or manual start:
  ```bash
  /etc/init.d/myservice start
  ```

### Watchdog:
- Procd monitors services for crashes (if `procd_set_param respawn` is set).
- Example:
  ```bash
  procd_set_param respawn 3600 5 0  # Retry every 5s, max 3600s
  ```
- Kernel watchdog support (if enabled) ensures system recovery on freezes:
  ```bash
  echo 1 > /proc/sys/kernel/sysrq
  ```

### Daemonization:
- Procd daemonizes services by default, running them in the background.
- Example init script:
  ```bash
  #!/bin/sh /etc/rc.common
  USE_PROCD=1
  START=50
  start_service() {
      procd_open_instance
      procd_set_param command /usr/sbin/myservice --daemon
      procd_set_param respawn
      procd_close_instance
  }
  ```
- Procd handles PID files, logging, and restarts.

### Interaction:
- Services publish status via ubus (e.g., `ubus call service list`).
- Managed via `/etc/init.d/<service> {start|stop|reload}`.
