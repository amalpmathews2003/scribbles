# Running ODL Files with a C Application Instead of amxrt

In this post, you'll learn how to run ODL (Object Definition Language) files directly from a C application, rather than using the `amxrt` command-line tool. We'll use the example from `my-amx/src/main.c` to show how to initialize, parse, and start your Ambiorix runtime from C code.

## Why Run ODL from C?

Running ODL files from a C application gives you more control over initialization, configuration, debugging, and integration with other C modules. 

## Example: main.c

Here's a typical flow for running an ODL file from C:

```c
#include "main.h"

#define ODL_FILE "/etc/amx/my-amx/my-amx.odl"

int main(int argc, char *argv[])
{
    amxrt_new();

    amxc_var_t* config = amxrt_get_config();
    amxo_parser_t *parser = amxrt_get_parser();
    amxd_dm_t *dm = amxrt_get_dm();
    amxo_parser_parse_file(parser, ODL_FILE, amxd_dm_get_root(dm));

    amxrt_config_scan_backend_dirs();
    amxrt_connect();
    amxrt_el_create();
    amxrt_register_or_wait();
    amxrt_el_start();

    amxrt_stop();
    amxrt_delete();
    return 0;
}
```

## Steps Explained

1. **Initialize Ambiorix Runtime:**
   - `amxrt_new()` sets up the runtime environment.
2. **Get Configuration and Parser:**
   - Retrieve config, parser, and DM pointers for further use.
3. **Parse the ODL File:**
   - `amxo_parser_parse_file()` loads your ODL file into the DM tree.
4. **Scan Backends and Connect:**
   - Discover and connect to available backends.
5. **Create Event Loop and Register:**
   - Set up the event loop and register the application.
6. **Start the Event Loop:**
   - Begin processing events and requests.
7. **Cleanup:**
   - Stop and delete the runtime before exiting.

## Advantages

- Full control over startup and shutdown
- Easy integration with other C modules
- Customizable configuration and event handling

## Conclusion

By running ODL files from your C application, you unlock advanced integration and automation possibilities for Ambiorix. This method is recommended for embedded systems, custom services, and when you need more than what the CLI tool offers.

---
