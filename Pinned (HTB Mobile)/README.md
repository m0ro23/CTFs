# Pinned - HackTheBox

**Category:** Mobile  
**Difficulty:** Easy

---

## Overview

Got an APK where someone's credentials are hardcoded and the app logs in automatically. The job is to catch that login request in a proxy and pull the password out of it. The app uses certificate pinning though, so just dropping Burp's cert on the device isn't going to cut it.

---

## Setup

The README says API 29 or lower. I used a Pixel 3 emulator running Android 10 via Genymotion.

Install the app:

```bash
adb install pinned.apk
```

---

## First Attempt with Burp

Set Burp to listen on all interfaces, then routed the device traffic through it:

```bash
adb root
adb shell "settings put global http_proxy 192.168.x.x:8080"
```

Opened the app, hit login - Burp's dashboard threw an error saying the connection to `https://pinned.com:443` failed. The app wasn't accepting Burp's certificate at all.

That's certificate pinning. The app has a specific certificate baked in and refuses anything that doesn't match it, so installing a custom CA doesn't help here.

---

## Bypassing the Pinning with Frida

Frida lets you hook into a running process and mess with its behavior at runtime. There's a well-known community script that patches out certificate pinning checks on the fly.

**Get the Frida server for the device:**

```bash
target_arch=$(adb shell 'getprop ro.product.cpu.abi')
# download frida-server for android-$target_arch from github releases
```

**Push it to the device and run it:**

```bash
adb push frida-server /data/local/tmp/frida
adb shell "chmod +x /data/local/tmp/frida"
adb shell "/data/local/tmp/frida" &
```

**Push Burp's certificate in DER format:**

```bash
adb push burp.crt /data/local/tmp/cert-der.crt
```

The bypass script expects the cert at that exact path.

**Find the app's package name:**

```bash
frida-ps -U -ai | grep -i pinned
# com.example.pinned
```

**Run Frida with the pinning bypass:**

```bash
frida -U \
  --codeshare pcipolloni/universal-android-ssl-pinning-bypass-with-frida \
  -f com.example.pinned
```

Went back to the app, hit login, and Burp caught the request this time. The credentials were right there in plaintext.

---

## Flag

```
HTB{trust_n0_1_n0t_3v3n_@_c3rt!}
```

---

## What's Going On

Certificate pinning is a step up from just using HTTPS - the app validates that the certificate it receives matches one it already knows about, so a MITM with a custom CA won't work. The weak point here is that pinning is enforced in code, and Frida can patch that check out at runtime without touching the APK at all. If the app had any anti-tampering or root/debugger detection, this approach would've needed more work.
