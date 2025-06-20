
# Cross-Compiling a C Program for OpenWRT

This guide outlines the process of cross-compiling a simple C program for OpenWRT using its toolchain, as described in the [OpenWRT Cross-Compilation Guide](https://openwrt.org/docs/guide-developer/toolchain/crosscompile). The example compiles a basic "hello world" program, transfers it to an OpenWRT device, and runs it.

## Prerequisites

- **OpenWRT Build Environment**: Set up as per [Module 3](#) (e.g., on Ubuntu/Debian).
- **Toolchain**: Available in the OpenWRT source tree after building or configuring the environment.
- **Target Device**: An OpenWRT device accessible via SSH/SCP.
- **Sample C Program**: A simple program to compile (e.g., `test.c`).

## Sample C Program

Create a file named `test.c` with the following content:

```c
#include <stdio.h>
int main()
{
    printf("hello\n");
    return 0;
}
```

## Setting Up the Cross-Compilation Environment

1. **Set Environment Variables**:
   - Add the OpenWRT toolchain to your `PATH`.
   - Set the `STAGING_DIR` to point to the OpenWRT staging directory.
   - Run the following commands (adjust the path to your OpenWRT source directory):
     ```bash
     export STAGING_DIR=${<OpenWrt>}/staging_dir
     export PATH=${PATH}:${STAGING_DIR}/toolchain-x86_64_gcc-14.3.0_musl/bin
     ```

2. **Verify Toolchain**:
   - Ensure the cross-compiler is accessible:
     ```bash
     x86_64-openwrt-linux-musl-gcc --version
     ```
   - This should display the GCC version (e.g., `gcc version 14.3.0`).

## Creating a Makefile

Create a `Makefile` to compile the program:

```make
all: test

test: test.c
	$(CC) -o test test.c
```

- **Explanation**:
  - `CC`: The cross-compiler (set via environment variable or command line).
  - `test`: The target binary built from `test.c`.

## Compiling the Program

Compile the program using the OpenWRT toolchain:

```bash
make CC=x86_64-openwrt-linux-musl-gcc LD=x86_64-openwrt-linux-ld test
```

- **Explanation**:
  - `CC=x86_64-openwrt-linux-musl-gcc`: Specifies the cross-compiler for x86_64 architecture with musl libc.
  - `LD=x86_64-openwrt-linux-ld`: Specifies the linker.
  - `test`: The target to build.
- **Output**: Generates a binary named `test` compatible with OpenWRT’s target architecture.

## Transferring to the OpenWRT Device

Transfer the compiled binary to the OpenWRT device using `scp`:

```bash
scp test root@192.168.1.1:/tmp/
```

- **Explanation**:
  - `root@192.168.1.1`: The OpenWRT device’s IP and user (default: `root`).
  - `/tmp/`: A writable directory on the device (volatile, in RAM).
- **Note**: Ensure SSH is enabled on the device (`dropbear` is included by default). Set a root password if needed:
  ```bash
  ssh root@192.168.1.1
  passwd
  ```

## Running the Program

1. **Connect to the Device**:
   ```bash
   ssh root@192.168.1.1
   ```

2. **Execute the Binary**:
   ```bash
   cd /tmp
   chmod +x test
   ./test
   ```

3. **Expected Output**:
   ```
   hello
   ```

## Troubleshooting

- **Compiler Not Found**:
  - Verify `PATH` includes the toolchain directory.
  - Check if the toolchain was built (`staging_dir/toolchain-x86_64_gcc-14.3.0_musl/bin/` exists).
- **Binary Fails to Run**:
  - Ensure the binary matches the device’s architecture (e.g., x86_64).
  - Check with `file test` to confirm the binary type.
  - Verify dependencies (this example has none, but others may require libraries like `libubus`).
- **SCP/SSH Issues**:
  - Confirm the device’s IP and SSH access.
  - Check firewall settings on the device (`uci show firewall`).

## Notes

- **Architecture**: Adjust the toolchain path (e.g., `toolchain-mips_24kc_gcc-14.3.0_musl`) for different architectures like MIPS or ARM.
- **Dependencies**: For complex programs, include libraries in the `Makefile` (e.g., `-lubus -lubox`).
- **Packaging**: For production, create an OpenWRT package (see Module 8) instead of manual compilation.
- **Debugging**: Use `strace` or `gdb` on the device if the program fails (see Module 8.7).

