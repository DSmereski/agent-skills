---
name: deploy-android-apps
description: Build the latest release APK of every Flutter Android app in a workspace directory and install them onto a connected Android device via adb. Use when you want to refresh your whole Flutter Android fleet in one shot — discovers apps automatically, builds each one, and side-loads via adb install.
---

> Public trimmed version shared from smereski.com.
> Adapt `<your projects dir>`, device serial, and tool paths to your own environment.

# Deploy Android Apps — build every Flutter app → install to device

Discover every Flutter app under a workspace directory, build a release APK for
each, and `adb install -r` it to a connected Android device. One command
refreshes your whole fleet.

---

## Requirements

| Tool | Notes |
|------|-------|
| `flutter` | In PATH or set `FLUTTER` env var to the full binary path |
| `adb` | Android SDK platform-tools; add to PATH or set `ADB` env var |
| Android device | USB-connected or on the same network (TCP/IP adb); confirm with `adb devices -l` |

### Add adb to PATH (example — Git Bash on Windows)

```bash
export PATH="$PATH:/c/Users/<you>/AppData/Local/Android/Sdk/platform-tools"
```

---

## Quick start

```bash
# Build + install every Flutter app found in the workspace
bash deploy.sh

# Build + install a single app (pass its directory name)
bash deploy.sh MyAppDir
```

Each release build takes roughly 1–4 minutes. Run in the background and tail
the log if you don't want to wait:

```bash
bash deploy.sh > /tmp/deploy-android.log 2>&1 &
tail -f /tmp/deploy-android.log
```

---

## Reference script — `deploy.sh`

```bash
#!/usr/bin/env bash
# deploy.sh — build release APK for every Flutter app in WORKSPACE and install
# to the first connected Android device (or the one set in SERIAL=).
#
# Usage:
#   bash deploy.sh              # all apps
#   bash deploy.sh MyAppDir     # one app (directory name under WORKSPACE)
#
# Env overrides:
#   WORKSPACE   root directory that contains your Flutter projects  (default: ~/Projects)
#   ADB         path to adb binary                                  (default: adb)
#   FLUTTER     path to flutter binary                              (default: flutter)
#   SERIAL      force a specific device serial                      (default: first device)

set -euo pipefail

WORKSPACE="${WORKSPACE:-$HOME/Projects}"
ADB="${ADB:-adb}"
FLUTTER="${FLUTTER:-flutter}"
LOG="/tmp/deploy-android-apps.log"
TARGET="${1:-}"          # optional: single app dir name

# ── resolve device serial ──────────────────────────────────────────────────────
if [[ -n "${SERIAL:-}" ]]; then
  DEVICE="$SERIAL"
else
  DEVICE=$("$ADB" devices | awk '/\tdevice$/{print $1; exit}')
  if [[ -z "$DEVICE" ]]; then
    echo "ERROR: no device in 'device' state. Run 'adb devices -l' to diagnose." >&2
    exit 1
  fi
fi
echo "Using device: $DEVICE"

# ── discover Flutter apps ──────────────────────────────────────────────────────
if [[ -n "$TARGET" ]]; then
  APP_DIRS=("$WORKSPACE/$TARGET")
else
  mapfile -t APP_DIRS < <(find "$WORKSPACE" -maxdepth 2 -name pubspec.yaml \
    -exec dirname {} \;)
fi

# ── build + install loop ───────────────────────────────────────────────────────
for DIR in "${APP_DIRS[@]}"; do
  [[ -f "$DIR/pubspec.yaml" ]] || { echo "SKIP $DIR (no pubspec.yaml)"; continue; }

  echo "──────────────────────────────────────────"
  echo "APP: $DIR"

  # Detect Flutter entrypoint; skip native-only scaffolds
  if [[ ! -f "$DIR/lib/main.dart" ]]; then
    echo "SKIP $DIR — no lib/main.dart (native-only or incomplete project)"
    continue
  fi

  # Regenerate missing android/ host scaffold (see Gotchas section)
  BUILD_GRADLE="$DIR/android/app/build.gradle"
  BUILD_GRADLE_KTS="$DIR/android/app/build.gradle.kts"
  if [[ ! -f "$BUILD_GRADLE" && ! -f "$BUILD_GRADLE_KTS" ]]; then
    echo "  → android/ host missing; regenerating scaffold..."
    PUBSPEC_NAME=$(grep '^name:' "$DIR/pubspec.yaml" | awk '{print $2}')
    (cd "$DIR" && "$FLUTTER" create -t app \
      --platforms=android \
      --project-name "$PUBSPEC_NAME" \
      --org com.example \
      .) | tee -a "$LOG"
  fi

  # Build release APK
  echo "  → building release APK..."
  (cd "$DIR" && "$FLUTTER" build apk --release 2>&1) | tee -a "$LOG" || {
    echo "  BUILD FAILED for $DIR — see $LOG"
    continue
  }

  # Locate the built APK
  APK=$(find "$DIR/build/app/outputs/flutter-apk" \
    -name "*.apk" -not -name "*unsigned*" 2>/dev/null | head -1)
  [[ -z "$APK" ]] && APK=$(find "$DIR/build/app/outputs/apk/release" \
    -name "*.apk" 2>/dev/null | head -1)

  if [[ -z "$APK" ]]; then
    echo "  APK not found after build — skipping install"
    continue
  fi

  # Install
  echo "  → installing $APK..."
  "$ADB" -s "$DEVICE" install -r "$APK" 2>&1 | tee -a "$LOG" \
    && echo "  ✓ installed" \
    || echo "  install failed — see $LOG"
done

echo "Done. Full log: $LOG"
```

---

## Gotchas

### Flutter apps without an android/ host scaffold

Apps scaffolded by code generators or AI assistants often ship with only
`lib/` and `pubspec.yaml` — no `android/` directory. Building them fails
immediately with errors like *"deleted Android v1 embedding"* or
*"unsupported Gradle project"*.

**Fix** — regenerate the host scaffold without touching your Dart code:

```bash
cd <project-root>
flutter create -t app \
  --platforms=android \
  --project-name <name-from-pubspec> \
  --org com.example \
  .
```

The `deploy.sh` above does this automatically when
`android/app/build.gradle[.kts]` is missing.

### No `lib/main.dart` → not a Flutter app

A `pubspec.yaml` alone does not guarantee a Flutter entrypoint. Native Android
projects or incomplete generated scaffolds sometimes carry a `pubspec.yaml`
but have no Dart source. The script detects this and skips those directories.

### Release APKs are debug-signed by default

`flutter build apk --release` uses a debug keystore unless you configure a
release keystore in `android/key.properties`. This is fine for sideloading
via `adb install`; it is not suitable for Google Play submission.

### Multiple devices attached

`adb` errors when more than one device is connected and no serial is specified.
Set the `SERIAL` env var or pass `-s <serial>` explicitly. List attached
devices with:

```bash
adb devices -l
```

### `adb install -r` keeps app data

Reinstalling preserves the app's data directory, but major version jumps can
reset internal state. Re-login or re-pair inside the app if needed after a
major upgrade.

---

## Verify installs

```bash
# List packages matching your app identifier
adb -s <device-serial> shell pm list packages | grep <your-identifier>

# Launch an app from the command line
adb -s <device-serial> shell monkey -p <package.name> \
  -c android.intent.category.LAUNCHER 1
```

---

## Related skills

- `android-emulation` — drive and observe an installed app via adb (screencap
  + input injection) without touching a physical device. Useful for verifying a
  deployed app actually runs.
