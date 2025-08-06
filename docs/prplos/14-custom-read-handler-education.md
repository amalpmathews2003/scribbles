# Ambiorix: Implementing a Custom Read Handler for DM Parameters

This post demonstrates how to implement a custom read handler for a Data Model (DM) parameter in Ambiorix. We'll show how to return a random value from a predefined list when the parameter is read, using the example of an `education` string parameter.

## 1. DM Parameter Definition

Define your DM parameter with a custom read handler:

```c
string education {
    on action read call read_education;
}
```

## 2. Custom Read Handler Implementation

The custom read handler function returns a random education from a list:

```c
amxd_status_t _read_education(UNUSED amxd_object_t *object,
                              UNUSED amxd_param_t *param,
                              amxd_action_t reason,
                              UNUSED const amxc_var_t *const args,
                              amxc_var_t *const retval,
                              UNUSED void *priv)
{
    // List of educations
    static const char *educations[] = {
        "B.Tech",
        "M.Tech",
        "MCA",
        "BCA",
        "B.Sc",
    };
    amxd_status_t status = amxd_status_ok;
    // Generate a random index to select an education
    size_t index = rand() % (sizeof(educations) / sizeof(educations[0]));
    const char *education = educations[index];

    amxc_var_set(cstring_t, retval, education);
    return status;
}
```

## 3. How It Works

- When the `education` parameter is read, the handler is called.
- The handler randomly selects one value from the list and returns it.
- This allows dynamic responses for DM reads, useful for testing, demos, or simulating real-world variability.

## 4. Integration Steps

- Add the parameter definition to your ODL or DM source.
- Implement the handler in your C source (e.g., in your `my-amx` module).
- Register the handler with Ambiorix so it is invoked on read actions.

## 5. Example Usage

When you query the `education` parameter, you'll get a random value each time:

```sh
ubus call Person _get '{"parameters":["education"]}'
```

**Example response:**

```json
{
    "Person.": {
        "education": "M.Tech"
    }
}
```

## Conclusion

Custom read handlers in Ambiorix allow you to control and customize the behavior of DM parameters. This pattern is powerful for dynamic data, simulations, and advanced integrations.

---

