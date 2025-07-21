
# How to Patch an OpenWrt (prplOS) Package

Sometimes, you’ll need to modify an existing package in OpenWrt or prplOS—for example, to fix bugs, change behavior, or add debug prints. The cleanest and most reproducible way to do that is to create a patch and integrate it into the build system.

This guide shows how to properly patch a package the **OpenWrt way**, using quilt-style patches that apply during the build process.

---

## Prerequisites

Make sure you have:
- A working OpenWrt (or prplOS) build system.
- Basic familiarity with Linux commands.
- `quilt` (optional, but recommended).

---

## Step 1: Locate the package

All OpenWrt packages live in:

```sh
./package/<feed>/<pkgname>/
```

For example, the `ubus` package might be at:

```sh
./package/system/ubus/
```

If you're using a custom package (like `dm_first`), the directory might be:

```sh
./package/custom/dm_first/
```

---

## Step 2: Build once to generate source

Build the package to let the OpenWrt build system unpack and prepare the source code:

```sh
make package/<pkgname>/prepare V=s
```

This will extract the source into `build_dir/`.

---

## Step 3: Modify the source

Find the prepared source in:

```sh
./build_dir/target-*/<pkgname>-<version>/
```

Now modify the source files directly here. For example:

```sh
cd build_dir/target-*/ubus-*/src
vim ubus.c
```

Make your changes (e.g., add a debug `printf()` or fix a bug).

---

## Step 4: Create the patch

Return to the package directory:

```sh
cd package/<feed>/<pkgname>/
```

Use `quilt` to create the patch:

```sh
quilt new 001-fix-my-bug.patch
quilt add ../../build_dir/target-*/<pkgname>-*/src/ubus.c
vim ../../build_dir/.../src/ubus.c   # if not already edited
quilt refresh
```

This will generate a patch file at:

```sh
package/<feed>/<pkgname>/patches/001-fix-my-bug.patch
```

If the `patches/` directory doesn’t exist, create it:

```sh
mkdir -p patches
```

If you're not using `quilt`, you can manually generate a patch like this:

```sh
diff -u original_file.c modified_file.c > patches/001-fix-my-bug.patch
```

---

## Step 5: Rebuild with patch

Clean and rebuild the package:

```sh
make package/<pkgname>/clean
make package/<pkgname>/compile V=s
```

The build system will automatically detect and apply the patches in the `patches/` folder.

---

## Step 6: Verify

You can check if the patch was applied by inspecting the compiled binary or printing debug logs. If the patch doesn’t apply cleanly, the build will fail, showing the patch error.

---

## Tips

- Keep patch filenames ordered (`001-`, `002-`, etc.).
- Always use relative paths in patch files.
- Avoid patching inside `build_dir/` directly unless using `quilt`.

---

## Example

Let’s say you want to patch `dm_first` to change a debug print.

**Modify**:
```c
// dm_first.c
printf("DEBUG: Hello Patch\n");
```

**Patch with quilt**:
```sh
cd package/custom/dm_first
quilt new 001-debug-print.patch
quilt add ../../build_dir/target-*/dm_first-*/dm_first.c
quilt refresh
```

**Rebuild**:
```sh
make package/dm_first/clean
make package/dm_first/compile V=s
```

---

## References

- [OpenWrt: Use Patches with Buildsystem](https://openwrt.org/docs/guide-developer/toolchain/use-patches-with-buildsystem)
- [Quilt Tutorial (Debian Wiki)](https://wiki.debian.org/Quilt)

---
