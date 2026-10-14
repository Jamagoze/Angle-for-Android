# Angle-for-Android

Build [ANGLE](https://chromium.googlesource.com/angle/angle) (OpenGL ES on top of Vulkan) for Android **without a PC**, then install it as a **system-wide Magisk module** so every app launched after boot renders through ANGLE.

<img width="1632" height="1020" alt="1000866288" src="https://github.com/user-attachments/assets/894084cb-fe2a-4a80-8d72-cb8b273a9053" />
YT Link: https://www.youtube.com/watch?v=1FC1FKy6MPM

Made and tested on a **Redmi Pad SE** (Snapdragon 680 / Adreno 610, Android 14, HyperOS, rooted with Magisk).

> ⚠️ **Rooted devices only.** Replacing the system EGL loader can break apps. Read [Uninstall / Recovery](#uninstall--recovery) before flashing.

> 🚫 **Do not use the "Enable ANGLE" toggle in Developer options** (see [the warning below](#-do-not-force-angle-from-developer-options)).

---

## How it works

1. **GitHub Actions** builds ANGLE for `arm64` from the official source (no PC needed, runs from a phone browser).
2. A small **Magisk module** (a ready-made sample is in this repo) puts the four ANGLE libraries in `/vendor/lib64/egl/`.
3. On every boot, `service.sh` sets `persist.graphics.egl=angle`, so Android's EGL loader picks the ANGLE libraries for newly started apps.

The prop is set **after boot completes**, so SurfaceFlinger (the compositor) keeps using the native driver. This makes a bootloop unlikely.

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

- Rooted Android device with Magisk
- Vulkan-capable GPU/driver (Adreno 610 on the test device)
- A GitHub account
- Termux (`pkg install git unzip zip`) for editing and zipping the module

---

## 1. Build ANGLE with GitHub Actions

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

Then:

1. Open the **Actions** tab → **build-angle** → **Run workflow**.
2. Wait about 30-35 minutes.
3. Download **angle-arm64** from the *Artifacts* section of the finished run.

The artifact contains `libEGL_angle.so`, `libGLESv2_angle.so`, `libGLESv1_CM_angle.so` and `libfeature_support_angle.so` (already named correctly, so no renaming is needed).

---

## 2. The sample module

You don't need to write the module yourself. **Copy the sample module that is already in this repo**, then, if you want your own build, replace the four `.so` files with the ones from your Actions artifact.

### Layout

```
angle_sys/
├── module.prop
├── customize.sh
├── service.sh
├── uninstall.sh
└── system/vendor/lib64/egl/
    ├── libEGL_angle.so
    ├── libGLESv2_angle.so
    ├── libGLESv1_CM_angle.so
    └── libfeature_support_angle.so
```

### What each file does

| File | Purpose |
|------|---------|
| `module.prop` | Module id, name and version shown in Magisk. |
| `system/vendor/lib64/egl/*.so` | The four ANGLE libraries. Magisk overlays them onto `/vendor/lib64/egl/`, where Android's EGL loader looks for `libEGL_<name>.so`. Nothing on the real partition is modified. |
| `customize.sh` | Runs at install time and sets permissions (`0755` folders, `0644` files) and the SELinux label `u:object_r:same_process_hal_file:s0`, which the loader needs to be allowed to load them into app processes. |
| `service.sh` | Runs on every boot. It waits until `sys.boot_completed` is `1`, sleeps 5 seconds, then runs `resetprop persist.graphics.egl angle`. Because this happens after boot, the compositor is already running on the native driver and only apps started afterwards use ANGLE. |
| `uninstall.sh` | Runs when you remove the module in Magisk and deletes `persist.graphics.egl`, so Android doesn't keep looking for libraries that no longer exist. |

### Using it

1. Copy the module folder to your device (for example into Termux or `/sdcard/Download`).
2. *(Optional)* Replace the four `.so` files in `system/vendor/lib64/egl/` with the ones from your own build.
3. Zip the **contents** of the folder, so `module.prop` sits at the root of the zip and not inside an extra folder. From Termux:

   ```sh
   cd angle_sys
   zip -r /sdcard/Download/angle_sys.zip . -x '.*'
   ```

### Notes on the sample

- It only contains **64-bit (arm64)** libraries. 32-bit-only apps are not covered, and with the prop set they may fail to start. This was not tested.
- The libraries are tied to the ANGLE source you built. Rebuild and replace them if you want a newer ANGLE.

---

## 3. Install

1. Disable or remove any older ANGLE modules to avoid conflicts.
2. Magisk → **Modules** → **Install from storage** → pick `angle_sys.zip or angle_sys_v2_3.zip`.
3. Reboot.

---

## 4. Verify

```sh
su -c "getprop persist.graphics.egl"
```
Should print `angle`.

Open an app or game (e.g. Minecraft), then check that ANGLE libs are really loaded in its process:

```sh
su -c "cat /proc/\$(pidof com.mojang.minecraftpe)/maps | grep -i angle"
```
You should see `libEGL_angle.so`, `libGLESv2_angle.so` and `libGLESv1_CM_angle.so` from `/vendor/lib64/egl/`.

Optionally confirm the renderer:

```sh
su -c "logcat -d | grep -i 'ANGLE (' | tail -3"
```
Expect something like `ANGLE (Qualcomm, Vulkan ... Adreno ...)`.

---

## Uninstall / Recovery

`persist.*` properties survive reboots. If the libs are removed while the prop still says `angle`, apps can crash or show black screens. The included `uninstall.sh` cleans this up when you remove the module in Magisk.

If something breaks, run this from Termux (root) or adb, then reboot:

```sh
su -c "resetprop -p --delete persist.graphics.egl"
```

If you can't boot at all, boot into Magisk safe mode or delete `/data/adb/modules/angle_sys` from recovery or adb.

---

## Notes

- Affects apps started **after** boot. Skia-on-GL UI rendering goes through ANGLE; **Vulkan-direct** apps and Skia-on-Vulkan (`skiavk`) bypass EGL and are unaffected.
- Some apps misbehave on ANGLE. If one glitches, compare with the module disabled.
- If Magisk says *Unable to extract zip file*, add a `META-INF/com/google/android/` folder with Magisk's standard `update-binary` and `updater-script` to the zip.
- Why not build on-device? The ANGLE/Chromium toolchain only ships for x86_64 Linux, and the checkout is huge, so a cloud build is the practical option on a phone or tablet.

## Credits

- [ANGLE](https://chromium.googlesource.com/angle/angle) by Google and contributors (BSD-3-Clause)
- [Magisk](https://github.com/topjohnwu/Magisk) by topjohnwu

## License

The scripts in this repo are provided as-is, without warranty. ANGLE itself is licensed under its own BSD-3-Clause license.
