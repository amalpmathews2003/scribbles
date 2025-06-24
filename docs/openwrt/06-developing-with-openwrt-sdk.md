# Developing with OpenWRT SDK

This guide explains how to build and use the OpenWRT Software Development Kit (SDK) from the OpenWRT source tree using `make menuconfig`, and develop a custom "Hello, World!" package, following the [OpenWRT SDK Guide](https://openwrt.org/docs/guide-developer/toolchain/using_the_sdk) and [Hello World Tutorial](https://openwrt.org/docs/guide-developer/helloworld/start). It covers why the SDK is beneficial, its use cases, and step-by-step instructions for building the SDK, creating a package, building, deploying, and testing.

## Why Use the OpenWRT SDK?

The OpenWRT SDK is a streamlined tool for developing applications without compiling the entire OpenWRT firmware. Its key advantages include:

- Efficiency: Pre-configured with a toolchain and libraries for a specific architecture, reducing setup time.
- Lightweight: Smaller footprint than the full OpenWRT source tree, ideal for application development on resource-constrained systems.
- Flexibility: Allows developers to create custom packages (e.g., utilities, services) that integrate with OpenWRT’s package manager (`apk`).
- Portability: Packages built with the SDK can be deployed to any OpenWRT device with a compatible architecture.
- Simplified Workflow: Focuses on application development without managing kernel or firmware components.

### Best Use Cases
- Custom Applications: Develop utilities, daemons, or scripts for OpenWRT devices (e.g., network tools, IoT applications).
- Third-Party Software: Port existing software to OpenWRT as installable packages.
- Prototyping: Test applications on OpenWRT without modifying the full firmware.
- CI/CD Pipelines: Integrate with automated build systems for rapid package development.
- Embedded Development: Create lightweight applications for routers, IoT gateways, or embedded devices running OpenWRT.

## Building the OpenWRT SDK from Menuconfig

1. Set Up OpenWRT Build Environment:
   - Clone the OpenWRT source code and prepare the build environment (see Module 3).
     ```bash
     git clone https://github.com/openwrt/openwrt.git
     cd openwrt
     ./scripts/feeds update -a
     ./scripts/feeds install -a
     ```

2. Configure the SDK Build:
   - Run `make menuconfig` to configure the build:
     ```bash
     make menuconfig
     ```
   - Select the target architecture for your device:
     - Navigate to `Target System` (e.g., `x86` for x86_64).
     - Select `Subtarget` (e.g., `x86_64`).
   - Enable SDK generation:
     - Navigate to `Build the OpenWrt SDK` and mark it with `*`.
   - Save and exit.

3. Build the SDK:
   - Compile the SDK:
     ```bash
     make -j$(nproc)
     ```
   - Output: The SDK archive is generated in `bin/targets/<target>/<subtarget>/`, e.g., `openwrt-sdk-23.05.5-x86_64_gcc-14.3.0_musl.Linux-x86_64.tar.zst`.

4. Extract the SDK:
   - Navigate to the output directory:
     ```bash
     cd bin/targets/x86/64
     ```
   - Extract the archive:
     ```bash
     tar --zstd -xvf openwrt-sdk-23.05.5-x86_64_gcc-14.3.0_musl.Linux-x86_64.tar.zst
     ```

5. Navigate to the SDK Directory:
   ```bash
   cd openwrt-sdk-23.05.5-x86_64_gcc-14.3.0_musl.Linux-x86_64
   ```

## Creating the Hello World Package

1. Create Package Directory:
   - Inside the SDK directory, create a directory for the custom package:
     ```bash
     mkdir -p mypackages/examples/helloworld
     ```

2. Write the C Program:
   - Create `mypackages/examples/helloworld/helloworld.c` with the following content:
     ```c
     #include <stdio.h>

     int main(int argc, char *args[])
     {
         if (argc < 2)
         {
             printf("Usage: %s <name>\n", args[0]);
             return 1;
         }
         printf("Hello, %s!\n", args[1]);
         return 0;
     }
     ```

3. Create the Package Manifest (Makefile):
   - Create `mypackages/examples/helloworld/Makefile` with the following content:
     ```make
      include $(TOPDIR)/rules.mk

      # Name, version, and release number
      PKG_NAME:=helloworld
      PKG_VERSION:=1.0
      PKG_RELEASE:=1

      # Source settings
      SOURCE_DIR:=$(TOPDIR)/my-packages/helloworld

      include $(INCLUDE_DIR)/package.mk

      # Package definition
      define Package/helloworld
          SECTION:=examples
          CATEGORY:=Examples
          TITLE:=Hello World
      endef

      # Package description
      define Package/helloworld/description
          A simple "Hello, world!" application.
      endef

      # Package preparation
      define Build/Prepare
          mkdir -p $(PKG_BUILD_DIR)
          $(CP) $(SOURCE_DIR)/* $(PKG_BUILD_DIR)/
          $(Build/Patch)
      endef

      # Package build instructions
      define Build/Compile
          $(TARGET_CC) $(TARGET_CFLAGS) -o $(PKG_BUILD_DIR)/helloworld.o -c $(PKG_BUILD_DIR)/hello.c
          $(TARGET_CC) $(TARGET_LDFLAGS) -o $(PKG_BUILD_DIR)/helloworld $(PKG_BUILD_DIR)/helloworld.o
      endef

      # Package install instructions
      define Package/helloworld/install
          $(INSTALL_DIR) $(1)/usr/bin
          $(INSTALL_BIN) $(PKG_BUILD_DIR)/helloworld $(1)/usr/bin/
      endef

      # Register the package
      $(eval $(call BuildPackage,helloworld))
     ```

## Including the New Package Feed

1. Create a Feeds Configuration:
   - Create or edit `feeds.conf` in the SDK root directory:
     ```bash
     touch feeds.conf
     ```
   - Add the custom feed:
     ```bash
     echo "src-link mypackages $(pwd)/mypackages" >> feeds.conf
     ```

2. Update and Install the Feed:
   - Update the feed index and install the package:
     ```bash
     ./scripts/feeds update mypackages
     ./scripts/feeds install -a -p mypackages
     ```
   - Expected output:
     ```
     Installing package 'helloworld' from mypackages
     ```

## Building, Deploying, and Testing the Application

1. Configure the Package:
   - Run `make menuconfig` to select the package:
     ```bash
     make menuconfig
     ```
   - Navigate to `Examples > helloworld` and press `Y` to include it (mark with `*`).
   - Save and exit.

2. Build the Package:
   - Compile the package:
     ```bash
     make package/helloworld/compile
     ```
   - Output: A package file named `helloworld_1.0-1_<arch>.apk` in `bin/packages/<arch>/mypackages/`.
     - Example: `bin/packages/x86_64/mypackages/helloworld_1.0-1_x86_64.apk`.

3. Deploy the Package:
   - Transfer the `.apk` file to an OpenWRT device using `scp`:
     ```bash
     scp bin/packages/x86_64/mypackages/helloworld_1.0-1_x86_64.apk root@192.168.1.1:/tmp/
     ```
   - SSH into the device and install the package:
     ```bash
     ssh root@192.168.1.1
     cd /tmp
     apk add helloworld_1.0-1_x86_64.apk
     ```

4. Test the Application:
   - Run the program on the device:
     ```bash
     helloworld Alice
     ```
   - Expected output:
     ```
     Hello, Alice!
     ```
   - Test error case:
     ```bash
     helloworld
     ```
   - Expected output:
     ```
     Usage: /usr/bin/helloworld <name>
     ```

## Troubleshooting

- SDK Build Fails:
  - Ensure `Build the OpenWrt SDK` is enabled in `make menuconfig`.
  - Verify sufficient disk space and dependencies (see Module 3).
- Package Not Found:
  - Confirm `feeds.conf` points to the correct `mypackages` directory.
  - Re-run `./scripts/feeds update mypackages` and `./scripts/feeds install -a -p mypackages`.
- Build Errors:
  - Check for missing dependencies in the `Makefile` (e.g., `DEPENDS:=+libc`).
  - Ensure the SDK matches the target device’s architecture.
- Missing seperator:
  - The "missing separator" error in a Makefile typically occurs when the commands within a rule are not properly indented with a tab character. Makefiles require that each command line in a rule be preceded by a tab, not spaces.
- Installation Fails:
  - Verify the `.apk` file matches the device’s architecture (`apk install` will show errors if mismatched).
  - Ensure sufficient storage on the device (`df -h`).

## Notes

- Architecture: The SDK is architecture-specific (e.g., x86_64, mips_24kc). Select the correct target in `make menuconfig`.
- Dependencies: Add required libraries to the `Makefile` (e.g., `DEPENDS:=+libubus` for ubus-based programs).
- Custom Feeds: Use a separate directory for `mypackages` to keep the SDK clean.
- Full Build System: For kernel modules or firmware modifications, use the full OpenWRT build system (see Module 3).
- Debugging: Use `logread`, `strace`, or `gdb` on the device for debugging (see Module 8.7).

