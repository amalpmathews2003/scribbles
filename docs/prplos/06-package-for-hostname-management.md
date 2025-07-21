# Creating a PRPL Package for Hostname Management with UBUS

This blog post guides you through creating a PRPL package, `get_set_host`, to manage the hostname of a device using the TR-181 data model via UBUS. The package provides two methods: an asynchronous `get_hostname` method to retrieve the hostname and a synchronous `set_hostname` method to update it. We’ll explore the differences between synchronous and asynchronous UBUS methods and provide the complete package structure and code.

## Prerequisites

- PRPLOS environment (e.g., OpenWrt-based system with PRPL support)
- Development tools: `make`, `cmake`, and dependencies (`libubus`, `libubox`, `libblobmsg-json`)
- SSH/SCP access to the target device
- Basic knowledge of C programming, UBUS, and OpenWrt package development

## Step 1: Understanding UBUS Synchronous vs. Asynchronous Methods

UBUS, the OpenWrt micro bus, facilitates communication between processes. It supports both synchronous and asynchronous method calls:

- **Synchronous Methods**: These block until the operation completes, returning the result directly. They are simpler to implement but can cause delays if the operation is slow. In this package, `set_hostname` uses a synchronous call to update the hostname immediately.
- **Asynchronous Methods**: These defer the response, allowing the caller to continue processing while waiting for the result. They are ideal for operations that may take time, such as querying external services. The `get_hostname` method in this package uses an asynchronous approach to retrieve the hostname, leveraging a callback to process the response.

The choice between synchronous and asynchronous depends on the use case. Asynchronous methods are preferred for operations that involve waiting for external responses to avoid blocking the main loop, while synchronous methods suit quick, local operations.

## Step 2: Package Structure

The package includes a `Makefile` for OpenWrt, a `CMakeLists.txt` for building, and C source files (`main.c`, `methods.c`, `methods.h`) to implement the UBUS service.

### Directory Structure

Organize the package in the PRPLOS source tree as follows:

```
prplos/
└── package/
    └── get_set_host/
        ├── Makefile
        └── src/
            ├── CMakeLists.txt
            ├── main.c
            ├── methods.c
            └── methods.h
```

### Makefile

The `Makefile` defines the package metadata and installation instructions.

```makefile
include $(TOPDIR)/rules.mk

PKG_NAME:=get_set_host
PKG_RELEASE:=1

include $(INCLUDE_DIR)/package.mk
include $(INCLUDE_DIR)/cmake.mk

define Package/${PKG_NAME}
	SECTION:=utils
	CATEGORY:=Utilities
	TITLE:=Get & Set Hostname
	DEPENDS:=+libubus +libubox +libblobmsg-json
endef

define Package/${PKG_NAME}/install
	$(INSTALL_DIR) $(1)/usr/bin
	$(INSTALL_BIN) $(PKG_BUILD_DIR)/${PKG_NAME} $(1)/usr/bin/${PKG_NAME}
endef

$(eval $(call BuildPackage,${PKG_NAME}))
```

- **Key Components**:
  - `PKG_NAME`: Package name (`get_set_host`).
  - `DEPENDS`: Specifies dependencies (`libubus`, `libubox`, `libblobmsg-json`).
  - `install`: Installs the binary to `/usr/bin`.

### CMakeLists.txt

The `CMakeLists.txt` configures the build process, enabling debug symbols for development.

```cmake
cmake_minimum_required(VERSION 3.10)
project(get_set_host C)

set(CMAKE_BUILD_TYPE Debug)
set(CMAKE_C_FLAGS_DEBUG "-g3 -ggdb3")

add_executable(get_set_host
    main.c
    methods.c
)

target_include_directories(get_set_host PRIVATE ${CMAKE_CURRENT_SOURCE_DIR})
target_link_libraries(get_set_host PRIVATE ubus ubox blobmsg_json)

install(TARGETS get_set_host DESTINATION bin)
```

- **Key Components**:
  - Links against `ubus`, `ubox`, and `blobmsg_json`.
  - Includes debugging flags for easier troubleshooting.
  - Installs the executable to the `bin` directory.

### main.c

The main program sets up the UBUS service and event loop.

