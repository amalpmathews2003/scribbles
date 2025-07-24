# Building the my-amx Package for Ambroix PRPL

The `my-amx` package is a custom utility under development for registering AMX methods within the Ambroix PRPL framework. Designed for embedded systems, it leverages the Ambroix ecosystem to provide a flexible solution for method registration, with potential applications in IoT, network configuration, or device management. This blog post dives into the current structure, functionality, and build process of `my-amx`, highlighting its work-in-progress nature and future potential.

## Project Structure

The `my-amx` package is organized as follows:

```
.
├── Makefile
└── src
    ├── CMakeLists.txt
    ├── my-amx.c
    └── odl
        └── my-amx.odl
```

- **Makefile**: Configures the build and installation for an OpenWRT-like environment.
- **CMakeLists.txt**: Manages the build process for the executable and shared library.
- **my-amx.c**: Implements the main AMX functionality, including initialization and cleanup.
- **odl/**: Contains ODL files to configure the Ambroix runtime, with `my-amx.odl` defining the module.

## Key Components

### 1. **my-amx.c: Core Functionality**

The `my-amx.c` file defines the `_my_amx_main` function, the entry point for the Ambroix runtime.

```c
#include <stdio.h>
#include <amxd/amxd_types.h>
#include <amxo/amxo.h>

int _my_amx_main(int reason, amxd_dm_t *dm, amxo_parser_t *parser)
{
    switch (reason)
    {
    case AMXO_START:
        printf("AMXO_START: Initializing my-amx\n");
        break;
    case AMXO_STOP:
        printf("AMXO_STOP: Cleaning up my-amx\n");
        break;
    default:
        break;
    }
    return 0;
}
```

This code handles:
- **AMXO_START**: Initializes the module.
- **AMXO_STOP**: Cleans up resources.

### 2. **Makefile: Building the Package**

The `Makefile` integrates with an OpenWRT-like build system.

```makefile
include $(TOPDIR)/rules.mk

PKG_NAME:=my-amx
PKG_RELEASE:=1

include $(INCLUDE_DIR)/package.mk
include $(INCLUDE_DIR)/cmake.mk

define Package/${PKG_NAME}
	SECTION:=utils
	CATEGORY:=Utilities
	TITLE:=AMX Methods Registration Utility
	DEPENDS:=+libamxo +libamxs +libamxb +libamxd +libubox +libblobmsg-json +ubus
endef

define Package/${PKG_NAME}/install
	$(INSTALL_DIR) $(1)/usr/bin
	$(INSTALL_BIN) $(PKG_BUILD_DIR)/${PKG_NAME} $(1)/usr/bin/${PKG_NAME}
	$(INSTALL_DIR) $(1)/etc/amx/my-amx/odl
	$(INSTALL_DATA) $(PKG_BUILD_DIR)/odl/* $(1)/etc/amx/my-amx/odl/
	
	$(INSTALL_DIR) $(1)/usr/lib/amx/my-amx
	$(INSTALL_DATA) $(PKG_BUILD_DIR)/libmy-amxl.so $(1)/usr/lib/amx/my-amx/my-amx.so
endef

$(eval $(call BuildPackage,${PKG_NAME}))
```

It specifies dependencies (Ambroix and OpenWRT libraries) and installs the executable, shared library, headers, and ODL files.

### 3. **CMakeLists.txt: Configuring the Build**

The `CMakeLists.txt` file sets up the build for the executable and shared library.

```cmake
cmake_minimum_required(VERSION 3.10)
project(my-amx C)

set(STAGING_DIR <TODO>/staging_dir/target-x86_64_musl)
set(AMX_INCLUDE_DIRS ${STAGING_DIR}/usr/include)
set(AMX_LIBRARIES ${STAGING_DIR}/usr/lib/libamxs.so
                  ${STAGING_DIR}/usr/lib/libamxo.so
                  ${STAGING_DIR}/usr/lib/libamxb.so
                  ${STAGING_DIR}/usr/lib/libamxd.so)

add_executable(my-amx main.c)
target_include_directories(my-amx PRIVATE ${AMX_INCLUDE_DIRS})
target_link_libraries(my-amx ${AMX_LIBRARIES})

add_library(my-amxl SHARED my-amx.c)
target_include_directories(my-amxl PRIVATE ${AMX_INCLUDE_DIRS})
target_link_libraries(my-amxl ${AMX_LIBRARIES})

install(TARGETS my-amx DESTINATION bin)
```

It targets a cross-compilation environment for `x86_64_musl`, building `my-amx` and `libmy-amxl.so`.

### 4. **my-amx.odl: Configuring the Ambroix Runtime**

The `my-amx.odl` file configures the Ambroix runtime.

```odl
#!/usr/bin/amxrt

%config {
    name = my-amx;
    so-dir = "/usr/lib/amx/${name}";
    so-name = "${name}";
}

import "${so-dir}/${so-name}.so" as "${name}";

%define {
    entry-point my-amx.my_amx_main;
}
```

It loads the `my-amx.so` shared library and sets `_my_amx_main` as the entry point. The `my-amx-definition.odl` file, still in development, will likely define specific AMX methods or objects.

## Building and Installing

To build and install:
1. Configure the OpenWRT build environment (`TOPDIR`, `INCLUDE_DIR`).
2. Run `make` to build the executable and shared library.
3. Artifacts will be installed in `/usr/bin`, `/usr/lib/amx/my-amx`, `/usr/include/my-amx`, and `/etc/amx/my-amx/odl`.
4. Run the odl file `amxrt` (Ambiorix run time).

