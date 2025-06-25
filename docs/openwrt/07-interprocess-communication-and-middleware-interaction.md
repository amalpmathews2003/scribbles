
# Interprocess Communication and Middleware Interaction

This guide explores OpenWRT’s interprocess communication (IPC) mechanisms, focusing on the **ubus** message bus, its integration with UCI, event handling, and JSON-RPC via **rpcd**.

## UBUS: Message Bus Internals and Protocol

**Ubus** (Micro Bus) is OpenWRT’s lightweight IPC framework for communication between daemons, services, and applications.

### Role:
- Facilitates modular interaction in OpenWRT’s distributed architecture.
- Supports method calls, event notifications, and publish/subscribe patterns.
- Optimized for embedded systems with minimal resource usage.

### Internals:
- Implemented in C as a library (`libubus`) and daemon (`ubusd`).
- Uses Unix domain sockets (`/var/run/ubus.sock`) for local communication.
- Message format: Compact binary protocol with headers and payloads.
- Supports asynchronous and synchronous calls.
- Maintains a registry of objects, methods, and subscribers in `ubusd`.

### Protocol:
- Messages include:
  - **Type**: Request, response, event, or subscription.
  - **Peer ID**: Unique identifier for sender/receiver.
  - **Object/Method**: Target service and function.
  - **Payload**: JSON-encoded data (optional).
- Flow:
  1. Client connects to `ubusd` via socket.
  2. Client sends a request (e.g., method call) or subscribes to events.
  3. `ubusd` routes the message to the target object or broadcasts events.
  4. Response or acknowledgment is returned (if applicable).

### Key Features:
- Lightweight compared to D-Bus or MQTT.
- Extensible for custom services via API.
- CLI tool (`ubus`) for debugging:
  ```bash
  ubus list  # List objects
  ubus call network.interface status  # Call method
  ```

## UBUS Objects and Methods

Ubus organizes communication around **objects** and **methods**.

### Objects:
- Represent services or components (e.g., `network.interface`, `system`).
- Registered with `ubusd` by daemons like `netifd` or `procd`.
- Identified by a unique path (e.g., `service.network`).

### Methods:
- Functions exposed by objects (e.g., `status`, `reload`).
- Accept JSON-formatted arguments and return JSON responses.
- Example: `network.interface.status` returns interface details:
  ```bash
  ubus call network.interface status '{"interface":"lan"}'
  ```

### Structure:
- Objects are hierarchical, similar to a filesystem.
- Methods are defined with signatures (input/output parameters).
- Example object/method:
  - Object: `system`
  - Methods: `info` (returns system info), `reboot` (triggers reboot).

### Accessing:
- Via `ubus` CLI:
  ```bash
  ubus list system  # Show system object methods
  ubus call system.info  # Get system information
  ```
- Programmatically via `libubus` (see Client and Server Implementations).

## Client and Server Implementations in C

Ubus provides a C API (`libubus`) for creating clients and servers.

### Client Implementation:
- Connects to `ubusd` and calls methods or subscribes to events.
- Example: Simple client to call `system.info` and listen for `interface.up` events:
  ```c
  #include <libubus.h>
  #include <libubox/blobmsg_json.h>
  #include <stdio.h>

  static void handle_result(struct ubus_request *req, int type, struct blob_attr *msg) {
      char *json = blobmsg_format_json(msg, true);
      printf("Result: %s\n", json);
      free(json);
  }

  static void handle_event(struct ubus_context *ctx, struct ubus_event_handler *ev,
                          const char *type, struct blob_attr *msg) {
      char *json = blobmsg_format_json(msg, true);
      printf("Event %s: %s\n", type, json);
      free(json);
  }

  int main() {
      struct ubus_context *ctx;
      uint32_t id;
      struct ubus_event_handler ev = { .cb = handle_event };

      ctx = ubus_connect(NULL);  // Connect to /var/run/ubus.sock
      if (!ctx) {
          fprintf(stderr, "Failed to connect to ubus\n");
          return -1;
      }

      // Subscribe to interface.up events
      if (ubus_register_event_handler(ctx, &ev, "interface.up")) {
          fprintf(stderr, "Failed to register event handler\n");
          ubus_free(ctx);
          return -1;
      }

      // Call system.info method
      if (ubus_lookup_id(ctx, "system", &id)) {
          fprintf(stderr, "Failed to lookup system object\n");
          ubus_free(ctx);
          return -1;
      }

      struct blob_buf b = {};
      blob_buf_init(&b, 0);
      ubus_invoke(ctx, id, "info", b.head, handle_result, NULL, 1000);  // Call system.info

      uloop_init();
      ubus_add_uloop(ctx);
      uloop_run();

      blob_buf_free(&b);
      ubus_free(ctx);
      uloop_done();
      return 0;
  }
  ```
- **Key Components**:
  - `ubus_connect`: Establishes connection to `ubusd`.
  - `ubus_lookup_id`: Resolves object path to ID.
  - `ubus_invoke`: Calls a method with a callback for results.
  - `ubus_register_event_handler`: Subscribes to events.
  - `uloop`: Event loop for asynchronous handling.

