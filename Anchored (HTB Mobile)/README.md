# Anchored - HackTheBox

**Category:** Mobile  
**Difficulty:** Easy

---

## Overview

The challenge gives you an Android APK. The task is simple - intercept the HTTPS traffic the app sends and grab a secret value that gets transmitted with the user's email. The catch is the app is meant to run on a non-rooted device.

---

## Getting Started

Two constraints in the challenge README:

- Run it on API 29 or lower
- No rooted device

I spun up a Pixel 4 emulator at API 25. The app loaded fine and showed a basic screen with an email input field - nothing fancy.

---

## Poking Around the APK

Threw the APK into `jadx` to see what's inside. Found a login endpoint hardcoded in `MainActivity`:

```
https://anchored.com/api/login
```

Tried pointing Burp at the device and installing the CA cert, but nothing showed up in the proxy. The app was just ignoring it.

Went back to the decompiled code and looked at `AndroidManifest.xml` more carefully. There was this line:

```xml
android:networkSecurityConfig="@xml/network_security_config"
```

That's the app telling Android to use a custom trust configuration. Opened the file at `res/xml/network_security_config.xml`:

```xml
<?xml version="1.0" encoding="utf-8"?>
<network-security-config>
    <base-config>
        <trust-anchors>
            <certificates src="system"/>
        </trust-anchors>
    </base-config>
</network-security-config>
```

Only system certs are trusted. User-installed certs - including anything you drop in via Burp - get silently rejected. That's why the proxy wasn't seeing anything.

---

## Patching the APK

The fix is straightforward. Since we can repack and re-sign the APK, we just modify that one XML file and rebuild.

**Decode:**

```bash
apktool d Anchored.apk
```

**Edit `res/xml/network_security_config.xml`** to add user cert trust:

```xml
<?xml version="1.0" encoding="utf-8"?>
<network-security-config>
    <base-config>
        <trust-anchors>
            <certificates src="system"/>
            <certificates src="user"/>
        </trust-anchors>
    </base-config>
</network-security-config>
```

**Rebuild:**

```bash
apktool b Anchored -o anchored_patched.apk
```

**Create a keystore and sign:**

```bash
keytool -genkey -keystore test.keystore -keyalg RSA -keysize 2048 -validity 10000
apksigner sign --ks test.keystore anchored_patched.apk
```

> If you're on API below 30, make sure the Burp cert file has a `.crt` extension when you install it.

**Install:**

```bash
adb install anchored_patched.apk
```

---

## Getting the Flag

Opened the patched app, entered an email, hit submit. Burp caught the request this time:

```json
{
    "email": "user@anchored.com",
    "secret": "HTB{UnTrUst3d_C3rT1f1C4T3s}"
}
```

---

## Flag

```
HTB{UnTrUst3d_C3rT1f1C4T3s}
```

---

## What Made This Work

The app had no certificate pinning. It was relying entirely on `network_security_config.xml` to control what certificates get trusted. Since APKs can be unpacked and edited freely, changing that config and re-signing the APK bypassed the restriction without touching anything else.

If the app had implemented cert pinning on top of this, patching the XML alone wouldn't have been enough.
