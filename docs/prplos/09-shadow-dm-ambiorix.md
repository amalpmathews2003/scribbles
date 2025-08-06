# How to Create a Read-Only Shadow Data Model in Ambiorix

In this post, you'll learn how to create a read-only Data Model (DM) in Ambiorix that automatically mirrors the value of a settable DM. This technique is useful for scenarios where you want to expose a value for monitoring or auditing, but prevent direct modification. We'll walk through the DM definition, event handling, and initialization, ensuring the read-only DM always stays in sync with the settable DM.

## 1. Define the Data Model

First, define your DMs. The `shadow_name` DM is marked as read-only, meaning it cannot be set directly by external actions. The `name` DM is settable and will be the source of truth. Whenever `name` changes, `shadow_name` will be updated to reflect the new value.

```c
%read-only string shadow_name {
    default "";
}
string name {
    on action write call handle_name_write;
}
```

## 2. Handle Changes with Events

To keep `shadow_name` updated, you need to react to changes in `name`. Ambiorix allows you to listen for events on the data model. Here, we set up an event handler for the `dm:object-changed` event, filtered so it only triggers when the `name` parameter of the `Person` object changes. This ensures efficient and targeted updates.

```c
on event "dm:object-changed" call on_person_name_change
    filter 'path == "Person." and contains("parameters.name")';
```

## 3. Implement the Event Handler

The event handler function is called whenever the `name` DM changes. Inside the handler, you:

1. Find the relevant object in the DM tree (`Person.`).
2. Retrieve the new value of `name`.
3. Start a transaction, marking it as a change to a read-only attribute.
4. Select the object and set the value of `shadow_name` to match `name`.
5. Apply the transaction, updating the DM.

This approach ensures that `shadow_name` is always synchronized with `name`, but remains protected from direct modification.

```c
void _on_person_name_change(const char *const event_name,
                            const amxc_var_t *const event_data,
                            void *const priv)
{
    printf("Person name changed\n", event_name);
    amxd_object_t *person = amxd_dm_findf(my_amx_get_dm(), "%s", "Person.");

    char *name = amxd_object_get_value(cstring_t, person, "name", NULL);

    amxd_trans_t trans;
    amxd_trans_init(&trans);
    amxd_trans_set_attr(&trans, amxd_tattr_change_ro, true);
    amxd_trans_select_object(&trans, person);
    amxd_trans_set_cstring_t(&trans, "shadow_name", name);
    amxd_trans_apply(&trans, my_amx_get_dm());
}
```

## 4. Initial Value from ODL

It's important that `shadow_name` is consistent from the very beginning. During initialization, set its value from ODL so it matches the initial value of `name`. This prevents any mismatch between the two DMs and guarantees that your shadow DM is always accurate, even before any changes occur.



## Conclusion

By combining DM definitions, event-driven updates, and careful initialization, you can create robust read-only shadow DMs in Ambiorix. This pattern is valuable for scenarios where you need to expose internal state without allowing direct modification, such as audit logs, monitoring, or enforcing business rules. The synchronization logic ensures your shadow DM is always up-to-date and reliable.

---

