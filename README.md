# Angle-for-Android

Some Android ROMs ship with an ANGLE (OpenGL ES on top of Vulkan) driver, and many don't. This project is for the ROMs that **don't**. It builds ANGLE for Android **without a PC**, then installs it as a **system-wide Magisk / KernelSU module** so every app launched after boot renders through ANGLE.

> **Does your ROM already have ANGLE?** Check first, in Termux with root:
> ```sh
> find /system /vendor /system_ext /product -iname "*angle*" 2>/dev/null
> getprop ro.gfx.angle.supported
> pm list packages | grep -i angle
> ```
> If these find `libEGL_angle.so` / `libGLESv2_angle.so`, report `ro.gfx.angle.supported` as `true`, or list an ANGLE package such as `com.android.angle`, your ROM already includes ANGLE and you don't need this module. If all three come back empty, this module is for you.
