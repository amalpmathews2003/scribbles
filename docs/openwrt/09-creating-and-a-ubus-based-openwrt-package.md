
# Creating a Ubus-Based OpenWRT Package

This guide outlines how to create, build, and install a custom OpenWRT package for a C program that uses the **ubus** IPC framework to register a service named `myservice`. The program exposes a `say_hello` method, and after installation, `ubus list` on the OpenWRT device should list `myservice`. 

## Prerequisites

- **OpenWRT Build Environment or SDK**: Set up as per [Module 3](#) or [Module 9](#) (e.g., OpenWRT source or SDK for the target architecture, such as x86_64).
- **Target Device**: An OpenWRT device with SSH access and `apk` package manager.
- **Dependencies**: Ensure `libubus` and `libubox` are available in the build system and on the target device.
- **Development Host**: A Linux system (e.g., Ubuntu) with `make`, `gcc`, and other build tools.

## Step 1: Create the Package Directory Structure

1. **Set Up Directory**:
   - Inside the OpenWRT source or SDK directory, create a directory for the custom package:
     ```bash
     mkdir -p my-packages/myservice/src
     ```

2. **Directory Structure**:
   ```
   my-packages/myservice/
   ├── Makefile
   ├── src/
   │   ├── Makefile
   │   ├── myservice.c
   ```

## Step 2: Write the C Program

Create the C program file `my-packages/myservice/src/myservice.c` with the provided ubus code:

```c
#include <libubus.h>
#include <libubox/blobmsg_json.h>

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

Create `my-packages/myservice/src/Makefile` to compile the C program:
for testing in host system
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

Create `my-packages/myservice/Makefile` to integrate with the OpenWRT build system:

```make
include $(TOPDIR)/rules.mk
include $(INCLUDE_DIR)/package.mk	

PKG_NAME:=myservice
PKG_VERSION:=1.0
PKG_RELEASE:=1

PKG_BUILD_DIR:=$(BUILD_DIR)/$(PKG_NAME)
SOURCE_DIR:=$(TOPDIR)/my-packages/${PKG_NAME}/src

define Package/${PKG_NAME}
  SECTION:=utils
  CATEGORY:=Utilities
  TITLE:=My Ubus Application
  DEPENDS:=+libubus +libubox
endef

define Package/${PKG_NAME}/description
  A custom application using ubus.
endef

define Build/Prepare
		mkdir -p $(PKG_BUILD_DIR)
		cp $(SOURCE_DIR)/* $(PKG_BUILD_DIR)
		$(Build/Patch)
endef

define Build/Compile
	$(TARGET_CC) $(TARGET_CFLAGS) -o $(PKG_BUILD_DIR)/${PKG_NAME} $(PKG_BUILD_DIR)/hello-service.c -lubus -lubox
endef

define Package/${PKG_NAME}/install
	$(INSTALL_DIR) $(1)/usr/bin
	$(INSTALL_BIN) $(PKG_BUILD_DIR)/${PKG_NAME} $(1)/usr/bin/
endef

$(eval $(call BuildPackage,${PKG_NAME}))
```

- **Explanation**:
  - Defines package metadata (`PKG_NAME`, `PKG_VERSION`, etc.).
  - Specifies dependencies (`libubus`, `libubox`).
  - Copies source files, compiles using the source `Makefile`, and installs the binary to `/usr/bin`.

## Step 5: Build the Package

1. **Update Feeds** (if using a custom feed):
   - If the package is in a custom feed (e.g., `my-packages/`), update `feeds.conf`:
     ```bash
     echo "src-link mypackages $(pwd)/my-packages" >> feeds.conf
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
   - Output: A package file named `myservice_1.0-1_<arch>.apk` in `bin/packages/<arch>/base/` (or `bin/packages/<arch>/mypackages/` if using a custom feed).
     - Example: `bin/packages/x86_64/base/myservice_1.0-1_x86_64.apk`.

## Step 6: Deploy the Package to OpenWRT

1. **Transfer the Package**:
   - Copy the `.apk` file to the OpenWRT device:
     ```bash
     scp bin/packages/x86_64/base/myservice_1.0-1_x86_64.apk root@192.168.1.1:/tmp/
     ```

2. **Install the Package**:
   - Install the `myservice` package:
     ```bash
     cd /tmp
     apk install myservice_1.0-1_x86_64.apk
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
     define Package/myservice/install
         $(INSTALL_DIR) $(1)/usr/bin
         $(INSTALL_BIN) $(PKG_BUILD_DIR)/myservice $(1)/usr/bin/
         $(INSTALL_DIR) $(1)/etc/init.d
         $(INSTALL_BIN) ./files/myservice.init $(1)/etc/init.d/myservice
     endef
     ```

3. **Rebuild and Reinstall**:
   - Rebuild the package:
     ```bash
     make package/myservice/clean
     make package/myservice/compile
     ```
   - Transfer and install the updated `.apk` file:
     ```bash
     scp bin/packages/x86_64/base/myservice_1.0-1_x86_64.apk root@192.168.1.1:/tmp/
     ssh root@192.168.1.1
     cd /tmp
     apk install myservice_1.0-1_x86_64.apk
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

## Step 9: Argument Handling 
```c
static int hello_with_name_method(
    struct ubus_context *ctx,
    struct ubus_object *obj,
    struct ubus_request_data *req,
    const char *method,
    struct blob_attr *msg)
{
    enum {
        HELLO_NAME,
        __HELLO_MAX
    };

    static const struct blobmsg_policy policy[] = {
        [HELLO_NAME] = { .name = "name", .type = BLOBMSG_TYPE_STRING },
    };

    struct blob_attr *tb[__HELLO_MAX];
    blobmsg_parse(policy, __HELLO_MAX, tb, blob_data(msg), blob_len(msg));

    char *name = "stranger";
    if (tb[HELLO_NAME]) {
        name = blobmsg_get_string(tb[HELLO_NAME]);
    }

    blob_buf_init(&b, 0);
    char buff[128];
    snprintf(buff, sizeof(buff), "hello %s", name);
    blobmsg_add_string(&b, "message", buff);
    ubus_send_reply(ctx, req, b.head);

    return 0;
}

static const struct ubus_method my_methods[] = {
    UBUS_METHOD_NOARG("say_hello", hello_method),
    UBUS_METHOD_NOARG("say_hello_with_name", hello_with_name_method),
};
```

- Call the `say_hello_with_name` method with an argument:
     ```bash
     ubus call myservice say_hello_with_name '{"name": "Amal"}'
     ```
- Expected output:
  ```
  {
      "message": "hello Amal"
  }
  ```

## Step 10: UCI Integration 
```c
static void __get_hostname_cb(
    struct ubus_request *req,
    int type,
    struct blob_attr *msg)
{

  struct ubus_request_data *req_t = req->priv;
  struct blob_attr *tb;
  static const struct blobmsg_policy policy = {
      .name = "value",
      .type = BLOBMSG_TYPE_STRING};

  blobmsg_parse(&policy, 1, &tb, blob_data(msg), blob_len(msg));
  if (tb && blobmsg_type(tb) == BLOBMSG_TYPE_STRING)
  {
    blob_buf_init(&b, 0);
    blobmsg_add_string(&b,"hostname", blobmsg_get_string(tb));
    ubus_send_reply(ctx, req_t, b.head);
  }

}

static int get_hostname_method(
    struct ubus_context *ctx,
    struct ubus_object *obj,
    struct ubus_request_data *req,
    const char *method,
    struct blob_attr *msg)
{

  uint32_t uci_id;
  ubus_lookup_id(ctx, "uci", &uci_id);

  blob_buf_init(&b, 0);
  blobmsg_add_string(&b, "config", "system");
  blobmsg_add_string(&b, "section", "@system[0]");
  blobmsg_add_string(&b, "option", "hostname");
  ubus_invoke(ctx, uci_id, "get", b.head, __get_hostname_cb, req, 5000);

  return 0;
}

```

- Call the `get_hostname` method:
     ```bash
     ubus call myservice get_hostname'
     ```
- Expected output:
  ```
  {
      "hostname": "OpenWrt"
  }
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
  - Ensure the `.apk` matches the device’s architecture (`apk install` will show errors if mismatched).
  - Install dependencies: `apk install libubus libubox`.
- **Service Fails to Start**:
  - Check init script permissions: `chmod +x /etc/init.d/myservice`.
  - View logs: `logread -f`.

## Notes

- **Architecture**: Ensure the OpenWRT source or SDK matches the target device’s architecture (e.g., x86_64, mips_24kc).
- **Dependencies**: The package requires `libubus` and `libubox`, which must be present on the device.
- **Procd Integration**: Adding an init script makes the service persistent and manageable via `/etc/init.d/`.
- **Debugging**: Use `logread`, `strace`, or `gdb` for debugging (see Module 8.7).
- **Testing**: The `say_hello` method can be extended to include more functionality, such as reading UCI configurations (see Module 8.3).

