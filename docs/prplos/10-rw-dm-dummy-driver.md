# Extending Ambiorix: Adding a Read-Write DM with Dummy Driver Integration

In this post, we'll walk through how to add a new read-write Data Model (DM) in Ambiorix that invokes a dummy driver call and logs the operation. We'll also cover how to extend your existing module, and how to implement both low-level (LLAPI) and high-level (HLAPI) APIs for this functionality.

## 1. Define the New Read-Write DM

Start by adding a new boolean DM, for example, `married_status`, which is read-write and triggers a handler on write:

```c
bool married_status {
    default false;
    on action write call handle_married_status_write;
}
```

## 2. Implement the Dummy Driver Call

Extend your module to include a dummy driver function for `married_status`. This function will simulate a hardware or system call and log the invocation:

```c
// Dummy driver function
int dummy_driver_set_married_status(bool status) {
    printf("Dummy driver called with married_status: %s\n", status ? "true" : "false");
    // Simulate success
    return 0;
}
```

## 3. Write the Low-Level API (LLAPI)

The LLAPI provides direct access to the driver function. It can be called from your DM handler or other internal logic:

```c
int llapi_set_married_status(bool status) {
    return dummy_driver_set_married_status(status);
}
```

## 4. Write the High-Level API (HLAPI)

The HLAPI wraps the LLAPI and can include additional logic, validation, or integration with the DM system:

```c
int hlapi_set_married_status(bool status) {
    printf("HLAPI: Setting married_status to %s\n", status ? "true" : "false");
    return llapi_set_married_status(status);
}
```

## 5. DM Write Handler Implementation

Update your DM write handler to use the HLAPI:

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

## 6. Extending the Existing Module

To integrate this new DM and driver call, extend your existing module by:
- Adding the new DM definition for `married_status`.
- Implementing the dummy driver, LLAPI, HLAPI, and handler functions as shown above.
- Registering the handler with the DM system.

You can also add an event handler for changes to `married_status`:

```c
on event "dm:object-changed" call on_married_status_change
    filter 'path == "Person." and contains("parameters.married_status")';
```

This modular approach keeps your code organized and makes it easy to add real driver logic in the future.

## Conclusion

By following these steps, you can add a new read-write DM in Ambiorix that invokes a dummy driver call and logs the operation. The separation of LLAPI and HLAPI ensures flexibility and maintainability, while extending your existing module keeps your codebase clean and scalable.

---
