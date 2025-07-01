# Creating a Ubus Event Listener Package for OpenWRT

This guide outlines how to create, build, and install a custom OpenWRT package for a C program that uses the **ubus** IPC framework to listen for all ubus events. The package, named `neti-status`, registers an event handler to capture and print event types and their JSON payloads. It includes signal handling for clean termination and integrates with OpenWRT’s system using `procd`. After installation, running `neti-status` will display event details, which can be tested by triggering ubus events (e.g., network interface changes).

## Prerequisites

- **OpenWRT Build Environment or SDK**: Set up as per [Module 3](#) or [Module 9](#) (e.g., OpenWRT source or SDK for the target architecture, such as x86_64).
- **Target Device**: An OpenWRT device with SSH access and `apk` package manager.
- **Dependencies**: Ensure `libubus`, `libubox`, and `libblobmsg-json` are available in the build system and on the target device.
- **Development Host**: A Linux system (e.g., Ubuntu) with `make`, `gcc`, and other build tools.

## Step 1: Create the Package Directory Structure

1. **Set Up Directory**:
   - Inside the OpenWRT source or SDK directory, create a directory for the custom package:
     ```bash
     mkdir -p my-packages/neti-status/src
     ```

2. **Directory Structure**:
   ```
   my-packages/neti-status/
   ├── Makefile
   ├── src/
   │   ├── main.c
   ```

## Step 2: Write the C Program with Event Handling

Create the C program file `my-packages/neti-status/src/main.c` with the provided ubus event listener code:

```c
#include <libubox/uloop.h>
#include <libubus.h>
#include <libubox/blobmsg_json.h>
#include <stdio.h>
#include <stdlib.h>
#include <signal.h>

static struct ubus_context *ctx;

static void event_cb(struct ubus_context *ctx, struct ubus_event_handler *ev,
                     const char *type, struct blob_attr *msg)
{
    char *str = NULL;
    fprintf(stderr, "DEBUG: Event %s callback triggered\n", type);
    if (msg) {
        str = blobmsg_format_json(msg, 1);
        fprintf(stderr, "Payload: %s\n", str);
        free(str);
    }
}

static void sig_handler(int sig)
{
    uloop_end();
}

int main(void)
{
    struct ubus_event_handler listener;

    uloop_init();

    ctx = ubus_connect(NULL);
    if (!ctx)
    {
        fprintf(stderr, "Failed to connect to ubus\n");
        return 1;
    }

    ubus_add_uloop(ctx);

    memset(&listener, 0, sizeof(listener));
    listener.cb = event_cb;
    if (ubus_register_event_handler(ctx, &listener, "*"))
    {
        fprintf(stderr, "Failed to register event handler\n");
        ubus_free(ctx);
        return 1;
    }

    signal(SIGINT, sig_handler);
    signal(SIGTERM, sig_handler);

    printf("Listening for events. Press Ctrl+C to stop.\n");
    uloop_run();

    ubus_free(ctx);
    uloop_done();
    return 0;
}
```

### Explanation of Event Handling
- **Event Callback (`event_cb`)**:
  - Triggered for all ubus events (registered with wildcard `*`).
  - Prints the event type (e.g., `interface.up`) and its JSON payload (if any) using `blobmsg_format_json`.
  - Frees the allocated JSON string to prevent memory leaks.
- **Signal Handling**:
  - Handles `SIGINT` (Ctrl+C) and `SIGTERM` to gracefully stop the event loop (`uloop_end`).
- **Ubus Integration**:
  - Connects to `ubusd` and registers a wildcard event handler to capture all events.
  - Uses `uloop` for asynchronous event processing.
- **Dependencies**:
  - Requires `libubox/uloop.h` for the event loop, `libubus.h` for ubus communication, and `libubox/blobmsg_json.h` for JSON formatting.

## Step 3: Create the Package Makefile

Create `my-packages/neti-status/Makefile` to integrate with the OpenWRT build system:

```make
include $(TOPDIR)/rules.mk
include $(INCLUDE_DIR)/package.mk

PKG_NAME:=neti-status
PKG_VERSION:=1.0
PKG_RELEASE:=1
PKG_LICENSE:=GPL-2.0

PKG_BUILD_DIR:=$(BUILD_DIR)/$(PKG_NAME)
SOURCE_DIR:=$(TOPDIR)/my-packages/$(PKG_NAME)/src

define Package/neti-status
    SECTION:=utils
    CATEGORY:=Utilities
    TITLE:=My Ubus Application
    DEPENDS:=+libubus +libubox +libblobmsg-json
endef

define Package/neti-status/description
    A custom application using ubus to listen for all events.
endef

define Build/Prepare
    mkdir -p $(PKG_BUILD_DIR)
    $(CP) $(SOURCE_DIR)/* $(PKG_BUILD_DIR)/
    $(Build/Patch)
endef

define Build/Compile
    $(TARGET_CC) $(TARGET_CFLAGS) -o $(PKG_BUILD_DIR)/$(PKG_NAME) $(PKG_BUILD_DIR)/main.c -lubus -lubox -lblobmsg_json
endef

define Package/neti-status/install
    $(INSTALL_DIR) $(1)/usr/bin
    $(INSTALL_BIN) $(PKG_BUILD_DIR)/$(PKG_NAME) $(1)/usr/bin/
endef

$(eval $(call BuildPackage,neti-status))
```

- **Explanation**:
  - Defines package metadata (`PKG_NAME`, `PKG_VERSION`, etc.).
  - Specifies dependencies (`libubus`, `libubox`, `libblobmsg-json`).
  - Copies source files, compiles `main.c` with the required libraries, and installs the binary to `/usr/bin`.

## Step 4: Include the Package Feed

1. **Update Feeds Configuration**:
   - Create or edit `feeds.conf` in the OpenWRT source or SDK root directory:
     ```bash
     touch feeds.conf
     ```
   - Add the custom feed:
     ```bash
     echo "src-link mypackages $(pwd)/my-packages" >> feeds.conf
     ```

2. **Update and Install Feeds**:
   - Update the feed index and install the package:
     ```bash
     ./scripts/feeds update mypackages
     ./scripts/feeds install -a -p mypackages
     ```
   - Expected output:
     ```
     Installing package 'neti-status' from mypackages
     ```

## Step 5: Build the Package

1. **Configure the Package**:
   - Run `make menuconfig`:
     ```bash
     make menuconfig
     ```
   - Navigate to `Utilities > neti-status` and mark it with `*` to include it.
   - Ensure `libubus`, `libubox`, and `libblobmsg-json` are selected under `Libraries`.
   - Save and exit.

2. **Build the Package**:
   - Compile the package:
     ```bash
     make package/neti-status/compile
     ```
   - Output: A package file named `neti-status_1.0-1_<arch>.apk` in `bin/packages/<arch>/mypackages/`.
     - Example: `bin/packages/x86_64/mypackages/neti-status_1.0-1_x86_64.apk`.

## Step 6: Deploy the Package to OpenWRT

1. **Transfer the Package**:
   - Copy the `.apk` file to the OpenWRT device:
     ```bash
     scp bin/packages/x86_64/mypackages/neti-status_1.0-1_x86_64.apk root@192.168.1.1:/tmp/
     ```

2. **Install the Package**:
   - Install the `neti-status` package:
     ```bash
     cd /tmp
     apk add neti-status_1.0-1_x86_64.apk
     ```

## Step 7: Run and Test the Event Listener

1. **Run the Program**:
   - Start the `neti-status` program:
     ```bash
     /usr/bin/neti-status &
     ```
   - Expected output in the terminal:
     ```
     Listening for events. Press Ctrl+C to stop.
     ```

2. **Test Event Handling**:
   - From another terminal, SSH into the device:
     ```bash
     ssh root@192.168.1.1
     ```
   - Trigger a ubus event (e.g., reload a network interface):
     ```bash
     ubus call network.interface reload '{"interface":"lan"}'
     ```
   - In the terminal running `neti-status`, expect output like:
     ```
     DEBUG: Event interface.up callback triggered
     Payload: {
         "interface": "lan"
     }
     ```
   - Note: The exact event type and payload depend on the triggered action (e.g., `interface.up`, `interface.down`).

3. **Stop the Program**:
   - Press `Ctrl+C` in the terminal running `neti-status` to stop it gracefully.



## Troubleshooting

- **Events Not Captured**:
  - Ensure `neti-status` is running (`ps | grep neti-status`).
  - Check logs for errors: `logread | grep neti-status`.
  - Verify `ubusd` is running: `ps | grep ubusd`.
- **Compilation Errors**:
  - Confirm `libubus`, `libubox`, and `libblobmsg-json` are selected in `make menuconfig` (`Libraries > libubus`, `libubox`, `libblobmsg-json`).
  - Ensure all required headers are included in `main.c`.
- **Package Installation Fails**:
  - Verify the `.apk` matches the device’s architecture (`apk add` will show errors if mismatched).
- **Service Fails to Start**:
  - Check init script permissions: `chmod +x /etc/init.d/neti-status`.
  - Enable verbose logging in the init script (`procd_set_param stderr 1`) and check `logread -f`.

## Notes

- **Architecture**: Ensure the OpenWRT source or SDK matches the target device’s architecture (e.g., x86_64, mips_24kc).
- **Dependencies**: The package requires `libubus`, `libubox`, and `libblobmsg-json`, which must be present on the device.
- **Procd Integration**: The init script makes the service persistent and manageable via `/etc/init.d/`, with logs redirected to `logread`.
- **Debugging**: Use `logread`, `strace`, or `gdb` for debugging (see Module 8.7).
- **Extending Functionality**: The event handler can be modified to filter specific events (e.g., `interface.*`) or process payloads further (e.g., integrate with UCI or trigger actions).

