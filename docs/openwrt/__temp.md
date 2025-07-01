This guide extends the previous instructions for testing the HTTP+JSON API for the `myservice` ubus package running on an OpenWRT device. It includes adding an Access Control List (ACL) file to define permissions for the `myservice` ubus object, ensuring that the `say_hello` and `get_hostname` methods can be accessed via the HTTP API provided by `rpcd` and `uhttpd` with the `uhttpd-mod-ubus` module. The ACL file is installed as part of the package, and `rpcd` and `uhttpd` are restarted to apply the changes. The guide then tests the HTTP API using a bash script with `curl`.


# Testing HTTP Ubus API for myservice Package with ACL on OpenWRT

This guide demonstrates how to enhance the existing `myservice` ubus package on an OpenWRT device by adding an Access Control List (ACL) file to define permissions for the `myservice` ubus object, allowing HTTP access to the `say_hello` and `get_hostname` methods. The package uses the HTTP+JSON API provided by `rpcd` and `uhttpd` with the `uhttpd-mod-ubus` module, accessible at `http://<router_ip>/ubus`. The guide covers installing the ACL file, restarting services, authenticating via `session.login`, and calling the `myservice.get_hostname` method using a bash script with `curl`.

## Prerequisites

- **OpenWRT Device**: Running the `myservice` package with `say_hello` and `get_hostname` methods, accessible via SSH.
- **Dependencies**: Ensure `libubus`, `libubox`, `libblobmsg-json`, `rpcd`, `uhttpd`, and `uhttpd-mod-ubus` are installed on the device.
- **Development Host**: A Linux system (e.g., Ubuntu) with `curl` and `jq` installed.
- **Router Configuration**: 
  - `uhttpd` is running and accessible (default port 80 or configured, e.g., 8080).
  - The root password is set for authentication.
- **myservice Status**: The `myservice` package is installed and running, with the ubus object `myservice` registered (verifiable via `ubus list`).

## Step 1: Update the Package with ACL File

1. **Create ACL File**:
   - Create `my-packages/myservice/files/myservice.json` with the following content to define read access for the `myservice` ubus methods:
     ```json
     {
         "myservice": {
             "description": "Access role for my_service",
             "read": {
                 "ubus": {
                     "myservice": ["say_hello", "get_hostname"]
                 }
             },
             "write": {}
         }
     }
     ```
   - **Explanation**:
     - Grants read access to the `say_hello` and `get_hostname` methods of the `myservice` ubus object.
     - No write permissions are defined, as the methods are read-only.

2. **Update Package Makefile**:
   - Modify `my-packages/myservice/Makefile` to install the ACL file to `/usr/share/rpcd/acl.d/`:
     ```make
     include $(TOPDIR)/rules.mk
     include $(INCLUDE_DIR)/package.mk

     PKG_NAME:=myservice
     PKG_VERSION:=1.0
     PKG_RELEASE:=1
     PKG_LICENSE:=GPL-2.0

     PKG_BUILD_DIR:=$(BUILD_DIR)/$(PKG_NAME)
     SOURCE_DIR:=$(TOPDIR)/my-packages/$(PKG_NAME)/src

     define Package/myservice
         SECTION:=utils
         CATEGORY:=Utilities
         TITLE:=My Ubus Service
         DEPENDS:=+libubus +libubox +libblobmsg-json
     endef

     define Package/myservice/description
         A ubus-based service exposing 'say_hello' and 'get_hostname' methods.
     endef

     define Build/Prepare
         mkdir -p $(PKG_BUILD_DIR)
         $(CP) $(SOURCE_DIR)/* $(PKG_BUILD_DIR)/
         $(Build/Patch)
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
         $(INSTALL_DIR) $(1)/usr/share/rpcd/acl.d
         $(INSTALL_DATA) ./files/myservice.json $(1)/usr/share/rpcd/acl.d/myservice.json
     endef

     $(eval $(call BuildPackage,myservice))
     ```

   - **Changes**:
     - Added `$(INSTALL_DIR) $(1)/usr/share/rpcd/acl.d` and `$(INSTALL_DATA)` to install `myservice.json` to `/usr/share/rpcd/acl.d/`.

3. **Verify Source Files**:
   - Ensure `my-packages/myservice/src/myservice.c` and `my-packages/myservice/src/Makefile` are as previously defined:
     - `myservice.c`: Contains `say_hello` and `get_hostname` methods (see previous guide).
     - `src/Makefile`:
       ```make
       all: myservice

       myservice: myservice.c
           $(CC) $(CFLAGS) -o myservice myservice.c -lubus -lubox -lblobmsg_json

       clean:
           rm -f myservice *.o
       ```
   - Ensure `my-packages/myservice/files/myservice.init` exists:
     ```bash
     #!/bin/sh /etc/rc.common
     USE_PROCD=1
     START=50
     STOP=50

     start_service() {
         procd_open_instance
         procd_set_param command /usr/bin/myservice
         procd_set_param respawn
         procd_set_param stderr 1
         procd_close_instance
     }
     ```

