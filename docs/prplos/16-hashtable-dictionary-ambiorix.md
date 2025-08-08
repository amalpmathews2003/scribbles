# Ambiorix: Implementing a Hashtable-backed Dictionary with amxc_htable

This post demonstrates how to implement a dictionary (key-value store) in Ambiorix using the `amxc_htable` hashtable API. We'll show how to define the DM, initialize the hashtable, add and retrieve words, and expose these operations as Ambiorix functions.

## 1. DM Object Definition

Define your dictionary object and entry structure in the DM:

```c
object Dict {
    object Entry[] {
        string word;
        string meaning;
        on action add-inst call add_word_instance;
    }
    void init_ht();
    void add_word(%in string word, %in string meaning);
    void get_meaning(%in string word);
    void display_dict();
}
```

## 2. Hashtable Implementation in C

Use `amxc_htable` to store and manage dictionary entries:

```c
#include <amxc/amxc_htable.h>
#include <string.h>

static amxc_htable_t dict;
static bool dict_initialised = false;

typedef struct _entry {
    char meaning[20];
    amxc_htable_it_t it;
} entry_t;

int _init_ht() {
    if (dict_initialised) return 0;
    amxc_htable_init(&dict, 10);
    dict_initialised = true;
    return 0;
}

int add_word(const char *word, const char *meaning) {
    entry_t *entry = calloc(1, sizeof(entry_t));
    strcpy(entry->meaning, meaning);
    amxc_htable_it_init(&entry->it);
    int status = amxc_htable_insert(&dict, word, &entry->it);
    printf("status %d\n", status);
    return 0;
}

char *get_meaning(const char *word) {
    amxc_htable_it_t *it = amxc_htable_get(&dict, word);
    if (it != NULL) {
        entry_t *entry = amxc_container_of(it, entry_t, it);
        return entry->meaning;
    }
    return NULL;
}
```

## 3. Exposing Functions to Ambiorix

Wrap your hashtable logic in Ambiorix-callable functions:

```c
void _add_word(UNUSED amxd_object_t *object, UNUSED amxd_function_t *func, amxc_var_t *args, UNUSED amxc_var_t *ret) {
    const char *word = GET_CHAR(args, "word");
    const char *meaning = GET_CHAR(args, "meaning");
    _init_ht();
    if (!word || !meaning) return;
    add_word(word, meaning);
}

void _get_meaning(UNUSED amxd_object_t *object, UNUSED amxd_function_t *func, amxc_var_t *args, amxc_var_t *ret) {
    const char *word = GET_CHAR(args, "word");
    const char *meaning = get_meaning(word);
    amxc_var_set_type(ret, AMXC_VAR_ID_HTABLE);
    amxc_var_add_key(cstring_t, ret, "word", meaning);
}

void _display_dict(UNUSED amxd_object_t *object, UNUSED amxd_function_t *func, UNUSED amxc_var_t *args, UNUSED amxc_var_t *ret) {
    amxc_htable_iterate(it, &dict) {
        entry_t *entry = amxc_container_of(it, entry_t, it);
        printf("Word: %s, Meaning: %s\n", it->key, entry->meaning);
    }
    printf("Dictionary contents displayed.\n");
}
```

## 4. Adding Entries via Instance Action

Support adding entries as DM instances:

```c
amxd_status_t _add_word_instance(amxd_object_t *object, amxd_param_t *param, amxd_action_t reason, const amxc_var_t *const args, amxc_var_t *const retval, void *priv) {
    amxc_var_t *parameters = GET_ARG(args, "parameters");
    const char *word = GET_CHAR(parameters, "word");
    const char *meaning = GET_CHAR(parameters, "meaning");
    _init_ht();
    if (!word || !meaning) return amxd_status_invalid_arg;
    add_word(word, meaning);
    amxd_action_object_add_inst(object, param, reason, args, retval, priv);
    return amxd_status_ok;
}
```

## 5. Example Usage

Add and retrieve words using Ambiorix CLI or API:

```sh
# Add a word
ubus call Dict add_word '{"word":"apple", "meaning":"fruit"}'

# Get meaning
ubus call Dict get_meaning '{"word":"apple"}'
# Response: { "word": "fruit" }

# Display dictionary
ubus call Dict display_dict '{}'
```

## Conclusion

Using `amxc_htable`, you can efficiently implement a dictionary or key-value store in Ambiorix, with full DM integration and flexible function exposure.

---
