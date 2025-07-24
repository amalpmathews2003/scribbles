# Advancing the my-amx Package: Event Handling and Persistent Objects

In our ongoing development of the `my-amx` package for the Ambroix PRPL framework, we’ve introduced new features to enhance method registration and event handling. This blog post dives into the latest updates, focusing on the `my-amx-definition.odl`, `my-amx.odl`, and `my-amx.c` files. These updates enable the package to define persistent objects and handle events, making it more robust for embedded systems applications, such as IoT or network configuration.

## Project Structure Recap

The `my-amx` package, designed for an OpenWRT-like build environment, has the following structure:

```
.
├── Makefile
└── src
    ├── CMakeLists.txt
    ├── my-amx.c
    └── odl
        ├── my-amx-definition.odl
        └── my-amx.odl
```

This blog focuses on three updated components:
- **my-amx-definition.odl**: Defines a persistent `Person` object and sets up event handling.
- **my-amx.odl**: Configures the Ambroix runtime and extends the `Person` object definition.
- **my-amx.c**: Implements an event handler to process runtime events.

## New Features and Components

### 1. **my-amx-definition.odl: Defining Persistent Objects**

The `my-amx-definition.odl` file introduces a persistent `Person` object and sets up event monitoring for changes to this object.

```odl
%define {
    %persistent object Person {
        string name {
        }
    }
}

%populate {
    on event "dm:object-changed" call on_event
            filter 'object == "Person."';
}
```

Key points:
- **Persistent Object**: The `Person` object is marked as `%persistent`, meaning its state is retained across system restarts, ideal for embedded systems requiring consistent data.
- **Attributes**: The `Person` object includes a `name` attribute of type `string`.
- **Event Handling**: The `%populate` section triggers the `on_event` function whenever the `dm:object-changed` event occurs for the `Person` object, filtered by the object path `Person.`.

This file lays the foundation for managing data models in the Ambroix runtime, with the `Person` object serving as a simple example that can be extended for more complex use cases.

### 2. **my-amx.odl: Extending Object Definitions**

The updated `my-amx.odl` file configures the Ambroix runtime and refines the `Person` object definition.

```odl
#!/usr/bin/amxrt

%config {
    name = my-amx;
    so-dir = "/usr/lib/amx/${name}";
    so-name = "${name}";
}

import "${so-dir}/${so-name}.so" as "${name}";
include "${name}-definition.odl";

%define {
    entry-point my-amx.my_amx_main;
}

```

Key updates:
- **Include Directive**: The `include "${name}-definition.odl"` statement imports the object definitions from `my-amx-definition.odl`, ensuring modularity.

### 3. **my-amx.c: Implementing Event Handling**

The `my-amx.c` file now includes an event handler function, `_on_event`, to process events triggered by the Ambroix runtime.

```c
void _on_event(const char *const event_name,
               const amxc_var_t *const event_data,
               void *const priv)
{
    printf("AMXO_EVENT: Event '%s' triggered with data: \n", event_name);
}
```

Key points:
- **Event Handler**: The `_on_event` function is called when events like `dm:object-changed` occur, as defined in `my-amx-definition.odl`. It prints the event name, with `event_data` and `priv` available for processing event details or private data.
- **Future Expansion**: The current implementation is minimal, printing a message to confirm event triggering. Future iterations could parse `event_data` (an `amxc_var_t` variant type) to extract and process specific event information.

## Integration with the Ambroix Framework

The updates enable `my-amx` to:
- **Define Data Models**: The `Person` object provides a structured way to manage data, with persistence ensuring durability in embedded environments.
- **Handle Events**: The `dm:object-changed` event monitoring allows the package to react to changes in the `Person` object, useful for real-time applications.
- **Modular Configuration**: The separation of `my-amx.odl` and `my-amx-definition.odl` promotes maintainability and scalability.

## Building and Testing

To build and test the updated package:
1. **Update the Build Environment**: Ensure the `Makefile` and `CMakeLists.txt` (from the previous blog) are configured for the target platform (e.g., `x86_64_musl`).
2. **Compile**: Run `make` to build the `my-amx` executable and `libmy-amxl.so` shared library.
3. **Install**: Install artifacts to `/usr/bin`, `/usr/lib/amx/my-amx`, `/usr/include/my-amx`, and `/etc/amx/my-amx/odl`.
4. **Test Events**: Use the Ambroix runtime (`amxrt`) to load `my-amx.odl` and simulate changes to the `Person` object to trigger the `on_event` handler.