### Server Implementation:
- Registers an object and exposes methods.
- Example: Server exposing a `custom.hello` method:
  ```c
  #include <libubus.h>
  #include <libubox/blobmsg_json.h>
  #include <stdio.h>

  static int hello_method(struct ubus_context *ctx, struct ubus_object *obj,
                          struct ubus_request_data *req, const char *method,
                          struct blob_attr *msg) {
      struct blob_buf b = {};
      blob_buf_init(&b, 0);
      blobmsg_add_string(&b, "message", "Hello from ubus!");
      ubus_send_reply(ctx, req, b.head);
      blob_buf_free(&b);
      return 0;
  }

  static const struct ubus_method methods[] = {
      UBUS_METHOD("hello", hello_method, NULL),
  };

  static struct ubus_object_type obj_type =
      UBUS_OBJECT_TYPE("custom", methods);

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
- **Key Components**:
  - `ubus_add_object`: Registers the object with `ubusd`.
  - `UBUS_METHOD`: Defines methods and their handlers.
  - `ubus_send_reply`: Sends response to clients.

### Compilation:
- Link against `libubus` and `libubox`:
  ```bash
  gcc -o client client.c -lubus -lubox
  gcc -o server server.c -lubus -lubox
  ```

## UCI + UBUS Integration Flow

UCI and ubus work together to manage configurations and runtime state.

### Integration Flow:
1. **Configuration Storage**:
   - UCI stores settings in `/etc/config/` (e.g., `/etc/config/network`).
   - Example: `network.lan.ipaddr='192.168.1.1'`.
2. **Daemon Initialization**:
   - Daemons like `netifd` read UCI files during startup.
   - `netifd` applies configurations (e.g., sets IP addresses).
3. **Ubus Registration**:
   - `netifd` registers objects (e.g., `network.interface`) with ubus.
   - Exposes methods like `status` or `reload`.
4. **Runtime Interaction**:
   - Clients (e.g., LuCI, CLI) use ubus to query state (e.g., `ubus call network.interface status`).
   - Changes are written to UCI via `uci set` and applied via ubus (e.g., `ubus call network reload`).
5. **Event Propagation**:
   - `netifd` publishes events (e.g., interface up) via ubus.
   - Subscribers (e.g., firewall) react to events.

### Example:
- Change LAN IP:
  ```bash
  uci set network.lan.ipaddr='192.168.1.100'
  uci commit network
  ubus call network reload
  ```
- Query status:
  ```bash
  ubus call network.interface status '{"interface":"lan"}'
  ```

### Benefits:
- Decouples configuration (UCI) from runtime state (ubus).
- Enables dynamic updates without restarting daemons.

## Event Notification and Subscription Mechanism

Ubus supports event-driven communication via notifications and subscriptions.

### Event Notification:
- Daemons publish events to `ubusd` (e.g., `interface.up`).
- Events are broadcast to all subscribers.
- Example: `netifd` sends an event when a WAN interface connects:
  ```bash
  ubus send interface.up '{"interface":"wan"}'
  ```

### Subscription Mechanism:
- Clients subscribe to events using `ubus subscribe`.
- `ubusd` maintains a list of subscribers and forwards events.
- Example CLI subscription:
  ```bash
  ubus listen interface.up
  ```
- Programmatic subscription (C):
  ```c
  static void handle_event(struct ubus_context *ctx, struct ubus_event_handler *ev,
                           const char *type, struct blob_attr *msg) {
      char *json = blobmsg_format_json(msg, true);
      printf("Event %s: %s\n", type, json);
      free(json);
  }

  int main() {
      struct ubus_context *ctx;
      struct ubus_event_handler ev = { .cb = handle_event };

      ctx = ubus_connect(NULL);
      if (!ctx) return -1;

      ubus_register_event_handler(ctx, &ev, "interface.up");

      uloop_init();
      ubus_add_uloop(ctx);
      uloop_run();

      ubus_free(ctx);
      uloop_done();
      return 0;
  }
  ```

### Use Cases:
- Firewall updates rules when an interface changes.
- LuCI refreshes UI on network events.
- Custom scripts react to system events (e.g., USB insertion).

## RPCD and Remote Procedure Calls (via JSON-RPC)

**Rpcd** exposes ubus functionality over HTTP/HTTPS using JSON-RPC.

### Role:
- Enables remote management of OpenWRT via a standardized API.
- Used by LuCI and external applications.

### Internals:
- Runs as a plugin for `uhttpd` (OpenWRT’s web server).
- Translates JSON-RPC requests to ubus calls and returns JSON responses.
- Supports authentication via session tokens (managed by LuCI).

### JSON-RPC Protocol:
- Requests include:
  - `jsonrpc`: Version (e.g., "2.0").
  - `method`: `call` (for ubus methods).
  - `params`: Array of `[session_id, object, method, arguments]`.
  - `id`: Request identifier.
- Example request (get system info):
  ```json
  {
      "jsonrpc": "2.0",
      "method": "call",
      "params": ["00000000000000000000000000000000", "system", "info", {}],
      "id": 1
  }
  ```
- Response:
  ```json
  {
      "jsonrpc": "2.0",
      "id": 1,
      "result": [0, {"uptime": 3600, "memory": {...}}]
  }
  ```

### Accessing:
- Via HTTP/HTTPS at `/ubus` (e.g., `http://192.168.1.1/ubus`).
- Requires authentication for sensitive methods (LuCI manages sessions).
- Example with `curl`:
  ```bash
  curl -d '{"jsonrpc":"2.0","method":"call","params":["00000000000000000000000000000000","system","info",{}],"id":1}' \
    http://192.168.1.1/ubus
  ```

### Security:
- Uses ACLs to restrict access to methods.
- HTTPS recommended for remote access.
- Session tokens prevent unauthorized calls.

### Interaction:
- LuCI sends JSON-RPC requests to update UI or apply changes.
- External tools (e.g., mobile apps) can integrate via the API.


