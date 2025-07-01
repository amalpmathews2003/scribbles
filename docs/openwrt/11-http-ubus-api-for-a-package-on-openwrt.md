# HTTP Ubus API for a package on OpenWRT

This guide demonstrates how to interact with an existing `myservice` ubus package running on an OpenWRT device using the HTTP+JSON API provided by `rpcd` and `uhttpd` with the `uhttpd-mod-ubus` module. The `myservice` package exposes two methods: `say_hello` (returns `{"message": "Hello from C"}`) and `get_hostname` (returns `{"hostname": "<device_hostname>"}`). The guide covers setting up the HTTP API, authenticating via `session.login`, and calling the `myservice.get_hostname` method using a bash script with `curl`.

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
   
     define Package/myservice/install
         $(INSTALL_DIR) $(1)/usr/bin
         $(INSTALL_BIN) $(PKG_BUILD_DIR)/myservice $(1)/usr/bin/
         $(INSTALL_DATA) ./files/myservice.json $(1)/usr/share/rpcd/acl.d/myservice.json
     endef

     $(eval $(call BuildPackage,myservice))
     ```

   - **Changes**:
     - Added `$(INSTALL_DATA)` to install `myservice.json` to `/usr/share/rpcd/acl.d/`.

3. **Deploy the package**:
    ```bash
     scp bin/packages/x86_64/base/myservice_1.0-1_x86_64.apk root@192.168.1.1:/tmp/
     ssh root@192.168.1.1
     cd /tmp
     apk install myservice_1.0-1_x86_64.apk
4. **Restart Services**:
   - Restart `rpcd` and `uhttpd` to apply the ACL:
     ```bash
     /etc/init.d/rpcd restart
     /etc/init.d/uhttpd restart
     ```
## Step 2: Create and Run the Bash Script for HTTP Ubus API

1. **Create the Bash Script**:
   - On the development host, create `test_ubus.sh` to authenticate and call `myservice.get_hostname`:
     ```bash
     #!/bin/bash

     ROUTER_IP='127.0.0.1:8080'
     USERNAME='root'
     PASSWORD='password'

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
     Logging in to 127.0.0.1:8080 ...
    
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


## Troubleshooting

- **HTTP API Fails**:
  - Verify `uhttpd` is running: `ps | grep uhttpd`.
  - Check if `uhttpd-mod-ubus` is installed: `opkg list-installed | grep uhttpd-mod-ubus`.
  - Ensure the correct port is used: `uci show uhttpd.main.listen_http`.
  - Confirm the password is correct: Test SSH login with `root` and `amalaleena`.
  - Check firewall rules: Ensure port 8080 (or 80) is open (`uci show firewall`).
- **Login Failed**:
  - Verify `rpcd` is running: `ps | grep rpcd`.
  - Ensure `uhttpd-mod-ubus` is enabled in `uhttpd` configuration.
  - Check logs: `logread | grep uhttpd` or `logread | grep rpcd`.
- **myservice Not Responding**:
  - Ensure `myservice` is running: `ps | grep myservice`.
  - Restart the service: `/etc/init.d/myservice restart`.
  - Check logs: `logread | grep myservice`.
  - Verify `ubusd` is running: `ps | grep ubusd`.
- **Dependencies Missing**:
  - Install required packages: `opkg install libubus libubox libblobmsg-json rpcd uhttpd uhttpd-mod-ubus`.

## Notes

- **HTTP API**: The `rpcd` and `uhttpd` setup with `uhttpd-mod-ubus` enables remote access to ubus methods via HTTP, suitable for web-based management or automation scripts.
- **Security**: Use a strong password for the root user and consider enabling HTTPS in `uhttpd` for production environments (`uci set uhttpd.main.listen_https="0.0.0.0:443"`).
- **Port Configuration**: Adjust `ROUTER_IP` in the script to match the device’s IP and port (e.g., `192.168.1.1:80` for default).
- **Debugging**: Use `logread`, `strace`, or `gdb` for deeper debugging (see Module 8.7).
- **Extending Functionality**: Add more ubus methods to `myservice` or integrate with UCI for configuration (see Module 8.3).

