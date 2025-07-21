# Creating a PRPL Package for Accessing TR-181 Data Model

This guide walks you through creating, building, and deploying a PRPL package to query the TR-181 data model using `libubus` and `libubox`. The package, named `dm_first`, retrieves the `UserNumberOfEntries` from `Device.Users` in the PRPL data model.

## Prerequisites

- A PRPLOS environment (e.g., OpenWrt-based system with PRPL support)
- Development tools: `make`, `cmake`, and dependencies (`libubus`, `libubox`, `libblobmsg-json`)
- SSH/SCP access to the target device
- Basic knowledge of C programming and OpenWrt package structure

## Step 1: Package Structure

The package consists of three main files: a `Makefile` for OpenWrt, a C source file (`dm_first.c`), and a `CMakeLists.txt` for building the executable. Below are the contents of each file, followed by a deployment script.

### Directory Structure

Organize your package in the PRPLOS source tree as follows:

```
prplos/
└── package/
    └── dm_first/
        ├── Makefile
        ├── CMakeLists.txt
        └── dm_first.c
```

Place the files in the `prplos/package/dm_first/` directory.

### Makefile

The `Makefile` defines the package metadata and installation instructions for OpenWrt.

```makefile
include $(TOPDIR)/rules.mk

PKG_NAME:=dm_first
PKG_RELEASE:=1

include $(INCLUDE_DIR)/package.mk
include $(INCLUDE_DIR)/cmake.mk

define Package/${PKG_NAME}
	SECTION:=utils
	CATEGORY:=Utilities
	TITLE:=Check TR-181 ModelName from prpl-datamodel
	DEPENDS:=+libubus +libubox +libblobmsg-json
endef

define Package/${PKG_NAME}/install
	$(INSTALL_DIR) $(1)/usr/bin
	$(INSTALL_BIN) $(PKG_BUILD_DIR)/${PKG_NAME} $(1)/usr/bin/${PKG_NAME}
endef

$(eval $(call BuildPackage,${PKG_NAME}))
```

- **Key Components**:
  - `PKG_NAME`: Package name (`dm_first`).
  - `DEPENDS`: Specifies dependencies (`libubus`, `libubox`, `libblobmsg-json`).
  - `install`: Installs the compiled binary to `/usr/bin`.

### C Source File (dm_first.c)

The C program uses `libubus` to query the `Device.Users` object and parse the `UserNumberOfEntries` field.

```c
#include <libubus.h>
#include <libubox/blobmsg_json.h>
#include <stdio.h>
#include <stdlib.h>

static struct ubus_context *ctx;

static void result_cb(struct ubus_request *req, int type, struct blob_attr *msg)
{
    if (!msg)
    {
        printf("No data received\n");
        return;
    }

    char *str = blobmsg_format_json(msg, true);
    if (str)
    {
        printf("Received data: %s\n", str);
        free(str);
    }

    enum
    {
        Policy_Device_Users,
        __POLICY_MAX
    };

    static const struct blobmsg_policy policy1[] = {
        [Policy_Device_Users] = {
            .name = "Device.Users.", .type = BLOBMSG_TYPE_TABLE}};

    enum
    {
        POLICY_USER_COUNT,
        __USER_POLICY_MAX,
    };
    static const struct blobmsg_policy policy2[] = {
        [POLICY_USER_COUNT] = {
            .name = "UserNumberOfEntries", .type = BLOBMSG_TYPE_INT32},
    };

    struct blob_attr *tb[__POLICY_MAX];
    blobmsg_parse(policy1, __POLICY_MAX, tb, blob_data(msg), blob_len(msg));

    if (tb[Policy_Device_Users])
    {
        struct blob_attr *user_tb[__USER_POLICY_MAX];
        blobmsg_parse(policy2, __USER_POLICY_MAX,
                      user_tb, blobmsg_data(tb[Policy_Device_Users]),
                      blob_len(tb[Policy_Device_Users]));
        
        if(user_tb[POLICY_USER_COUNT]){
            int user_count = blobmsg_get_u32(user_tb[POLICY_USER_COUNT]);
            printf("UserNumberOfEntries %d\n",user_count);
        }else{
            fprintf(stderr, "missing UserNumberOfEntries");
        }
    }
    else
    {
        fprintf(stderr, "missing Device.Users");
    }
}

int main()
{
    uint32_t id;

    ctx = ubus_connect(NULL);
    if (!ctx)
    {
        fprintf(stderr, "Failed to connect to ubus\n");
        return 1;
    }

    // Find object by name
    if (ubus_lookup_id(ctx, "Device.Users", &id) != 0)
    {
        fprintf(stderr, "Failed to find Device.Users\n");
        ubus_free(ctx);
        return 1;
    }

    // Call get method
    if (ubus_invoke(ctx, id, "_get", NULL, result_cb, NULL, 3000) != 0)
    {
        fprintf(stderr, "Failed to invoke get method\n");
    }

    ubus_free(ctx);
    return 0;
}
```

- **Key Components**:
  - Connects to the `ubus` service.
  - Queries the `Device.Users` object using `ubus_lookup_id` and `ubus_invoke`.
  - Parses the response using `blobmsg` to extract `UserNumberOfEntries`.
  - Handles errors for missing objects or fields.

### CMakeLists.txt

The `CMakeLists.txt` configures the build process for the C program.

```cmake
cmake_minimum_required(VERSION 3.10)
project(dm_first C)

add_executable(dm_first dm_first.c)
target_link_libraries(dm_first ubus ubox blobmsg_json)

install(TARGETS dm_first DESTINATION bin)
```

- **Key Components**:
  - Specifies the minimum CMake version.
  - Links against `ubus`, `ubox`, and `blobmsg_json` libraries.
  - Installs the executable to the `bin` directory.

### Build and Deploy Script

The shell script builds the package and deploys it to a remote device.

```bash
#!/bin/bash

# === CONFIG ===
PACKAGE_NAME="dm_first"
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
  - Builds the package using `make package/dm_first/compile`.
  - Locates the generated `.ipk` file.
  - Copies the `.ipk` to the target device via `scp`.
  - Installs the package using `opkg` over SSH.



## Step 2: Building and Deploying

1. Save the build script as `build_and_deploy.sh`.
2. Ensure the PRPLOS device is running and accessible via SSH.
3. Execute the script: `bash build_and_deploy.sh`.
4. On the target device, run the program: `/usr/bin/dm_first`.

## Step 3: Running the Program

The program outputs:
- The raw JSON response from `Device.Users`.
- The `UserNumberOfEntries` value, or an error if the field/object is missing.

## Troubleshooting

- **Build Errors**: Ensure all dependencies (`libubus`, `libubox`, `libblobmsg-json`) are installed in the PRPLOS build environment.
- **Ubus Connection Failure**: Verify that the `ubus` daemon is running on the target device (`ubus status`).
- **Missing Data Model**: Confirm that the `Device.Users` object exists in the TR-181 data model on your device.

## Conclusion

This package demonstrates how to create a simple PRPL package to query the TR-181 data model using `libubus`. You can extend it to retrieve other parameters or integrate it into larger PRPLOS applications.