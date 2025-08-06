# Ambiorix: Adding a DM with LLAPI, and HLAPI Within Existing Module

This post demonstrates how to add a new read-write Data Model (DM) in Ambiorix, invoke a dummy driver call, and log the operation. We'll also show how to implement both low-level (LLAPI) and high-level (HLAPI) APIs, and extend your existing module for integration.

## 1. Define the New Read-Write DM

Add a boolean DM called `married_status`, which is read-write and triggers a handler on write:

```c
bool married_status {
    default false;
}
```

## 2. Event Handler for DM Changes

Add an event handler to react to changes in `married_status`:

```c
on event "dm:object-changed" call on_married_status_change
    filter 'path == "Person." and contains("parameters.married_status")';
```
Event handler function:

```c
void _on_married_status_change(const char *const event_name,
                               const amxc_var_t *const event_data,
                               void *const priv)
{
    amxd_object_t *person = amxd_dm_findf(my_amx_get_dm(), "%s", "Person.");
    bool married_status = amxd_object_get_value(bool, person, "married_status", NULL);
    printf("Happy for you, you are %s!\n", married_status ? "married" : "single");

    hlapi_set_married_status(married_status);
}
```

## 3. Low-Level API (LLAPI)

The LLAPI provides direct access to the driver function:

```c
int custom_married_status_function_ll(UNUSED const char *function_name,
                                   amxc_var_t *args,
                                   amxc_var_t *ret)
{
    printf("LL Custom function called with args: %s\n",
           amxc_var_dyncast(cstring_t, args));

    amxc_var_set(cstring_t, ret, "LL Custom married status function executed");
    return 0;
}
```

## 4. LL API Function Registration

Register your custom function in the module initialization:

```c
amxm_shared_object_t *so = amxm_so_get_current();
amxm_module_t *mod = NULL;

amxm_module_register(&mod, so, "my-amx");
amxm_module_add_function(mod,
                         "custom_married_status_function",
                         custom_married_status_function_ll);
```

## 5. High-Level API (HLAPI)

The HLAPI wraps the LLAPI, adds logging, and can include validation or business logic:

```c
int hlapi_set_married_status(bool status) {
    printf("HLAPI: Setting married_status to %s\n", status ? "true" : "false");
   
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


## 6. Function Control Flow

Here's how the function control flow works when `married_status` is changed:

1. **DM Change:** The value of `married_status` is updated in the data model.
2. **Event Handler Triggered:** The `dm:object-changed` event fires and calls `_on_married_status_change`.
3. **HLAPI Called:** Inside the event handler, `hlapi_set_married_status(married_status)` is invoked.
4. **HLAPI Logic:** The HLAPI logs the operation, prepares arguments, and calls the registered LLAPI function using `amxm_execute_function`.
5. **LLAPI Executed:** The LLAPI (`custom_married_status_function_ll`) performs the low-level logic and returns a result.
6. **Result Logging:** The HLAPI logs the result and cleans up.

---