## Step 2: Build and Install the Updated Package

1. **Include the Package Feed**:
   - Edit `feeds.conf` in the OpenWRT source or SDK root directory:
     ```bash
     echo "src-link mypackages $(pwd)/my-packages" >> feeds.conf
     ```
   - Update and install feeds:
     ```bash
     ./scripts/feeds update mypackages
     ./scripts/feeds install -a -p mypackages
     ```

2. **Configure the Package**:
   - Run `make menuconfig`:
     ```bash
     make menuconfig
     ```
   - Navigate to `Utilities > myservice` and mark it with `*`.
   - Ensure `libubus`, `libubox`, `libblobmsg-json`, `rpcd`, `uhttpd`, and `uhttpd-mod-ubus` are selected under `Utilities` and `Libraries`.
   - Save and exit.

3. **Build the Package**:
   - Clean and compile the package:
     ```bash
     make package/myservice/clean
     make package/myservice/compile
     ```
   - Output: A package file named `myservice_1.0-1_<arch>.ipk` in `bin/packages/<arch>/mypackages/`.
     - Example: `bin/packages/x86_64/mypackages/myservice_1.0-1_x86_64.ipk`.

4. **Deploy the Package**:
   - Transfer the `.ipk` file to the device:
     ```bash
     scp bin/packages/x86_64/mypackages/myservice_1.0-1_x86_64.ipk root@192.168.1.1:/tmp/
     ```
   - SSH into the device and install dependencies:
     ```bash
     ssh root@192.168.1.1
     opkg update
     opkg install libubus libubox libblobmsg-json rpcd uhttpd uhttpd-mod-ubus
     ```
   - Install the updated package:
     ```bash
     cd /tmp
     opkg install --force-reinstall myservice_1.0-1_x86_64.ipk
     ```

5. **Restart Services**:
   - Restart `rpcd` and `uhttpd` to apply the ACL:
     ```bash
     /etc/init.d/rpcd restart
     /etc/init.d/uhttpd restart
     ```

6. **Verify ACL Installation**:
   - Check the ACL file on the device:
     ```bash
     cat /usr/share/rpcd/acl.d/myservice.json
     ```
     Expected output:
     ```json
     {
         "myservice": {
             "description": "Access role for my_service",
             "read": {
                 "ubus": {
                     "myservice": ["say_hello", "get_hostname"]
                 }
             },
             "write": {}
         }
     }
     ```

7. **Start the Service**:
   - Enable and start `myservice`:
     ```bash
     /etc/init.d/myservice enable
     /etc/init.d/myservice start
     ```
   - Verify it’s running:
     ```bash
     ps | grep myservice
     ```

## Step 3: Test the HTTP Ubus API

1. **Create the Bash Script**:
   - On the development host, create `test_ubus.sh` to authenticate and call `myservice.get_hostname`:
     ```bash
     #!/bin/bash

     ROUTER_IP='192.168.1.1:8080'
     USERNAME='root'
     PASSWORD='amalaleena'

     UBUS_OBJECT="myservice"
     METHOD="get_hostname"
     PARAMS="{}"

     UBUS_URL="http://$ROUTER_IP/ubus"

     echo "Logging in to $ROUTER_IP ..."

     LOGIN_RESPONSE=$(curl -s -X POST "$UBUS_URL" \
       -H 'Content-Type: application/json' \
       -d "{
         \"jsonrpc\": \"2.0\",
         \"id\": 1,
         \"method\": \"call\",
         \"params\": [
             \"00000000000000000000000000000000\",
             \"session\",
             \"login\",
             {
                 \"username\": \"$USERNAME\",
                 \"password\": \"$PASSWORD\"
             }
         ]
       }")

     echo "$LOGIN_RESPONSE" | jq

     # Parse token
     TOKEN=$(echo "$LOGIN_RESPONSE" | jq -r '.result[1].ubus_rpc_session')

     if [ "$TOKEN" == "null" ] || [ -z "$TOKEN" ]; then
       echo "Login failed!"
       exit 1
     fi

     echo "Login successful. Session token: $TOKEN"

     echo "Calling: object='$UBUS_OBJECT' method='$METHOD' params=$PARAMS"

     CALL_RESPONSE=$(curl -s -X POST "$UBUS_URL" \
       -H 'Content-Type: application/json' \
       -d "{
         \"jsonrpc\": \"2.0\",
         \"id\": 2,
         \"method\": \"call\",
         \"params\": [
             \"$TOKEN\",
             \"$UBUS_OBJECT\",
             \"$METHOD\",
             $PARAMS
         ]
       }")

     echo "Response:"
     echo "$CALL_RESPONSE" | jq
     echo
     ```

