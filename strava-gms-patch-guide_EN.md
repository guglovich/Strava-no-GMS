# Guide: Patching Strava APK to Remove Google Play Services (GMS) Dependency

> **Goal:** Make Strava work without GMS — on Huawei devices without Google, phones with microG, custom ROMs.
> **Tested on:** Strava v448.10 (ReVanced mod), Strava v455.11 (premium mod)
> **OS:** Linux (Arch, Ubuntu, etc.)

---

## Key Takeaways from Practice

### Which mod to choose
- **Strava v448.10 ReVanced** — email code login **works** ✅
- **Strava v455.11 premium mod** — email code login **does not work** ❌ (issue in the mod itself, not in GMS patching)
- Always test login **before** patching GMS — if it doesn't work in the mod, GMS patching won't help

### Why ReVanced is better for repatching
- Already rebuilt and signed → apktool runs without issues
- Signature is already non-original → no fear of "breaking" it
- Some GMS dependencies may already be removed

---

## Required Tools

```bash
# Arch Linux
sudo pacman -S android-tools jdk-openjdk
yay -S android-sdk-build-tools apktool

# Ubuntu/Debian
sudo apt install apktool aapt adb openjdk-17-jdk
# apksigner and zipalign — via android-sdk-build-tools or Android Studio
```

Check paths:
```bash
find /opt/android-sdk -name "apksigner" 2>/dev/null
find /opt/android-sdk -name "zipalign" 2>/dev/null
# Usually: /opt/android-sdk/build-tools/37.0.0/apksigner
```

You also need **jadx-gui** for exploration:
- Download: https://github.com/skylot/jadx/releases

---

## Step 1 — Exploration via jadx-gui

Open the APK in jadx-gui. Use **Navigate → Search Text** and search for:

```
isGooglePlayServicesAvailable
```

Make sure the **Code** checkbox is checked. Note which classes contain this call.

**Key classes to patch:**
- `com.strava.SplashActivity` — main launch blocker
- Obfuscated classes like `f8/a`, `T6/a`, `X8/e4`, `K7/h4` — internal checks

