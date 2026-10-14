# Contributing to Angle-for-Android

Thanks for helping out! This repo builds ANGLE (OpenGL ES on Vulkan) for Android and packages it as a Magisk module.

## Reporting issues

Please include:
- Device model, SoC/GPU, and Android version
- Magisk version and module version
- The app that misbehaves (package name)
- Logs: `logcat` output and `dumpsys gpu` if relevant

## Suggesting changes

1. Fork the repo and create a branch from `main`
2. Make your change and test it on a real device
3. Open a pull request describing what changed and why

## Testing checklist

- Module installs and boots without a bootloop
- `ro.gfx.angle.supported` / `persist.graphics.egl` behave as expected
- Target apps map `libEGL_angle` / `libGLESv2_angle`
- Apps that break on ANGLE are noted in the issue

## Guidelines

- Keep pull requests small and focused
- Don't commit built binaries; the GitHub Actions workflow produces them
- Be respectful and constructive

## License

By contributing, you agree your contributions are licensed under the same license as this project.
