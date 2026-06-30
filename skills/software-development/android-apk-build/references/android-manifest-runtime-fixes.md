# Android Manifest Runtime Fixes — FundManager APK Rebuild

Reproduction session: `20260630_000100_447baa`

## Problem Sequence

The FundManager V2 Android app was rebuilt with a new production backend URL (`http://103.94.11.78/api/v1`) but failed at runtime in two sequential errors:

### Error 1: `socket failed: EPERM (operation not permitted)`

**After:** APK installed from `http://103.94.11.78/fundsmanager.apk`  
**Symptom:** App opens → login attempt → socket error immediately

**Root cause:** `AndroidManifest.xml` had no `<uses-permission android:name="android.permission.INTERNET" />`.

**Investigation:** The manifest only had `<application>` tag with no permissions at all. Android requires explicit INTERNET permission for any network access; missing it results in EPERM from the kernel.

**Fix:**
```xml
<manifest ...>
    <uses-permission android:name="android.permission.INTERNET" />
    <uses-permission android:name="android.permission.ACCESS_NETWORK_STATE" />
    <application ...>
```

### Error 2: `Cleartext HTTP traffic to 103.94.11.78 not permitted`

**After:** INTERNET permission was added but cleartext blocking remained.  
**Symptom:** Next login attempt after rebuild → `Cleartext HTTP traffic not permitted by network security policy`

**Root cause:** Android 9+ (API 28) default blocks cleartext HTTP. The app uses `http://` scheme (no HTTPS) and had no `network_security_config.xml`.

**Fix:** Created `res/xml/network_security_config.xml`:
```xml
<?xml version="1.0" encoding="utf-8"?>
<network-security-config>
    <domain-config cleartextTrafficPermitted="true">
        <domain includeSubdomains="false">103.94.11.78</domain>
    </domain-config>
    <base-config cleartextTrafficPermitted="false">
        <trust-anchors>
            <certificates src="system" />
        </trust-anchors>
    </base-config>
</network-security-config>
```

Referenced in manifest:
```xml
<application
    ...
    android:networkSecurityConfig="@xml/network_security_config"
    ...>
```

## Key Observations

| Observation | Detail |
|-------------|--------|
| Error order | EPERM appears FIRST (permission denied before socket opens), cleartext appears SECOND (socket opens but TLS/policy check fails) |
| Both are needed | Adding only INTERNET permission will still fail with cleartext error if using HTTP |
| Scoped is better | `domain-config` scoped to a specific IP/domain is safer than `android:usesCleartextTraffic="true"` |
| Uninstall required | Because the manifest permissions changed, the user had to **uninstall the old app first** before installing the rebuilt APK. Overwriting (without uninstall) may not re-evaluate permissions. |
