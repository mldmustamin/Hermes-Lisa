# APK Version Auto-Increment

User pref: "setiap build, jangan lupa versi apknya juga di perbaharui"

## bump_version.sh
```bash
#!/bin/bash
cd "$(dirname "$0")"
BUILD_FILE="app/build.gradle.kts"
CURRENT_VERSION=$(grep "versionCode" "$BUILD_FILE" | grep -o '[0-9]\+' | head -1)
NEW_VERSION=$((CURRENT_VERSION + 1))
sed -i "s/versionCode = $CURRENT_VERSION/versionCode = $NEW_VERSION/" "$BUILD_FILE"
sed -i "s/versionName = \"[^\"]*\"/versionName = \"2.0.0-b$NEW_VERSION\"/" "$BUILD_FILE"
echo "Version bumped: $CURRENT_VERSION → $NEW_VERSION"
```

## Usage
```bash
bash bump_version.sh && ./gradlew assembleDebug
```

This auto-increments `versionCode` and `versionName` before every build.
Always run before build, commit the version change.
