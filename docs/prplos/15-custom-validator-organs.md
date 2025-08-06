# Ambiorix: Implementing a Custom Validator for DM Parameters

This post demonstrates how to implement a custom validator for a Data Model (DM) parameter in Ambiorix. We'll show how to validate the value of a string parameter against a list of allowed organ names using a custom function.

## 1. DM Object and Parameter Definition

Define your DM object and parameter with a custom validator:

```c
object Organs[] {
    string kind = "" {
        on action validate call check_is_valid_organ;
    }
    bool still_kicking = false;
}
```

## 2. Custom Validator Implementation

The custom validator checks if the value of `kind` matches any allowed organ name (case-insensitive):

```c
int stricmp(const char *s1, const char *s2) {
    while (*s1 && *s2) {
        if (tolower((unsigned char)*s1) != tolower((unsigned char)*s2)) {
            break;
        }
        s1++;
        s2++;
    }
    return (unsigned char)*s1 - (unsigned char)*s2;
}

amxd_status_t _check_is_valid_organ(amxd_object_t *object, amxd_path_t *param,
                                    amxd_action_t reason, const amxc_var_t *const args,
                                    amxc_var_t *const retval, UNUSED void *priv)
{
    const char* organ = amxc_var_constcast(cstring_t, args);
    if (organ == NULL || organ[0] == '\0') {
        return amxd_status_ok;
    }
    printf("Validating organ: %s\n", organ);
    const char *organs[] = {
        "",
        "Heart",
        "Liver",
        "Kidney",
        "Lungs",
        "Brain",
    };
    for (size_t i = 0; i < sizeof(organs) / sizeof(organs[0]); i++) {
        if (stricmp(organ, organs[i]) == 0) {
            return amxd_status_ok;
        }
    }
    return amxd_status_invalid_value;
}
```

## 3. How It Works

- When the `kind` parameter is set, the validator is called.
- The validator checks if the value matches any allowed organ name (case-insensitive).
- If valid, the value is accepted; otherwise, an invalid value status is returned.

## 4. Integration Steps

- Add the parameter definition and validator reference to your ODL or DM source.
- Implement the validator in your C source (e.g., in your `my-amx` module).
- Register the validator with Ambiorix so it is invoked on validate actions.

## 5. Example Usage

When you set the `kind` parameter, validation occurs automatically:

```sh
# Example CLI or API call
ubus call Person.Organs _add '{"parameters":{"kind":"liber"}}'
{
        "retval": ""
}

{
        "amxd-error-code": 10
}
Command failed: Invalid argument

```

## Conclusion

Custom validators in Ambiorix allow you to enforce business rules and data integrity for DM parameters. This pattern is powerful for input validation, security, and robust application logic.

---

