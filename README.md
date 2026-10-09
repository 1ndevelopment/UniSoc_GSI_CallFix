# GSI on Unisoc Incoming Call Fix

Magisk module that restores incoming calls on some Unisoc mobiles, running a GSI (Generic System Image). Most mobile operators require VoLTE. The stock firmware's Unisoc IMS stack is deployed to trigger IMS PDN establishment via `urild`'s `vendor.unisoc.hardware.radio.ims.IImsRadio/slot1`.

## Root Cause

- Voice calls must use VoLTE.
- GSI ROMs ship FLOSS IMS, which lacks the Unisoc AIDL stubs needed to call `IImsRadio.setCallback()`. Without this call, urild never establishes the IMS PDN bearer.
- The stock firmware's `com.spreadtrum.ims` APK contains the required stubs but won't run unmodified on a GSI (signature mismatch, missing permissions, SELinux denials).

## What This Module Does

1. **Replaces the IMS service** — Installs the stock Unisoc `com.spreadtrum.ims` APK with all required Unisoc framework JARs merged as multidex (5 DEX files). The `android:sharedUserId` is removed to avoid platform key conflicts with the GSI's TeleService.

2. **Smali-level crash fixes** — Wraps permission-denied and SELinux-blocked operations in try-catch blocks so the app survives on a GSI:
   - `IMS_SERVICE_UP` broadcast → caught, logged
   - `ServiceManager.addService("ims_ex"/"ims_ut_ex")` → caught, logged (app UID not allowed)
   - `SystemProperties.set()` in `setTelephonyProperty` → caught, logged
   - `ImsManager.updateImsServiceConfig()` → caught, logged (missing `WRITE_SECURE_SETTINGS`)
   - `sendImsRegistedBroadcast` (VOLTE_REGISTED) → caught, logged

3. **Installs the IMS bridge daemon** — `ims_bridged`, `libimsbrd.so`, and the init `.rc` file from the stock firmware. The daemon is started at boot via `post-fs-data.sh` (GSI init does not recognize the service).

4. **Adds SELinux rules** (`sepolicy.rule` + live `magiskpolicy --live` in post-fs-data):
   - `priv_app → ext_radio_service:service_manager { find }` — find IImsRadio on vndbinder
   - `priv_app → default_android_service:service_manager { find }` — find SRMI service
   - `priv_app → rild:binder { call transfer }` + `radio` — binder IPC with urild
   - `priv_app → radio_prop:property_service { set }` — set system properties
   - `priv_app → property_socket:sock_file { write }` — write to property socket

5. **Sets SELinux permissive at boot** (if still enforcing after sepolicy.rule load).

6. **Enables VoLTE** — Sets `persist.vendor.sys.volte.enable=true`, IMS APN, LTE preferred network mode (9), and activates the SPRD overlay.

## Files

| Path | Purpose |
|------|---------|
| `module.prop` | Magisk module metadata |
| `customize.sh` | Installer UI and diagnostics |
| `post-fs-data.sh` | Post-boot setup: overlays, properties, ims_bridged, SELinux rules |
| `sepolicy.rule` | SELinux rules merged at boot (short type names — `priv_app`, not `u:r:priv_app:s0`) |
| `system.prop` | Persistent system properties |
| `system/system_ext/priv-app/ims/ims.apk` | Patched stock Unisoc IMS APK |
| `system/system_ext/bin/ims_bridged` | IMS bridge userspace daemon |
| `system/system_ext/lib64/libimsbrd.so` | IMS bridge library (arm64) |
| `system/system_ext/lib/libimsbrd.so` | IMS bridge library (arm) |
| `system/system_ext/etc/init/ims_bridged.rc` | Init service declaration (GSI ignores it; started directly by post-fs-data.sh) |
| `system/system_ext/etc/permissions/com.spreadtrum.ims.xml` | Priv-app permissions grant |

## Installation

1. Install the module ZIP in Magisk Manager or via `magisk --install-module`.
2. Reboot.
3. Verify: `getprop gsm.sys.volte.state` should return `1`.
4. Test an incoming call.

If `gsm.sys.volte.state` is empty after boot, the `ims_bridged` daemon may not have started. Check `ps -A | grep ims_bridge` — if missing, run manually:
```sh
adb shell "su -c 'nohup /system/system_ext/bin/ims_bridged >/dev/null 2>&1 &'"
```
Then kill the IMS process so it reconnects to IImsRadio after SELinux is permissive:
```sh
adb shell "su -c 'kill \$(pidof com.spreadtrum.ims)'"
```

## Verification

```sh
# IMS service bound?
dumpsys phone | grep -A5 'Active controllers:'
# Should show isBound=true

# MMTEL feature ready?
dumpsys phone | grep 'state=READY'

# IMS registration events?
logcat -d | grep -E 'EVENT_IMS_BEARER|notifyImsRegister|volteEnable'

# ims_bridge running?
ps -A | grep ims_bridge
```

## How It Works

The stock Unisoc IMS APK communicates with urild (the vendor RIL) through the AIDL interface `vendor.unisoc.hardware.radio.ims.IImsRadio/slot1` on the vndbinder. When the APK calls `setCallback()` on this interface, urild begins establishing the IMS PDN and SIP registration. The APK internally discovers this service via `ServiceManager.getService("vendor.unisoc.hardware.radio.ims.IImsRadio/slot1")` — no NDK library is needed.

The `ims_bridged` daemon provides the bridge between the kernel IMS driver and userspace, handling packet forwarding for the IMS PDN.

## Compatibility

- **Devices**: FOXXD A67L (Unisoc SC9863A), POCO C71, Redmi A5
- **Modem**: Spreadtrum/Unisoc with urild implementing `vendor.unisoc.hardware.radio.ims.IImsRadio`
- **Carrier**: AT&T (or any carrier requiring VoLTE with CSFB disabled)
- **GSI**: Android 14 PHH-based GSIs (tested on LineageOS GSI, crDroid GSI)
- **Magisk**: 24+
