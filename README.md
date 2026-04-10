# Strava-no-GMS

Modified version of Strava with GMS dependencies removed — allowing the app to launch and authenticate without Google Mobile Services.

[Читать на русском](README_RU.md)

---

## Why this exists

The official Strava app refuses to launch on devices without Google Play Services. This mod strips out the GMS dependencies required for app startup and login, so you can use Strava on de-Googled devices.

## Compatible devices

- **Huawei / Honor** — devices shipped without Google Services (2019+)
- **Custom ROMs** — LineageOS, /e/OS, GrapheneOS, CalyxOS, and others
- **Any Android device** with GMS disabled or removed

## What's changed

- Removed GMS dependencies required for app launch
- Removed GMS dependencies required for authentication

## Limitations

- ⚠️ **Google Sign-In does not work** — GMS has been intentionally removed
- ✅ **Email login works** — use your Strava email & password

## Installation

1. Download the latest APK from [Releases](../../releases)
2. Enable "Install from unknown sources" in your device settings
3. Install the APK and log in with your email

## FAQ

**Q: Why doesn't Google Sign-In work?**
A: GMS dependencies have been intentionally stripped. Use email login instead.

**Q: How often is this updated?**
A: Updates depend on ReVanced mod releases. A new build is made shortly after a new ReVanced patch becomes available.

**Q: Is this safe?**
A: Only the GMS-related code blocks are modified. The rest of the app is unchanged.

## Disclaimer

This is an unofficial mod and is not affiliated with Strava, Inc. Built on top of ReVanced patches. Use at your own risk.