**Classes you can skip** (don't block launch):
- `com.facebook.internal.*` — Facebook analytics
- `io.branch.referral.*` — referral system
- `com.google.android.gms.*` — GMS itself (don't touch)
- `com.mapbox.common.location.*` — maps (separate story)
- `com.google.android.recaptcha.*` — captcha (separate story)

---

## Step 2 — Decompilation

```bash
apktool d "filename.apk" -o output_dir
```

If the folder already exists:
```bash
apktool d "filename.apk" -o output_dir -f
```

---

## Step 3 — Find all occurrences in smali

```bash
grep -r "isGooglePlayServicesAvailable" output_dir/smali* -l
```

You'll get a list of files. For each relevant file:

```bash
grep -n "isGooglePlayServicesAvailable" output_dir/smali_classesX/path/to/file.smali
```

Note the line numbers.

---

## Step 4 — View context

For each occurrence, view the context (±7 lines):

```bash
sed -n 'LINE_NUM_MINUS_7,LINE_NUM_PLUS_7p' path/to/file.smali
```

**What to look for:** the `move-result vX` line after the call. The `vX` register is where the GMS check result lands. There may be `.line N` directives between the call and `move-result` — that's normal, don't touch them.

**Two types of conditions after move-result:**
- `if-eqz vX, :cond_N` — if 0, all good (standard)
- `if-nez vX, :cond_N` — if non-zero, all good (reversed logic)

In both cases the patch is the same — forcibly write 0 into the register.

---

## Step 5 — Find the exact move-result line number

```bash
grep -n "move-result vX" path/to/file.smali
```

Where `vX` is the register from the call line (v0, v1, v2, v4, etc.). Pick the line that comes right after the relevant call by line number.

---

## Step 6 — Insert the patch

Insert `const/4 vX, 0x0` **after** the `move-result` line:

```bash
sed -i 'MOVE_RESULT_LINEa\    const/4 vX, 0x0' path/to/file.smali
```

> ⚠️ Must be exactly 4 spaces before `const/4` — smali uses spaces for indentation.

**Verify the result:**
```bash
sed -n 'LINE_MINUS_3,LINE_PLUS_5p' path/to/file.smali
```

**Correct result should look like:**
```smali
    invoke-virtual {vA, vB}, Lcom/google/android/gms/common/GoogleApiAvailability;->isGooglePlayServicesAvailable(...)I
    .line 103          ← optional, may not be present
    move-result v2
    const/4 v2, 0x0   ← our patch
    if-eqz v2, :cond_4
```

**Common mistake** — `const/4` inserted BEFORE `move-result`. Delete and fix:
```bash
sed -i 'WRONG_LINE_NUMBERd' path/to/file.smali
```

Then repeat grep for the current line number (after each insert/delete, line numbers shift!).

---

## Step 7 — Repeat for all files

Patch all files from the list (excluding the skips from Step 1). For each:
1. `grep -n "isGooglePlayServicesAvailable"` → call line number
2. `grep -n "move-result vX"` → move-result line number
3. `sed -i 'NUMa\    const/4 vX, 0x0'` → insert
4. Verify with `sed -n`

---

## Step 8 — Build

```bash
apktool b output_dir -o patched.apk
```

Should complete without errors:
```
I: Building resources with aapt2...
I: Building apk file...
I: Built apk into: patched.apk
```

---

## Step 9 — Create signing key (one time only)

```bash
keytool -genkey -v -keystore ~/my.keystore -alias mykey \
  -keyalg RSA -keysize 2048 -validity 10000 -noprompt \
  -dname "CN=My, OU=My, O=My, L=My, S=My, C=US" \
  -storepass password -keypass password
```

Save `~/my.keystore` — you'll need it for all future patches. If you lose it, you'll have to reinstall all patched APKs.

---

## Step 10 — Sign and align

**Strictly in this order: zipalign first, then apksigner.**

```bash
# 1. Align
/opt/android-sdk/build-tools/37.0.0/zipalign -v 4 patched.apk aligned.apk

# 2. Sign (v2 scheme)
/opt/android-sdk/build-tools/37.0.0/apksigner sign \
  --ks ~/my.keystore \
  --ks-key-alias mykey \
  --ks-pass pass:password \
  --key-pass pass:password \
  --out final.apk aligned.apk
```

> ⚠️ Don't use `jarsigner` — it only creates a v1 signature. Android 7+ requires v2. The APK will fail to install.

> ⚠️ If you previously used jarsigner — remove the old signature before apksigner:
> ```bash
> zip -d patched.apk "META-INF/*"
> ```

---

## Step 11 — Install

```bash
adb install final.apk
```

If you get a signature conflict error — uninstall the old app version completely:
```bash
adb shell pm uninstall com.strava
adb install final.apk
```

---

## Troubleshooting Common Errors

| Error | Cause | Fix |
|-------|-------|-----|
| `INSTALL_PARSE_FAILED_NO_CERTIFICATES` | Used jarsigner instead of apksigner | Remove META-INF, rebuild with apksigner |
| `INSTALL_FAILED_UPDATE_INCOMPATIBLE` | Signature conflict | Uninstall the old app |
| `There was a problem parsing the package` | APK corrupted during build | Rebuild, check apktool errors |
| `const/4` inserted before `move-result` | Wrong line number in sed | Delete with `sed -i 'Nd'`, repeat grep |
| Auth code "expired" immediately | Issue with the mod itself, not GMS | Use a different mod version |

---

## Testing login without VPN (for Russia)

Strava is geo-blocked via CloudFront (`403 ERROR - configured to block access from your country`). You need a full tunnel (WireGuard, VLESS) — SNI split-routing won't help since CF sees the real IP.

When authorizing via WireGuard on a VPS — make sure the VPS is not in a datacenter on Strava's blocklist.

---

## Quick Checklist

- [ ] Test login in the mod BEFORE patching
- [ ] Decompile with apktool
- [ ] grep across all smali* folders
- [ ] Exclude facebook/branch/gms/mapbox/recaptcha
- [ ] For each file: find move-result, insert const/4, verify
- [ ] Build with `apktool b`
- [ ] zipalign → apksigner (in this exact order)
- [ ] adb install
- [ ] Uninstall old version if signature conflict
