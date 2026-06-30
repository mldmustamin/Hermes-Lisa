# AVD Headless Setup for Server Debugging

## When to Use

Need to test Android APK on a Linux server without GUI/desktop. Requires KVM support on the host machine.

## Prerequisites Check

```bash
grep -E "vmx|svm" /proc/cpuinfo  # CPU virtualization support
ls -la /dev/kvm                   # KVM device
free -h                           # At least 4GB RAM available
```

## Setup (15-20 minutes)

### 1. Install Android SDK command-line tools

```bash
export ANDROID_SDK_ROOT=/home/android-sdk
mkdir -p $ANDROID_SDK_ROOT/cmdline-tools
cd /tmp
wget https://dl.google.com/android/repository/commandlinetools-linux-11076708_latest.zip
unzip commandlinetools-linux-*.zip -d cmdline-tmp
mv cmdline-tmp/cmdline-tools $ANDROID_SDK_ROOT/cmdline-tools/latest
export PATH=$ANDROID_SDK_ROOT/cmdline-tools/latest/bin:$PATH
```

### 2. Install emulator + system image

```bash
yes | sdkmanager --licenses
sdkmanager "emulator" "platform-tools" "platforms;android-34"
sdkmanager "system-images;android-34;default;x86_64"
```

### 3. Create AVD

```bash
echo "no" | avdmanager create avd -n "DebugAVD" \
  -k "system-images;android-34;default;x86_64" \
  -d "pixel_6" -f
```

### 4. Start emulator (headless)

```bash
export ANDROID_SDK_ROOT=/home/android-sdk
export PATH=$ANDROID_SDK_ROOT/emulator:$ANDROID_SDK_ROOT/platform-tools:$PATH

emulator -avd DebugAVD \
  -no-window \          # No GUI window
  -no-audio \           # No audio output
  -gpu swiftshader_indirect \  # Software GPU
  -memory 2048 \        # 2GB RAM
  -netdelay none \
  -netspeed full \
  -no-snapshot &        # Fresh boot
```

### 5. Wait for boot

```bash
adb wait-for-device
for i in $(seq 1 30); do
    state=$(adb shell getprop sys.boot_completed 2>/dev/null | tr -d '\r \t')
    [ "$state" = "1" ] && break
    sleep 10
done
```

### 6. Install and launch APK

```bash
adb install -r /path/to/app-debug.apk
adb shell am start -n com.example.fundsmanager/.MainActivity
```

### 7. Capture logs

```bash
# In background:
adb logcat -v time > /tmp/app_logcat.txt &

# Filter specific:
adb logcat -v time | grep -iE "fatal|AndroidRuntime|fundsmanager"
```

## Input Simulation (Headless)

Headless emulators lack IME (keyboard). Use these workarounds:

```bash
# Text input (works without IME)
adb shell input text "username"

# Key events
adb shell input keyevent 66  # ENTER
adb shell input keyevent 61  # TAB

# Tap coordinates (screen-dependent: 1080x2400 for Pixel 6)
adb shell input tap 540 900   # center-top area

# Take screenshot
adb exec-out screencap -p > screen.png

# UI dump to find elements
adb shell uiautomator dump /sdcard/ui.xml
adb pull /sdcard/ui.xml /tmp/ui.xml
grep -o 'text="[^"]*"' /tmp/ui.xml  # Find clickable text
```

## Network

Emulator uses `10.0.2.2` to reach host machine's localhost. For external URLs (e.g., `103.94.11.78`), the emulator uses the host's network directly.

## Troubleshooting

| Issue | Fix |
|-------|-----|
| `/dev/kvm` not found | Load KVM module: `modprobe kvm && modprobe kvm_intel` |
| Emulator EXIT immediately | Check `cat /tmp/emu.log` — usually missing GPU/cpu |
| ADB says "no devices" | `adb kill-server && adb start-server && adb wait-for-device` |
| IME not showing | Use `input keyevent` and `input text` instead of taps |
| APK not installing | Check `adb logcat` for package conflicts |