```c
#include <stdio.h>
#include <stdlib.h>

#include "methods.h"

// UBUS methods for get/set hostname
static const struct ubus_method my_methods[] = {
    UBUS_METHOD_NOARG("get_hostname", get_hostname_method),
    UBUS_METHOD("set_hostname", set_hostname_method, set_hostname_policy),
};

static struct ubus_object_type get_set_host_object_type =
    UBUS_OBJECT_TYPE("get_set_host_service", my_methods);

// UBUS object definition
static struct ubus_object get_set_host_object = {
    .name = "get_set_host",
    .type = &get_set_host_object_type,
    .methods = my_methods,
    .n_methods = ARRAY_SIZE(my_methods),
};

int main(void)
{
    struct ubus_context *ctx = ubus_connect(NULL);
    if (!ctx) {
        fprintf(stderr, "Failed to connect to ubus\n");
        return EXIT_FAILURE;
    }

    ubus_add_uloop(ctx);

    if (ubus_add_object(ctx, &get_set_host_object)) {
        fprintf(stderr, "Failed to add ubus object\n");
        ubus_free(ctx);
        return EXIT_FAILURE;
    }

    uloop_run();
    ubus_free(ctx);
    uloop_done();

    return EXIT_SUCCESS;
}
```

- **Key Components**:
  - Defines UBUS methods (`get_hostname`, `set_hostname`) and the object (`get_set_host`).
  - Connects to UBUS and integrates with the `uloop` event loop for handling asynchronous calls.
  - Registers the UBUS object to expose the methods.

### methods.h

The header file declares the method signatures and utility macros.

```c
#ifndef METHODS_H
#define METHODS_H

#include <libubox/blobmsg_json.h>
#include <libubus.h>

struct ubus_async_ctx
{
    struct ubus_context *ctx;
    struct ubus_request_data req;
    void *data; // optional app-specific pointer
};

#define UBUS_DEFER_REQUEST(_ctx, _req, _target)               \
    do                                                        \
    {                                                         \
        (_target) = calloc(1, sizeof(struct ubus_async_ctx)); \
        (_target)->ctx = (_ctx);                              \
        ubus_defer_request((_ctx), (_req), &(_target)->req);  \
    } while (0)

#define UBUS_REPLY_AND_FREE(_ac, _blob_buf)                          \
    do                                                               \
    {                                                                \
        ubus_send_reply((_ac)->ctx, &(_ac)->req, (_blob_buf)->head); \
        blob_buf_free((_blob_buf));                                  \
        ubus_complete_deferred_request((_ac)->ctx, &(_ac)->req, 0);  \
        free((_ac));                                                 \
    } while (0)

// Asynchronous get hostname method
int get_hostname_method(
    struct ubus_context *ctx,
    struct ubus_object *obj,
    struct ubus_request_data *req,
    const char *method,
    struct blob_attr *msg);

// Callback for get_hostname_method
void __get_hostname_cb(
    struct ubus_request *req,
    int type,
    struct blob_attr *msg);

// Synchronous set hostname method
int set_hostname_method(
    structs ubus_context *ctx,
    struct ubus_object *obj,
    struct ubus_request_data *req,
    const char *method,
    struct blob_attr *msg);

// Utility to parse nested string from blobmsg
bool blobmsg_parse_nested_string(
    struct blobの問題attr *msg,
    const char *outer,
    const char *inner,
    const char **result);

extern const struct blobmsg_policy set_hostname_policy[1];

#endif
```

- **Key Components**:
  - Declares the `ubus_async_ctx` structure for asynchronous request handling.
  - Defines macros (`UBUS_DEFER_REQUEST`, `UBUS_REPLY_AND_FREE`) to simplify asynchronous UBUS operations.
  - Specifies method prototypes and the policy for `set_hostname`.

### methods.c

This file implements the `get_hostname` (asynchronous) and `set_hostname` (synchronous) methods.

