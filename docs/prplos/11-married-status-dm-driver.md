# Ambiorix: Adding a Read-Write DM with Dummy Driver, LLAPI, and HLAPI

This post demonstrates how to add a new read-write Data Model (DM) in Ambiorix, invoke a dummy driver call, and log the operation. We'll also show how to implement both low-level (LLAPI) and high-level (HLAPI) APIs, and extend your existing module for integration.

## 1. Define the New Read-Write DM

Add a boolean DM called `married_status`, which is read-write and triggers a handler on write:

```c
bool married_status {
    default false;
    on action write call handle_married_status_write;
}
```

## 2. Dummy Driver Call Implementation

Create a dummy driver function to simulate hardware/system interaction and log the call:

```c
int dummy_driver_set_married_status(bool status) {
    printf("Dummy driver called with married_status: %s\n", status ? "true" : "false");
    // Simulate success
    return 0;
}
```

## 3. Low-Level API (LLAPI)

The LLAPI provides direct access to the driver function:

```c
int llapi_set_married_status(bool status) {
    return dummy_driver_set_married_status(status);
}
```

## 4. High-Level API (HLAPI)

The HLAPI wraps the LLAPI, adds logging, and can include validation or business logic:

```c
int hlapi_set_married_status(bool status) {
    printf("HLAPI: Setting married_status to %s\n", status ? "true" : "false");
    return llapi_set_married_status(status);
}
```

## 5. DM Write Handler

Use the HLAPI in your DM write handler:

```c
void handle_married_status_write(const amxd_object_t* object, bool status) {
    int result = hlapi_set_married_status(status);
    if(result == 0) {
        printf("married_status set successfully\n");
    } else {
        printf("Failed to set married_status\n");
    }
}
```

## 6. Event Handler for DM Changes

Add an event handler to react to changes in `married_status`:

```c
on event "dm:object-changed" call on_married_status_change
    filter 'path == "Person." and contains("parameters.married_status")';
```

Example implementation:

```c
void _on_married_status_change(const char *const event_name,
                               const amxc_var_t *const event_data,
                               void *const priv)
{
    amxd_object_t *person = amxd_dm_findf(my_amx_get_dm(), "%s", "Person.");
    bool married_status = amxd_object_get_value(bool, person, "married_status", NULL);
    printf("Happy for you, you are %s!\n", married_status ? "married" : "single");

    amxc_var_t data, ret;
    amxc_var_init(&data);
    amxc_var_set(cstring_t, &data, "married_status changed");
    amxc_var_init(&ret);

    amxm_shared_object_t *so = amxm_so_get_current();
    amxm_execute_function(so->name, "my-amx", "custom_married_status_function", &data, &ret);

    printf("Function returned: %s\n", amxc_var_dyncast(cstring_t, &ret));
    amxc_var_clean(&data);
    amxc_var_clean(&ret);
}
```

## 7. Custom Function Registration

Register your custom function in the module initialization:

```c
amxm_shared_object_t *so = amxm_so_get_current();
amxm_module_t *mod = NULL;

amxm_module_register(&mod, so, "my-amx");
amxm_module_add_function(mod,
                         "custom_married_status_function",
                         custom_married_status_function);
```

## 8. Custom Function Example

```c
int custom_married_status_function(UNUSED const char *function_name,
                                   amxc_var_t *args,
                                   amxc_var_t *ret)
{
    printf("Custom function called with args: %s\n",
           amxc_var_dyncast(cstring_t, args));

    amxc_var_set(cstring_t, ret, "Custom married status function executed");
    return 0;
}
```

## 9. Extending the Existing Module

To integrate this new DM and driver call:
- Add the DM definition for `married_status`.
- Implement the dummy driver, LLAPI, HLAPI, handler, and event functions.
- Register the custom function in your module.

This modular approach keeps your code organized and makes it easy to add real driver logic in the future.

---
