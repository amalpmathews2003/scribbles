
# Creating and Installing a Ubus-Based OpenWRT Package

This guide outlines how to create, build, and install a custom OpenWRT package for a C program that uses the **ubus** IPC framework to register a service named `myservice`. The program exposes a `say_hello` method, and after installation, `ubus list` on the OpenWRT device should list `myservice`. 

## Prerequisites

- **OpenWRT Build Environment or SDK**: Set up as per [Module 3](#) or [Module 9](#) (e.g., OpenWRT source or SDK for the target architecture, such as x86_64).
- **Target Device**: An OpenWRT device with SSH access and `opkg` package manager.
- **Dependencies**: Ensure `libubus` and `libubox` are available in the build system and on the target device.
- **Development Host**: A Linux system (e.g., Ubuntu) with `make`, `gcc`, and other build tools.

## Step 1: Create the Package Directory Structure

1. **Set Up Directory**:
   - Inside the OpenWRT source or SDK directory, create a directory for the custom package:
     ```bash
     mkdir -p package/myservice/src
     ```

2. **Directory Structure**:
   ```
   package/myservice/
   ├── Makefile
   ├── src/
   │   ├── Makefile
   │   ├── myservice.c
   ```

## Step 2: Write the C Program

Create the C program file `package/myservice/src/myservice.c` with the provided ubus code:

```c
#include <libubus.h>
#include <libubox/ustream.h>

static struct ubus_context *ctx;
static struct blob_buf b;

static int hello_method(
    struct ubus_context *ctx,
    struct ubus_object *obj,
    struct ubus_request_data *req,
    const char *method,
    struct blob_attr *msg)
{
    blob_buf_init(&b, 0);
    blobmsg_add_string(&b, "message", "Hello from C");

    ubus_send_reply(ctx, req, b.head);
    return 0;
}

static const struct ubus_method my_methods[] = {
    UBUS_METHOD_NOARG("say_hello", hello_method),
};

static struct ubus_object_type my_object_type = UBUS_OBJECT_TYPE("my_service", my_methods);

static struct ubus_object my_object = {
    .name = "myservice",
    .type = &my_object_type,
    .methods = my_methods,
    .n_methods = ARRAY_SIZE(my_methods),
};

int main()
{
    uloop_init();
    ctx = ubus_connect(NULL);

    if (!ctx) {
        fprintf(stderr, "Failed to connect to ubus\n");
        return -1;
    }

    ubus_add_uloop(ctx);

    if (ubus_add_object(ctx, &my_object)) {
        fprintf(stderr, "Failed to add ubus object\n");
        ubus_free(ctx);
        return -1;
    }

    printf("UBus service running as 'myservice'.\n");

    uloop_run();
    ubus_free(ctx);
    uloop_done();

    return 0;
}
```

- **Explanation**:
  - Registers a ubus object named `myservice` with a `say_hello` method.
  - The method returns a JSON response: `{"message": "Hello from C"}`.
  - Uses `uloop` for event handling and connects to `ubusd` via `libubus`.

## Step 3: Create the Source Makefile

Create `package/myservice/src/Makefile` to compile the C program:

```make
all: myservice

myservice: myservice.c
	$(CC) $(CFLAGS) -o myservice myservice.c -lubus -lubox

clean:
	rm -f myservice *.o
```

- **Explanation**:
  - Compiles `myservice.c` into the `myservice` binary.
  - Links against `libubus` and `libubox` for ubus functionality.

## Step 4: Create the Package Makefile

Create `package/myservice/Makefile` to integrate with the OpenWRT build system:

```make
include $(TOPDIR)/rules.mk

# Name, version, and release number
PKG_NAME:=myservice
PKG_VERSION:=1.0
PKG_RELEASE:=1
PKG_LICENSE:=GPL-2.0

PKG_BUILD_DIR:=$(BUILD_DIR)/$(PKG_NAME)

include $(INCLUDE_DIR)/package.mk

define Package/myservice
    SECTION:=utils
    CATEGORY:=Utilities
    TITLE:=My Ubus Service
    DEPENDS:=+libubus +libubox
endef

define Package/myservice/description
    A simple ubus-based service exposing a 'say_hello' method.
endef

define Build/Prepare
    mkdir -p $(PKG_BUILD_DIR)
    $(CP) ./src/* $(PKG_BUILD_DIR)/
endef

define Build/Compile
    $(MAKE) -C $(PKG_BUILD_DIR) \
        CC="$(TARGET_CC)" \
        CFLAGS="$(TARGET_CFLAGS)" \
        LDFLAGS="$(TARGET_LDFLAGS)"
endef

define Package/myservice/install
    $(INSTALL_DIR) $(1)/usr/bin
    $(INSTALL_BIN) $(PKG_BUILD_DIR)/myservice $(1)/usr/bin/
endef

$(eval $(call BuildPackage,myservice))
```

- **Explanation**:
  - Defines package metadata (`PKG_NAME`, `PKG_VERSION`, etc.).
  - Specifies dependencies (`libubus`, `libubox`).
  - Copies source files, compiles using the source `Makefile`, and installs the binary to `/usr/bin`.

## Step 5: Build the Package

1. **Update Feeds** (if using a custom feed):
   - If the package is in a custom feed (e.g., `my-packages/`), update `feeds.conf`:
     ```bash
     echo "src-link mypackages $(pwd)/package" >> feeds.conf
     ```
   - Update and install feeds:
     ```bash
     ./scripts/feeds update mypackages
     ./scripts/feeds install -a -p mypackages
     ```
   - Note: If the package is in the OpenWRT source’s `package/` directory, skip this step.

2. **Configure the Package**:
   - Run `make menuconfig`:
     ```bash
     make menuconfig
     ```
   - Navigate to `Utilities > myservice` and mark it with `*` to include it.
   - Save and exit.

3. **Build the Package**:
   - Compile the package:
     ```bash
     make package/myservice/compile
     ```
   - Output: A package file named `myservice_1.0-1_<arch>.ipk` in `bin/packages/<arch>/base/` (or `bin/packages/<arch>/mypackages/` if using a custom feed).
     - Example: `bin/packages/x86_64/base/myservice_1.0-1_x86_64.ipk`.

## Step 6: Deploy the Package to OpenWRT

1. **Transfer the Package**:
   - Copy the `.ipk` file to the OpenWRT device:
     ```bash
     scp bin/packages/x86_64/base/myservice_1.0-1_x86_64.ipk root@192.168.1.1:/tmp/
     ```

2. **Install Dependencies**:
   - SSH into the device and ensure `libubus` and `libubox` are installed:
     ```bash
     ssh root@192.168.1.1
     opkg update
     opkg install libubus libubox
     ```

3. **Install the Package**:
   - Install the `myservice` package:
     ```bash
     cd /tmp
     opkg install myservice_1.0-1_x86_64.ipk
     ```

## Step 7: Run and Test the Service

1. **Run the Service**:
   - Start the `myservice` program:
     ```bash
     /usr/bin/myservice &
     ```
   - Expected output in the terminal:
     ```
     UBus service running as 'myservice'.
     ```

2. **Verify Ubus Object**:
   - From another terminal, SSH into the device:
     ```bash
     ssh root@192.168.1.1
     ```
   - List ubus objects:
     ```bash
     ubus list
     ```
   - Expected output (partial):
     ```
     myservice
     ```
   - Note: `myservice` should appear in the list of available ubus objects.

3. **Test the Ubus Method**:
   - Call the `say_hello` method:
     ```bash
     ubus call myservice say_hello
     ```
   - Expected output:
     ```
     {
         "message": "Hello from C"
     }
     ```

## Step 8: Add a Procd Service (Optional)

To run `myservice` as a managed daemon, add an init script.

1. **Create Init Script**:
   - Create `package/myservice/files/myservice.init`:
     ```bash
     #!/bin/sh /etc/rc.common
     USE_PROCD=1
     START=50
     STOP=50

     start_service() {
         procd_open_instance
         procd_set_param command /usr/bin/myservice
         procd_set_param respawn
         procd_close_instance
     }
     ```

2. **Update Package Makefile**:
   - Modify `package/myservice/Makefile` to install the init script:
     ```make
     include $(TOPDIR)/rules.mk

     PKG_NAME:=myservice
     PKG_VERSION:=1.0
     PKG_RELEASE:=1
     PKG_LICENSE:=GPL-2.0

     PKG_BUILD_DIR:=$(BUILD_DIR)/$(PKG_NAME)

     include $(INCLUDE_DIR)/package.mk

     define Package/myservice
         SECTION:=utils
         CATEGORY:=Utilities
         TITLE:=My Ubus Service
         DEPENDS:=+libubus +libubox
     endef

     define Package/myservice/description
         A simple ubus-based service exposing a 'say_hello' method.
     endef

     define Build/Prepare
         mkdir -p $(PKG_BUILD_DIR)
         $(CP) ./src/* $(PKG_BUILD_DIR)/
     endef

     define Build/Compile
         $(MAKE) -C $(PKG_BUILD_DIR) \
             CC="$(TARGET_CC)" \
             CFLAGS="$(TARGET_CFLAGS)" \
             LDFLAGS="$(TARGET_LDFLAGS)"
     endef

     define Package/myservice/install
         $(INSTALL_DIR) $(1)/usr/bin
         $(INSTALL_BIN) $(PKG_BUILD_DIR)/myservice $(1)/usr/bin/
         $(INSTALL_DIR) $(1)/etc/init.d
         $(INSTALL_BIN) ./files/myservice.init $(1)/etc/init.d/myservice
     endef

     $(eval $(call BuildPackage,myservice))
     ```

3. **Rebuild and Reinstall**:
   - Rebuild the package:
     ```bash
     make package/myservice/clean
     make package/myservice/compile
     ```
   - Transfer and install the updated `.ipk` file:
     ```bash
     scp bin/packages/x86_64/base/myservice_1.0-1_x86_64.ipk root@192.168.1.1:/tmp/
     ssh root@192.168.1.1
     cd /tmp
     opkg install --force-reinstall myservice_1.0-1_x86_64.ipk
     ```

4. **Enable and Start the Service**:
   - Enable the service to start on boot:
     ```bash
     /etc/init.d/myservice enable
     ```
   - Start the service:
     ```bash
     /etc/init.d/myservice start
     ```
   - Verify it’s running:
     ```bash
     ps | grep myservice
     ```

5. **Test Again**:
   - Confirm `myservice` is listed:
     ```bash
     ubus list
     ```
   - Call the method:
     ```bash
     ubus call myservice say_hello
     ```

## Troubleshooting

- **Ubus Object Not Listed**:
  - Ensure `myservice` is running (`ps | grep myservice`).
  - Check logs for errors: `logread | grep myservice`.
  - Verify `ubusd` is running: `ps | grep ubusd`.
- **Compilation Errors**:
  - Confirm `libubus` and `libubox` are selected in `make menuconfig` (`Libraries > libubus`, `libubox`).
  - Check for missing headers or libraries in the `Makefile`.
- **Package Installation Fails**:
  - Ensure the `.ipk` matches the device’s architecture (`opkg install` will show errors if mismatched).
  - Install dependencies: `opkg install libubus libubox`.
- **Service Fails to Start**:
  - Check init script permissions: `chmod +x /etc/init.d/myservice`.
  - View logs: `logread -f`.

## Notes

- **Architecture**: Ensure the OpenWRT source or SDK matches the target device’s architecture (e.g., x86_64, mips_24kc).
- **Dependencies**: The package requires `libubus` and `libubox`, which must be present on the device.
- **Procd Integration**: Adding an init script makes the service persistent and manageable via `/etc/init.d/`.
- **Debugging**: Use `logread`, `strace`, or `gdb` for debugging (see Module 8.7).
- **Testing**: The `say_hello` method can be extended to include more functionality, such as reading UCI configurations (see Module 8.3).