2. **Make the Script Executable**:
   - ```bash
     chmod +x test_ubus.sh
     ```

3. **Run the Script**:
   - Execute the script:
     ```bash
     ./test_ubus.sh
     ```
   - Expected output (example, assuming hostname is `OpenWrt`):
     ```
     Logging in to 192.168.1.1:8080 ...
     {
       "jsonrpc": "2.0",
       "id": 1,
       "result": [
         0,
         {
           "ubus_rpc_session": "<some_token>",
           ...
         }
       ]
     }
     Login successful. Session token: <some_token>
     Calling: object='myservice' method='get_hostname' params={}
     Response:
     {
       "jsonrpc": "2.0",
       "id": 2,
       "result": [
         0,
         {
           "hostname": "OpenWrt"
         }
       ]
     }
     ```

4. **Test say_hello Method**:
   - Modify the script to test `say_hello`:
     ```bash
     UBUS_OBJECT="myservice"
     METHOD="say_hello"
     PARAMS="{}"
     ```
   - Run again:
     ```bash
     ./test_ubus.sh
     ```
   - Expected output:
     ```
     {
       "jsonrpc": "2.0",
       "id": 2,
       "result": [
         0,
         {
           "message": "Hello from C"
         }
       ]
     }
     ```

## Step 4: Verify Ubus Service via CLI

1. **Check Ubus Object**:
   - SSH into the device:
     ```bash
     ssh root@192.168.1.1
     ```
   - List ubus objects:
     ```bash
     ubus list
     ```
     Expected output (partial):
     ```
     myservice
     ```

2. **Test Methods via CLI**:
   - Call `get_hostname`:
     ```bash
     ubus call myservice get_hostname
     ```
     Expected output:
     ```
     {
         "hostname": "<device_hostname>"
     }
     ```
   - Call `say_hello`:
     ```bash
     ubus call myservice say_hello
     ```
     Expected output:
     ```
     {
         "message": "Hello from C"
     }
     ```

## Troubleshooting

- **HTTP API Fails**:
  - Verify `uhttpd` is running: `ps | grep uhttpd`.
  - Check if `uhttpd-mod-ubus` is installed: `opkg list-installed | grep uhttpd-mod-ubus`.
  - Ensure the correct port is used: `uci show uhttpd.main.listen_http`.
  - Confirm the password is correct: Test SSH login with `root` and `amalaleena`.
  - Check firewall rules: Ensure port 8080 (or 80) is open (`uci show firewall`).
- **Login Failed**:
  - Verify `rpcd` is running: `ps | grep rpcd`.
  - Ensure the ACL file is present: `cat /usr/share/rpcd/acl.d/myservice.json`.
  - Check logs: `logread | grep uhttpd` or `logread | grep rpcd`.
- **myservice Not Responding**:
  - Ensure `myservice` is running: `ps | grep myservice`.
  - Restart the service: `/etc/init.d/myservice restart`.
  - Check logs: `logread | grep myservice`.
  - Verify `ubusd` is running: `ps | grep ubusd`.
- **Permission Denied Errors**:
  - Ensure the ACL file grants read access to `myservice` methods.
  - Restart `rpcd` and `uhttpd` after installing the ACL file: `/etc/init.d/rpcd restart; /etc/init.d/uhttpd restart`.

## Notes

- **ACL Importance**: The `myservice.json` ACL file is critical for allowing HTTP access to ubus methods. Without it, calls to `myservice` methods may return permission errors.
- **HTTP API**: The `rpcd` and `uhttpd` setup with `uhttpd-mod-ubus` enables remote access to ubus methods, ideal for web-based management or automation.
- **Security**: Use a strong password for the root user and consider enabling HTTPS in `uhttpd` for production (`uci set uhttpd.main.listen_https="0.0.0.0:443"`).
- **Port Configuration**: Adjust `ROUTER_IP` in the script to match the device’s IP and port (e.g., `192.168.1.1:80` for default).
- **Debugging**: Use `logread`, `strace`, or `gdb` for deeper debugging (see Module 8.7).
- **Extending Functionality**: Add more ubus methods or integrate with UCI for configuration (see Module 8.3).

