# Creating a Custom LuCI Page for OpenWRT

This guide details the creation, building, and installation of a custom LuCI package named `luci-hello` for OpenWRT. The package adds a web page with a button that calls the `say_hello` method of the existing `myservice` ubus service, displaying the response (`"Hello from C"`) as a notification. The page is accessible under the "Admin > Services > My Button" menu in the LuCI web interface. An ACL file is included to grant permissions for the `say_hello` method via the HTTP+JSON API provided by `rpcd` and `uhttpd`.

## Prerequisites

- **OpenWRT Build Environment or SDK**: Set up as per [Module 3](#) or [Module 9](#), with the LuCI feed included (`feeds.conf` should have `src-git luci https://github.com/openwrt/luci.git`).
- **myservice Package**: The `myservice` package is installed and running on the OpenWRT device, exposing the `say_hello` method (verifiable via `ubus call myservice say_hello`).
- **Development Host**: A Linux system (e.g., Ubuntu) with `make`, `gcc`, and build tools.
- **Router Configuration**:
  - `uhttpd` is running and accessible (default port 80 or configured, e.g., 8080).
  - The root password is set for authentication.
- **LuCI Installed**: The LuCI web interface is installed on the device .

## Step 1: Create the Package Directory Structure

1. **Set Up Directory**:

   - Inside the OpenWRT source or SDK directory, create the package directory:
     ```bash
     mkdir -p my-packages/luci-hello/htdocs/luci-static/resources/view/myservice
     mkdir -p my-packages/luci-hello/luasrc/controller
     mkdir -p my-packages/luci-hello/root/usr/share/rpcd/acl.d
     ```
2. **Directory Structure**:

   ```
   my-packages/luci-hello/
   ├── Makefile
   ├── htdocs/
   │   ├── luci-static/
   │       ├── resources/
   │           ├── view/
   │               ├── myservice/
   │                   ├── myservice.js
   ├── luasrc/
   │   ├── controller/
   │       ├── myservice.lua
   ├── root/
   │   ├── usr/
   │       ├── share/
   │           ├── rpcd/
   │               ├── acl.d/
   │                   ├── luci-hello.json
   ```

## Step 2: Create the LuCI Page JavaScript

Create `my-packages/luci-hello/htdocs/luci-static/resources/view/myservice/myservice.js` with the following content:

```javascript
'use strict';
'require view';
'require rpc';
'require ui';

var callHello = rpc.declare({
    object: 'myservice',
    method: 'say_hello',
    expect: { message: '' }
});

return view.extend({
    render: function() {
        var btn = E('button', {
            'class': 'btn cbi-button cbi-button-action',
            'click': function() {
                callHello().then(function(message) {
                    ui.addNotification(null, E('div', message));
                }).catch(function(err) {
                    ui.addNotification(null, E('div', 'UBUS call failed'));
                });
            }
        }, _('Say Hello'));

        return E('div', [ btn ]);
    }
});
```

- **Explanation**:
  - Defines a `callHello` function using `rpc.declare` to call the `myservice.say_hello` ubus method, expecting a `message` field in the response.
  - Creates a button with the label "Say Hello" that, when clicked, invokes `callHello` and displays the response (`"Hello from C"`) as a notification or an error message if the call fails.
  - Uses LuCI’s `view`, `rpc`, and `ui` modules for rendering and ubus communication.

## Step 3: Create the LuCI Controller

Create `my-packages/luci-hello/luasrc/controller/myservice.lua` with the following content:

```lua
module("luci.controller.myservice", package.seeall)

function index()
    entry({"admin", "services", "myservice"}, view("myservice/myservice"), _("My Button"), 10)
end
```

- **Explanation**:
  - Registers a menu entry in LuCI under "Admin > Services > My Button".
  - Maps the path `/admin/services/myservice` to the `myservice/myservice.js` view.
  - The priority (`10`) determines the menu order.

## Step 4: Create the ACL File

Create `my-packages/luci-hello/root/usr/share/rpcd/acl.d/luci-hello.json` with the following content:

```json
{
    "luci-hello": {
        "description": "Hello World LuCI App",
        "read": {
            "ubus": {
                "myservice": ["say_hello"]
            }
        }
    }
}
```

- **Explanation**:
  - Grants read access to the `myservice.say_hello` method for the `luci-hello` role, allowing the LuCI page to call it via the HTTP+JSON API.
  - Note: The existing `myservice` ACL (`/usr/share/rpcd/acl.d/myservice.json`) already grants access to `say_hello` and `get_hostname`, but this ACL ensures the `luci-hello` package has its own permissions.

## Step 5: Create the Package Makefile

Create `my-packages/luci-hello/Makefile` with the following content:

```make
include $(TOPDIR)/rules.mk

LUCI_NAME:=luci-hello
LUCI_DEPENDS:=+luci-base +myservice
LUCI_TITLE:=Hello World LuCI App
LUCI_PKGARCH:=all

include $(TOPDIR)/feeds/luci/luci.mk
```

- **Explanation**:
  - Defines the package as `luci-hello` with dependencies on `luci-base` (for LuCI framework) and `myservice` (the ubus service).
  - Uses the `luci.mk` include file for LuCI package conventions.
  - Sets `LUCI_PKGARCH:=all` since the package contains only architecture-independent files (JavaScript, Lua, JSON).

## Step 6: Build and Install the Package

1. **Ensure LuCI Feed**:

   - Verify that the LuCI feed is included in `feeds.conf`:
     ```bash
     echo "src-git luci https://github.com/openwrt/luci.git" >> feeds.conf
     ```
   - Update and install feeds:
     ```bash
     ./scripts/feeds update luci mypackages
     ./scripts/feeds install -a -p luci
     ./scripts/feeds install -a -p mypackages
     ```
2. **Configure the Package**:

   - Run `make menuconfig`:
     ```bash
     make menuconfig
     ```
   - Navigate to `LuCI > Applications > luci-hello` and mark it with `*`.
   - Ensure `myservice`, `luci-base`, `libubus`, `libubox`, `libblobmsg-json`, `rpcd`, `uhttpd`, and `uhttpd-mod-ubus` are selected.
   - Save and exit.
3. **Build the Package**:

   - Compile the package:
     ```bash
     make package/luci-hello/compile
     ```
4. **Restart Services**:

   - Restart `rpcd` and `uhttpd` to apply the ACL:
     ```bash
     /etc/init.d/rpcd restart
     /etc/init.d/uhttpd restart
     ```
5. **Verify myservice**:

   - Ensure `myservice` is running:

     ```bash
     ps | grep myservice
     /etc/init.d/myservice restart
     ```
   - Verify the ubus object:

     ```bash
     ubus list
     ```

     Expected output (partial):
     ```
     myservice
     ```
   - Test `say_hello`:

     ```bash
     ubus call myservice say_hello

     ```

## Step 7: Test the LuCI Page

1. **Access the LuCI Interface**:

   - Open a web browser and navigate to the device’s LuCI interface (e.g., `http://192.168.1.1` or `http://192.168.1.1:8080` if configured).
   - Log in with the root credentials (e.g., username: `root`, password: password).
2. **Navigate to the Page**:

   - Go to `Admin > Services > My Button` in the LuCI menu.
   - Expected: A page with a single "Say Hello" button.
3. **Test the Button**:

   - Click the "Say Hello" button.
   - Expected: A notification appears with the text `"Hello from C"`.
   - If the call fails, a notification with `"UBUS call failed"` appears.

## Troubleshooting

- **LuCI Page Not Visible**:
  - Verify `luci-hello` is installed: `opkg list-installed | grep luci-hello`.
  - Check LuCI cache: Clear it with `rm -rf /tmp/luci-modulecache/*` and reload LuCI.
  - Ensure `myservice.lua` is in `/usr/lib/lua/luci/controller/`.
- **Button Click Fails**:
  - Check if `myservice` is running: `ps | grep myservice`.
  - Verify ACL permissions: `cat /usr/share/rpcd/acl.d/luci-hello.json` and `cat /usr/share/rpcd/acl.d/myservice.json`.
  - Restart services: `/etc/init.d/rpcd restart; /etc/init.d/uhttpd restart`.
  - Check logs: `logread | grep myservice` or `logread | grep uhttpd`.
- **HTTP API Fails**:
  - Ensure `uhttpd` and `uhttpd-mod-ubus` are installed and running.
  - Verify port configuration: `uci show uhttpd.main.listen_http`.
  - Test authentication: Ensure `root` and password work via SSH.
- **Permission Denied Errors**:
  - Confirm both ACL files (`myservice.json` and `luci-hello.json`) grant read access to `say_hello`.
  - Restart `rpcd` and `uhttpd` after ACL changes.

## Notes

- **ACL Importance**: The `luci-hello.json` ACL ensures the LuCI page can call `myservice.say_hello` via the HTTP API. The `myservice.json` ACL from the `myservice` package also grants access.
- **Security**: Use a strong root password and consider HTTPS for `uhttpd` in production.
- **Extending Functionality**: Add more buttons to call other methods (e.g., `get_hostname`) or integrate with forms for dynamic inputs (see LuCI documentation).
- **Debugging**: Use browser developer tools (F12) to inspect JavaScript errors and `logread` for server-side issues.
- **LuCI Dependencies**: Ensure `luci-base` is installed and compatible with the OpenWRT version.
