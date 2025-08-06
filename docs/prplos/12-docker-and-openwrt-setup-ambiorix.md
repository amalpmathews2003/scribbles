# Setting Up Ambiorix Project for Docker & OpenWrt Build

This guide walks you through preparing your Ambiorix project for Docker-based builds, referencing changes to CMake, Makefiles, and folder structure in the `my-amx` folder. The same approach can be adapted for OpenWrt builds, making your project portable across standard Linux and OpenWrt environments.

## 1. Organize Your Project Structure

A typical Ambiorix project should have a clear folder structure. For example:

```
my-amx/
├── CMakeLists.txt
├── Makefile
├── src/
│   ├── CMakeLists.txt
│   ├── my-amx.c
│   └── ...
├── include_priv/
│   └── my-amx.h
├── odl/
├── mod-custom/
├── build/
└── ...
```

- `src/`: Source files
- `include_priv/`: Header files
- `build/`: Output and intermediate files
- `CMakeLists.txt` and `Makefile`: Build configuration

## 2. Update Root CMakeLists.txt

Ensure your `CMakeLists.txt` includes all necessary source and header files, and sets up the build target:

```cmake
cmake_minimum_required(VERSION 3.10)
project(my-amx C)

set(CMAKE_C_STANDARD 11)
set(CMAKE_C_FLAGS_DEBUG "-g -O0 -DDEBUG")

# Detect if building for OpenWrt
if(DEFINED ENV{STAGING_DIR})
    message(STATUS "Cross-compiling for OpenWrt")
    set(STAGING_DIR $ENV{STAGING_DIR})
    set(AMX_INCLUDE_DIRS ${STAGING_DIR}/usr/include)
    set(AMX_INCLUDE_DIRS ${AMX_INCLUDE_DIRS} ${PROJECT_SOURCE_DIR}/include_priv)
    set(AMX_LIBRARIES
        ${STAGING_DIR}/usr/lib/libamxs.so
        ${STAGING_DIR}/usr/lib/libamxo.so
        ${STAGING_DIR}/usr/lib/libamxb.so
        ${STAGING_DIR}/usr/lib/libamxd.so
        ${STAGING_DIR}/usr/lib/libamxrt.so
        ${STAGING_DIR}/usr/lib/libamxc.so
        ${STAGING_DIR}/usr/lib/libamxm.so
        ${STAGING_DIR}/lib/libsahtrace.so
    )
else()
    message(STATUS "Native or Docker build")
    set(AMX_INCLUDE_DIRS
        ${PROJECT_SOURCE_DIR}/include_priv
    )
    set(AMX_LIBRARIES
        -lamxc -lamxb -lamxrt -lamxo -lamxd -lamxm -lsahtrace
    )
endif()


# Pass variables to sub-CMake
add_subdirectory(src)



# Optional subdirectory (optional if exists)
if(EXISTS "${CMAKE_CURRENT_SOURCE_DIR}/mod-custom")
    add_subdirectory(mod-custom)
endif()

```

## 3. Update Makefile

A simple Makefile for building your project:

```makefile
include $(TOPDIR)/rules.mk

PKG_NAME:=my-amx
PKG_RELEASE:=1

SOURCE_DIR:=~/amal/prpl/custom/my-amx

include $(INCLUDE_DIR)/package.mk
include $(INCLUDE_DIR)/cmake.mk

CMAKE_OPTIONS+= -DCMAKE_BUILD_TYPE=Debug

define Package/${PKG_NAME}
	SECTION:=utils
	CATEGORY:=Utilities
	TITLE:=AMX Methods Registration Utility
	DEPENDS:=+libamxo +libamxs +libamxb +libamxd 
	DEPENDS+=+libamxrt +libamxm +libubox +libblobmsg-json +ubus
	DEPENDS+=libsahtrace
endef

define Build/Prepare
	mkdir -p $(PKG_BUILD_DIR)
	$(CP) -r $(SOURCE_DIR)/* $(PKG_BUILD_DIR)/
	$(Build/Patch)
endef

define Package/${PKG_NAME}/install
	$(INSTALL_DIR) $(1)/usr/bin
	$(INSTALL_BIN) $(PKG_INSTALL_DIR)/usr/bin/${PKG_NAME} $(1)/usr/bin/${PKG_NAME}

	$(INSTALL_DIR) $(1)/etc/amx/${PKG_NAME}
	$(INSTALL_DATA) $(PKG_BUILD_DIR)/odl/* $(1)/etc/amx/${PKG_NAME}

	$(INSTALL_DIR) $(1)/usr/lib/amx/${PKG_NAME}
	$(CP) $(PKG_INSTALL_DIR)/usr/lib/amx/${PKG_NAME}/* $(1)/usr/lib/amx/${PKG_NAME}/

endef

$(eval $(call BuildPackage,${PKG_NAME}))

```

## 2. Update Src CMakeLists.txt
Ensure your `CMakeLists.txt` includes all necessary source and header files, and sets up the build target:

```cmake
include_directories(${AMX_INCLUDE_DIRS})

# Main executable
add_executable(my-amx main.c)
target_link_libraries(my-amx ${AMX_LIBRARIES})


# Shared library
add_library(my-amxl SHARED
    my-amx.c
    ...
)
target_link_libraries(my-amxl ${AMX_LIBRARIES})
set_target_properties(my-amxl PROPERTIES PREFIX "" OUTPUT_NAME "my-amx")


# Install target for OpenWrt only
if(DEFINED ENV{STAGING_DIR})
    install(TARGETS my-amx DESTINATION bin)
endif()

install(TARGETS my-amx RUNTIME DESTINATION bin)
install(TARGETS my-amxl LIBRARY DESTINATION lib/amx/my-amx)

```

## 6. Tips for Integration

- Keep your build scripts and Dockerfile up to date with any changes in your source or dependencies.
- Use `.dockerignore` to exclude unnecessary files from the build context.
- Test your build locally before pushing to CI/CD or production.
- For OpenWrt builds, adapt your Dockerfile to use OpenWrt toolchains and SDKs. You can mount or copy the OpenWrt SDK into your Docker image and use it for cross-compilation.

---

