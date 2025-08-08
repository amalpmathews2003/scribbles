# Ambiorix: Linked List Implementation for Life Events

This post demonstrates how to implement a linked list in Ambiorix using `amxc_llist` to manage a dynamic collection of life events. We'll show how to define the DM, initialize the list, add, retrieve, print, and clear events, and expose these operations as Ambiorix functions.

## 1. DM Object Definition

Define your life events object and functions in the DM:

```c
object LifeEvents {
    void initLE();
    uint16 append_event(%in %mandatory string name, %in %mandatory int16 year, %in %mandatory string details);
    void retrieve_event(%in %mandatory %strict uint16 index);
    void clearLE();
    void print_events();
}
```

## 2. Linked List Implementation in C

Use `amxc_llist` to store and manage life events:

```c
#include <amxc/amxc_llist.h>
#include <string.h>

static amxc_llist_t life_events;
static bool is_ll_initialized = false;

typedef struct _life_event {
    char event_name[20];
    uint16_t year;
    char details[50];
    amxc_llist_it_t it;
} life_event_t;

bool initLE() {
    if (is_ll_initialized) return true;
    amxc_llist_init(&life_events);
    is_ll_initialized = true;
    return true;
}

bool append_event(life_event_t *event) {
    amxc_llist_append(&life_events, &event->it);
    return true;
}

life_event_t *retrieve_event(size_t index) {
    if (index >= amxc_llist_size(&life_events)) return NULL;
    amxc_llist_it_t *it = amxc_llist_get_at(&life_events, index);
    return amxc_container_of(it, life_event_t, it);
}

void print_events() {
    int count = amxc_llist_size(&life_events);
    printf("Total Life Events: %d\n", count);
    amxc_llist_for_each(it, &life_events) {
        life_event_t *event = amxc_container_of(it, life_event_t, it);
        printf("Event Name: %s, Year: %d, Details: %s\n", event->event_name, event->year, event->details);
    }
}

void clearLE() {
    amxc_llist_clean(&life_events, free);
    is_ll_initialized = false;
}
```

## 3. Exposing Functions to Ambiorix

Wrap your linked list logic in Ambiorix-callable functions:

```c
amxd_status_t _initLE() {
    initLE();
    return amxd_status_ok;
}

amxd_status_t _append_event(amxd_object_t *object, amxd_function_t *func, amxc_var_t *args, amxc_var_t *ret) {
    initLE();
    const char *name = GET_CHAR(args, "name");
    uint16_t year = GET_INT32(args, "year");
    const char *details = GET_CHAR(args, "details");
    life_event_t *event = calloc(1, sizeof(life_event_t));
    snprintf(event->event_name, 20, "%s", name);
    event->year = year;
    snprintf(event->details, 50, "%s", details);
    append_event(event);
    amxc_var_set(uint16_t, ret, amxc_llist_size(&life_events));
    return amxd_status_ok;
}

amxd_status_t _retrieve_event(amxd_object_t *object, amxd_function_t *func, amxc_var_t *args, amxc_var_t *ret) {
    size_t index = GET_INT32(args, "index");
    life_event_t *event = retrieve_event(index);
    if (!event) return amxd_status_unknown_error;
    amxc_var_set(cstring_t, ret, event->event_name);
    amxc_var_set(uint16_t, ret, event->year);
    amxc_var_set(cstring_t, ret, event->details);
    return amxd_status_ok;
}

amxd_status_t _print_events(amxd_object_t *object, amxd_function_t *func, amxc_var_t *args, amxc_var_t *ret) {
    print_events();
    return amxd_status_ok;
}

amxd_status_t _clearLE() {
    clearLE();
    return amxd_status_ok;
}
```

## 4. Example Usage

Add, retrieve, print, and clear life events using Ambiorix CLI or API:

```sh
# Initialize the list
ubus call LifeEvents initLE 

# Append an event
ubus call LifeEvents append_event '{"name":"Graduation", "year":2020, "details":"Completed B.Tech"}'

# Retrieve an event
ubus call LifeEvents retrieve_event '{"index":0}'

# Print all events
ubus call LifeEvents print_events 

# Clear all events
ubus call LifeEvents clearLE 
```

## Conclusion

Using `amxc_llist`, you can efficiently implement a dynamic list in Ambiorix, with full DM integration and flexible function exposure for real-world use cases.

---