```c
#include "methods.h"

// Asynchronous method to get hostname via UBUS
int get_hostname_method(
    struct ubus_context *ctx,
    struct ubus_object *obj,
    struct ubus_request_data *req,
    const char *method,
    struct blob_attr *msg)
{
    uint32_t id;
    if (ubus_lookup_id(ctx, "System", &id) != 0) {
        fprintf(stderr, "Failed to find System\n");
        return UBUS_STATUS_UNKNOWN_ERROR;
    }

    struct ubus_async_ctx *async_ctx;
    UBUS_DEFER_REQUEST(ctx, req, async_ctx);
    async_ctx->data = NULL;

    // Invoke _get method asynchronously
    ubus_invoke(ctx, id, "_get", NULL, __get_hostname_cb, async_ctx, 3000);
    return 0;
}

// Callback for get_hostname_method
void __get_hostname_cb(
    struct ubus_request *req,
    int type,
    struct blob_attr *msg)
{
    struct ubus_async_ctx *async_ctx = req->priv;

    char *json = blobmsg_format_json(msg, true);
    if (json) {
        printf("Received UBUS JSON: %s\n", json);
        free(json);
    }

    struct blob_buf b = {};
    blob_buf_init(&b, 0);

    const char *hostname = NULL;
    if (!blobmsg_parse_nested_string(msg, "System.", "HostName", &hostname)) {
        return;
    }

    blobmsg_add_string(&b, "hostname", hostname ? hostname : "unknown");
    UBUS_REPLY_AND_FREE(async_ctx, &b);
}

// Utility to parse nested string from blobmsg
bool blobmsg_parse_nested_string(
    struct blob_attr *msg,
    const char *outer,
    const char *inner,
    const char **result)
{
    struct blobmsg_policy outer_policy[] = {
        {.name = outer, .type = BLOBMSG_TYPE_TABLE}
    };
    struct blob_attr *outer_tb[ARRAY_SIZE(outer_policy)];
    if (blobmsg_parse(outer_policy, ARRAY_SIZE(outer_policy), outer_tb,
                      blob_data(msg), blob_len(msg)) < 0 || !outer_tb[0])
        return false;

    struct blobmsg_policy inner_policy[] = {
        {.name = inner, .type = BLOBMSG_TYPE_STRING}
    };
    struct blob_attr *inner_tb[ARRAY_SIZE(inner_policy)];
    if (blobmsg_parse(inner_policy, ARRAY_SIZE(inner_policy), inner_tb,
                      blobmsg_data(outer_tb[0]), blobmsg_len(outer_tb[0])) < 0 || !inner_tb[0])
        return false;

    *result = blobmsg_get_string(inner_tb[0]);
    return true;
}

const struct blobmsg_policy set_hostname_policy[] = {
    { .name = "hostname", .type = BLOBMSG_TYPE_STRING }
};

// Synchronous method to set hostname via UBUS
int set_hostname_method(
    struct ubus_context *ctx,
    struct ubus_object *obj,
    struct ubus_request_data *req,
    const char *method,
    struct blob_attr *msg)
{
    enum { HOSTNAME, __MAX };
    struct blob_attr *tb[__MAX];
    blobmsg_parse(set_hostname_policy, __MAX, tb, blob_data(msg), blob_len(msg));

    if (!tb[HOSTNAME]) {
        fprintf(stderr, "Missing 'hostname' parameter\n");
        return UBUS_STATUS_INVALID_ARGUMENT;
    }

    const char *hostname = blobmsg_get_string(tb[HOSTNAME]);
    printf("Setting hostname to %s\n", hostname);

    struct blob_buf b = {};
    blob_buf_init(&b, 0);
    void *t = blobmsg_open_table(&b, "parameters");
    blobmsg_add_string(&b, "HostName", hostname);
    blobmsg_close_table(&b, t);

    uint32_t id;
    if (ubus_lookup_id(ctx, "System", &id) != 0) {
        fprintf(stderr, "Failed to find System\n");
        blob_buf_free(&b);
        return UBUS_STATUS_NOT_FOUND;
    }

    int ret = ubus_invoke(ctx, id, "_set", b.head, NULL, NULL, 3000);
    blob_buf_free(&b);
    if (ret != 0) {
        fprintf(stderr, "Failed to invoke _set on System: %d\n", ret);
        return UBUS_STATUS_UNKNOWN_ERROR;
    }

    return UBUS_STATUS_OK;
}
```

