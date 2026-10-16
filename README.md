# Angle-for-Android

Build [ANGLE](https://chromium.googlesource.com/angle/angle) (OpenGL ES on top of Vulkan) for Android **without a PC**, then install it as a **system-wide Magisk / KernelSU module** so every app launched after boot renders through ANGLE.

<img width="1632" height="1020" alt="1000866288" src="https://github.com/user-attachments/assets/894084cb-fe2a-4a80-8d72-cb8b273a9053" />
YT Link: https://www.youtube.com/watch?v=1FC1FKy6MPM

Made and tested on a **Redmi Pad SE** (Snapdragon 680 / Adreno 610, Android 14, HyperOS, rooted with Magisk).

- **64-bit (arm64)**
- **32-bit (arm)**

> ⚠️ **Rooted devices only.** Replacing the system EGL loader can break apps. Read [Bootloop protection](#bootloop-protection) and [Uninstall / Recovery](#uninstall--recovery) before flashing.

> 🚫 **Do not use the "Enable ANGLE" toggle in Developer options** (see [the warning below](#-do-not-force-angle-from-developer-options)).

---

## How it works

1. **GitHub Actions** builds ANGLE from the official source (no PC needed, runs from a phone browser). One workflow builds **64-bit**, a second one builds **32-bit**.
2. A small **Magisk / KernelSU module** (a ready-made sample is in this repo) puts the ANGLE libraries in `/vendor/lib64/egl/` (64-bit) and, optionally, `/vendor/lib/egl/` (32-bit).
3. On every boot, `post-fs-data.sh` first clears any saved `persist.graphics.egl` value, so the boot always starts on the native driver. After boot completes, `service.sh` verifies the libs and sets `persist.graphics.egl=angle` **in memory only** (`resetprop -n`), so Android's EGL loader picks the ANGLE libraries for newly started apps. 64-bit apps use the 64-bit libs and 32-bit apps use the 32-bit libs.
4. A **bootloop guard** disables the module automatically if a boot fails (see [Bootloop protection](#bootloop-protection)).

The prop is set **after boot completes**, so SurfaceFlinger (the compositor), the boot animation and zygote keep using the native driver. This makes a bootloop unlikely in the first place.

> ⚠️ **Why the prop must never be saved to disk.** Once `persist.graphics.egl` says `angle`, ANGLE *is* the GLES driver for every process that starts, not an optional layer, and the loader does not fall back to the native Adreno driver if the libs are missing. Zygote, the boot animation and SurfaceFlinger all use GL, so a saved `angle` value with missing libs (module disabled, removed, safe mode) means a black screen or bootloop. That is why this module sets the prop with `resetprop -n` (memory only) and clears any saved value at the start of every boot.

No ANGLE APK is needed.

---

## 🚫 Do not force ANGLE from Developer options

Many YouTube videos claim you can enable ANGLE **without root** by turning on the ANGLE option (or forcing a graphics driver) in Developer options. **Don't do it.**

- That toggle only tells Android to prefer an ANGLE *package* that is already installed. If none is registered, it does nothing.
- In testing on this device, turning it on changed nothing: `dumpsys gpu` still showed `angleLoadingCount=0` and apps stayed on the native driver.
- Mixing it with this module can cause conflicts and makes it harder to tell what is actually in use.

Leave Developer options at their defaults. This module works through the system EGL loader instead, and does not need that setting.

---

## Requirements

- Rooted Android device with Magisk or KernelSU
- Vulkan-capable GPU/driver (Adreno 610 on the test device)
- A GitHub account
- Termux (`pkg install git unzip zip`) for editing and zipping the module

> The module was developed and tested on **Magisk**. KernelSU support relies on the same standard module scripts, but has not been tested on every KernelSU variant. The scripts need a working `resetprop` (see [Things to know](#things-to-know)); without one, ANGLE stays off.

Check whether your device has 32-bit support (skip the 32-bit parts if this prints nothing):

```sh
getprop ro.product.cpu.abilist32
```

---

## 1. Build ANGLE with GitHub Actions

### 64-bit

Create `.github/workflows/angle.yml` in your repo (typing the full path in the filename box creates the folders):

```yaml
name: build-angle
on: workflow_dispatch
jobs:
  build:
    runs-on: ubuntu-24.04
    steps:
      - name: Free disk
        run: sudo rm -rf /usr/share/dotnet /opt/ghc /usr/local/lib/android
      - name: depot_tools
        run: |
          git clone https://chromium.googlesource.com/chromium/tools/depot_tools.git
          echo "$PWD/depot_tools" >> $GITHUB_PATH
      - name: Fetch ANGLE
        run: |
          git clone https://chromium.googlesource.com/angle/angle
          cd angle
          python3 scripts/bootstrap.py
          gclient sync -j8
      - name: Build
        run: |
          cd angle
          gn gen out/a64 --args='target_os="android" target_cpu="arm64" is_debug=false is_component_build=false angle_enable_gl=false angle_enable_vulkan=true android_ndk_api_level=29'
          autoninja -C out/a64 libEGL libGLESv2 libGLESv1_CM libfeature_support
      - uses: actions/upload-artifact@v4
        with:
          name: angle-arm64
          path: |
            angle/out/a64/*.so
```

### 32-bit (optional)

Create a second file, `.github/workflows/angle32.yml`. It is independent of the first one, so you can run them separately:

```yaml
name: build-angle-32
on: workflow_dispatch
jobs:
  build:
    runs-on: ubuntu-24.04
    steps:
      - name: Free disk
        run: sudo rm -rf /usr/share/dotnet /opt/ghc /usr/local/lib/android
      - name: depot_tools
        run: |
          git clone https://chromium.googlesource.com/chromium/tools/depot_tools.git
          echo "$PWD/depot_tools" >> $GITHUB_PATH
      - name: Fetch ANGLE
        run: |
          git clone https://chromium.googlesource.com/angle/angle
          cd angle
          python3 scripts/bootstrap.py
          gclient sync -j8
      - name: Build
        run: |
          cd angle
          gn gen out/a32 --args='target_os="android" target_cpu="arm" is_debug=false is_component_build=false angle_enable_gl=false angle_enable_vulkan=true android_ndk_api_level=29'
          autoninja -C out/a32 libEGL libGLESv2 libGLESv1_CM libfeature_support
      - uses: actions/upload-artifact@v4
        with:
          name: angle-arm32
          path: |
            angle/out/a32/*.so
```

### Running the builds

1. Open the **Actions** tab → **build-angle** (and/or **build-angle-32**) → **Run workflow**.
2. Wait about 30-35 minutes per build.
3. Download **angle-arm64** (and **angle-arm32**) from the *Artifacts* section of each finished run.

Each artifact contains `libEGL_angle.so`, `libGLESv2_angle.so`, `libGLESv1_CM_angle.so` and `libfeature_support_angle.so` (already named correctly, so no renaming is needed).

> Both builds fetch the current ANGLE `main` branch on the day they run. To keep the 64-bit and 32-bit libs on the same version, run the two builds close together. Check any set with `strings libGLESv2_angle.so | grep "git hash"`.

---

## 2. The sample module

You don't need to write the module yourself. **Copy the sample module that is already in this repo**, then, if you want your own build, replace the `.so` files with the ones from your Actions artifacts.

### Layout

```
angle_sys/
├── module.prop
├── customize.sh
├── post-fs-data.sh
├── service.sh
├── uninstall.sh
└── system/vendor/
    ├── lib64/egl/              # 64-bit libs (from angle-arm64)
    │   ├── libEGL_angle.so
    │   ├── libGLESv2_angle.so
    │   ├── libGLESv1_CM_angle.so
    │   └── libfeature_support_angle.so
    └── lib/egl/                # 32-bit libs (from angle-arm32), optional
        ├── libEGL_angle.so
        ├── libGLESv2_angle.so
        ├── libGLESv1_CM_angle.so
        └── libfeature_support_angle.so
```

If your device has no 32-bit support, leave out `lib/egl/`.

### What each file does

| File | Purpose |
|------|---------|
| `module.prop` | Module id, name and version shown in the root manager. |
| `system/vendor/lib64/egl/*.so` | The 64-bit ANGLE libraries. The root manager overlays them onto `/vendor/lib64/egl/`, where Android's EGL loader looks for `libEGL_<name>.so`. Nothing on the real partition is modified. |
| `system/vendor/lib/egl/*.so` | The 32-bit ANGLE libraries, overlaid onto `/vendor/lib/egl/`. Used by 32-bit apps only. |
| `customize.sh` | Runs at install time and sets permissions (`0755` folders, `0644` files) and the SELinux label `u:object_r:same_process_hal_file:s0` on the lib folders, which the loader needs to be allowed to load them into app processes. |
| `post-fs-data.sh` | Runs early on every boot, before zygote starts. It first deletes any saved `persist.graphics.egl` value, so the boot always starts on the native driver. It is also part of the bootloop guard: it adds 1 to a boot counter and disables the module if the previous boot never finished. |
| `service.sh` | Runs on every boot. It finds `resetprop` (Magisk, KernelSU, KernelSU Next or APatch locations, then `PATH`), waits until `sys.boot_completed` is `1`, resets the boot counter after 30 seconds of stability, checks that the ANGLE libs are really present, and only then runs `resetprop -n persist.graphics.egl angle` (memory only, never saved to disk). If `resetprop` is not found, ANGLE stays off. |
| `uninstall.sh` | Runs when you remove the module in the root manager. It deletes any saved `persist.graphics.egl` value and the module's data folder. |

### Using it

1. Copy the module folder to your device (for example into Termux or `/sdcard/Download`).
2. *(Optional)* Replace the `.so` files in `system/vendor/lib64/egl/` and `system/vendor/lib/egl/` with the ones from your own builds.
3. Zip the **contents** of the folder, so `module.prop` sits at the root of the zip and not inside an extra folder. From Termux:

   ```sh
   cd angle_sys
   zip -r /sdcard/Download/angle_sys.zip . -x '.*'
   ```

### Notes on the sample

- The libraries are tied to the ANGLE source you built. Rebuild and replace them if you want a newer ANGLE.
- **Why the safety check matters:** the ANGLE prop applies to every app. If the prop is on but the 32-bit libs are missing, 32-bit apps could fail to start. `service.sh` therefore keeps ANGLE off unless all the required libs are present.

---

## Bootloop protection

The module protects itself so a bad install can't leave you stuck:

1. **Clean start.** `post-fs-data.sh` deletes any saved `persist.graphics.egl` value before zygote starts, so every boot begins on the native driver even if the libs are missing or the module is disabled.
2. **Boot counter.** `post-fs-data.sh` adds 1 to a counter at the start of every boot. `service.sh` resets it to 0 once boot has completed and stayed stable for 30 seconds.
3. **One failed boot disables the module.** If a boot starts while the counter is still above zero, the previous boot never finished. The module then creates its `disable` file (so the root manager skips it) and deletes the ANGLE prop. The next boot runs on the native driver.
4. **Library check.** ANGLE is only switched on if every required `.so` file is present in `/vendor/lib64/egl/` (and `/vendor/lib/egl/` when the device has 32-bit support). If one is missing, the module stays off.
5. **Late, memory-only activation.** The prop is set only after the system has finished booting, and only in memory (`resetprop -n`). The compositor never starts on ANGLE, and a reboot without the module running returns to the native driver.

### Root manager safety nets (Magisk and KernelSU)

The module's own guard works on both Magisk and KernelSU, because it only relies on the standard `post-fs-data.sh` / `service.sh` scripts and the `disable` file. On top of that, each root manager has its own recovery mechanism that runs independently of this module.

| | Magisk | KernelSU |
|---|---|---|
| **Automatic protection** | Detects repeated failed boots and boots into **safe mode**, which skips all modules. | Detects repeated failed boots and skips all modules (safe mode). |
| **Manual safe mode** | Hold **Volume Down** during boot (device dependent). | Press **Volume Down** several times right after the boot logo appears. |
| **Disable one module** | Create `/data/adb/modules/angle_sys/disable` | Create `/data/adb/modules/angle_sys/disable` |
| **Remove one module** | Create `/data/adb/modules/angle_sys/remove`, or delete the folder | Create `/data/adb/modules/angle_sys/remove`, or delete the folder |
| **Manager app** | Modules → toggle off / remove | Modules → toggle off / remove |

After safe mode, modules usually stay disabled until you turn them back on in the manager app. The module's folder is untouched, so you can re-enable it or delete it from there.

### Things to know

- **To re-arm the module** after it disabled itself, switch it back on in the Modules list of your root manager and reboot.
- **False trip:** rebooting within 30 seconds of the UI appearing (or a crash in that window) counts as a failed boot and disables the module. Just re-enable it.
- **Safe mode is the safe outcome:** if Magisk or KernelSU catches a bootloop first, it skips all modules and none of this module's scripts run. ANGLE stays off, because the prop is never saved to disk. Older versions of this module (v2.4 and earlier) saved the prop, so after updating, boot once with the new module so it clears the old value. If you are stuck on an older version, see [Uninstall / Recovery](#uninstall--recovery).
- **Recovery without the manager app:** from adb or a custom recovery, run:
  ```sh
  touch /data/adb/modules/angle_sys/disable
  ```
  then reboot. This works the same on Magisk and KernelSU.
- **`resetprop` on KernelSU:** the scripts use `resetprop` to set the prop in memory and delete the saved one. Magisk always ships it. On other managers `service.sh` looks in `/data/adb/ksu/bin/`, `/data/adb/ap/bin/`, `/data/adb/magisk/` and your `PATH`. Check yours with `su -c "which resetprop"` or `ls /data/adb/ksu/bin`, and add its path to the list at the top of `service.sh` if it is somewhere else. Do **not** use plain `setprop` as a fallback: it saves the value to disk and brings back the black-screen problem. If no `resetprop` is found, ANGLE simply stays off.
- **KernelSU mounting:** without a KernelSU metamodule the module's libs are not mounted into `/vendor`, so the library check fails and ANGLE stays off. That is the safe outcome.
- The guard cannot help with problems that appear after a normal boot (for example one app crashing on ANGLE). See [Uninstall / Recovery](#uninstall--recovery) for that.

---

## 3. Install

1. Disable or remove any older ANGLE modules to avoid conflicts.
2. Magisk or KernelSU → **Modules** → **Install from storage** → pick the module zip (for example `angle_sys_v2_5_Magisk.zip` or `angle_sys_v2_5_KernelSU.zip`).
3. Reboot.

---

## 4. Verify

```sh
su -c "getprop persist.graphics.egl"
```
Should print `angle` after boot. The value lives in memory only, so it is gone after every reboot until `service.sh` runs again.

Open an app or game (e.g. Minecraft), then check that ANGLE libs are really loaded in its process:

```sh
su -c "cat /proc/\$(pidof com.mojang.minecraftpe)/maps | grep -i angle"
```
You should see `libEGL_angle.so`, `libGLESv2_angle.so` and `libGLESv1_CM_angle.so` from `/vendor/lib64/egl/`.

For a **32-bit app**, the same check should list the files from `/vendor/lib/egl/` instead.

Optionally confirm the renderer:

```sh
su -c "logcat -d | grep -i 'ANGLE (' | tail -3"
```
Expect something like `ANGLE (Qualcomm, Vulkan ... Adreno ...)`.

---

## Uninstall / Recovery

If the libs are removed while the prop still says `angle`, apps can crash or show black screens, and because ANGLE is the GLES driver at that point (no fallback to the native driver), the boot itself can fail. This module avoids that by setting the prop in memory only and clearing any saved value at the start of every boot, so a reboot with the module disabled, removed or skipped in safe mode returns to the native driver.

Older versions of this module (v2.4 and earlier) saved the prop with `resetprop persist.graphics.egl angle`, so it can still be on disk on devices that ran them. `uninstall.sh` and the bootloop guard delete it, but if the module was only disabled or skipped, clear it manually.

If something breaks, run this from Termux (root) or adb (or `adb shell` in recovery if you can reach it), then reboot:

```sh
su -c "resetprop -p --delete persist.graphics.egl"
```

If you can't boot at all, boot into safe mode (see [Root manager safety nets](#root-manager-safety-nets-magisk-and-kernelsu)) or delete `/data/adb/modules/angle_sys` from recovery or adb.

---

## Notes

- Affects apps started **after** boot. Skia-on-GL UI rendering goes through ANGLE; **Vulkan-direct** apps and Skia-on-Vulkan (`skiavk`) bypass EGL and are unaffected (for example Mobile Legends renders through Vulkan on the test device).
- ANGLE is all-or-nothing: the prop is global, so there is no per-app switch. If one app misbehaves, compare with the module disabled.
- If your root manager says *Unable to extract zip file*, add a `META-INF/com/google/android/` folder with the standard `update-binary` and `updater-script` to the zip.
- Why not build on-device? The ANGLE/Chromium toolchain only ships for x86_64 Linux, and the checkout is huge, so a cloud build is the practical option on a phone or tablet.

## Credits

- [ANGLE](https://chromium.googlesource.com/angle/angle) by Google and contributors (BSD-3-Clause)
- [Magisk](https://github.com/topjohnwu/Magisk) by topjohnwu
- [KernelSU](https://github.com/tiann/KernelSU) by tiann and contributors

## License

The scripts in this repo are provided as-is, without warranty. ANGLE itself is licensed under its own BSD-3-Clause license.