- **Key Components**:
  - **`get_hostname_method`**: Asynchronously queries the `System` object’s `_get` method, deferring the response using `UBUS_DEFER_REQUEST`. The callback (`__get_hostname_cb`) parses the `HostName` field and sends the reply.
  - **`set_hostname_method`**: Synchronously invokes the `System` object’s `_set` method with the provided hostname, returning the result immediately.
  - **`blobmsg_parse_nested_string`**: A utility function to parse nested strings from the TR-181 data model.

### Build and Deploy Script

The following script builds the package and deploys it to a remote device.

```bash
#!/bin/bash

# === CONFIG ===
PACKAGE_NAME="get_set_host"
TARGET_USER="root"
PRPL_DIR="./prplos"
REMOTE_IP="127.0.0.1"
REMOTE_PORT=1122
REMOTE_PATH="/tmp"

# === BUILD ===
echo "[*] Building package $PACKAGE_NAME..."
cd "$PRPL_DIR" || exit 1

make package/$PACKAGE_NAME/compile V=s || { echo "[!] Build failed"; exit 1; }

# === FIND IPK ===
echo "[*] Searching for .ipk file..."
IPK_PATH=$(find bin/packages/ -type f -name "${PACKAGE_NAME}_*.ipk" | head -n 1)

if [ -z "$IPK_PATH" ]; then
    echo "[!] .ipk file not found."
    exit 1
fi

echo "[*] Found IPK: $IPK_PATH"

# === SCP TO DEVICE ===
echo "[*] Copying to $REMOTE_IP:$REMOTE_PATH"
scp -O -P "$REMOTE_PORT" "$IPK_PATH" "$TARGET_USER@$REMOTE_IP:$REMOTE_PATH" || {
    echo "[!] SCP failed"
    exit 1
}

IPK_FILENAME=$(basename "$IPK_PATH")

# === REMOVE AND INSTALL ===
ssh -p "$REMOTE_PORT" "$TARGET_USER@$REMOTE_IP" <<EOF
opkg remove $PACKAGE_NAME || true
opkg install $REMOTE_PATH/$IPK_FILENAME
EOF

echo "[✓] Package $PACKAGE_NAME reinstalled on $REMOTE_IP"
```

- **Key Components**:
  - Builds the package using `make`.
  - Locates the generated `.ipk` file.
  - Transfers the `.ipk` to the target device via `scp`.
  - Installs the package using `opkg` over SSH.

## Step 3: Building and Deploying

1. Save the script as `build_and_deploy.sh` and place all files in `prplos/package/get_set_host/`.
2. Ensure the PRPLOS device is running and accessible via SSH.
3. Run the script: `bash build_and_deploy.sh`.
4. The binary will be installed at `/usr/bin/get_set_host` on the target device.

## Step 4: Testing the Package

Use the `ubus` CLI to interact with the service:

- **Get Hostname** (Asynchronous):
  ```bash
  ubus call get_set_host get_hostname
  ```
  Output example:
  ```json
  {
    "hostname": "mydevice"
  }
  ```

- **Set Hostname** (Synchronous):
  ```bash
  ubus call get_set_host set_hostname '{"hostname": "newhostname"}'
  ```
  Output: Success (`UBUS_STATUS_OK`) or an error code if the operation fails.

The asynchronous `get_hostname` method allows the UBUS event loop to handle other tasks while waiting for the `System` object’s response, while the synchronous `set_hostname` method ensures immediate confirmation of the hostname update.

## Troubleshooting

- **Build Errors**: Verify that `libubus`, `libubox`, and `libblobmsg-json` are installed in the PRPLOS build environment.
- **UBUS Connection Issues**: Ensure the `ubus` daemon is running (`ubus status`).
- **Missing System Object**: Confirm that the `System` object and `HostName` parameter exist in the TR-181 data model.
- **Asynchronous Callback Issues**: Check debug logs (enabled via `CMAKE_BUILD_TYPE=Debug`) for parsing or response errors.

## Conclusion

This `get_set_host` package demonstrates how to use UBUS synchronous and asynchronous methods in a PRPL package to manage the device hostname. The asynchronous `get_hostname` method leverages callbacks to handle potentially slow queries efficiently, while the synchronous `set_hostname` method provides immediate feedback for quick operations. You can extend this package to manage other TR-181 parameters or integrate it into larger PRPLOS applications.