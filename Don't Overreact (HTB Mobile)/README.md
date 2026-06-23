# Don't Overreact - HackTheBox

**Category:** Mobile  
**Difficulty:** Easy

---

## Overview

Got a zip file containing an Android APK. No network traffic to intercept, no pinning to bypass - this one is purely about poking around the app's contents and finding something hidden inside.

---

## Unpacking

Unzipped the archive, then decoded the APK with apktool:

```bash
apktool d app-release.apk
```

This gives you the full directory tree - smali code, resources, manifest, assets, everything.

---

## Looking Around

Browsed through the output folder and went into the `assets` directory. Found a file in there that didn't look like a typical resource. Ran `file` on it to check what it actually was, then started looking through its contents.

Tried grepping for the flag prefix `HTB{` directly but got nothing back. While going through the strings though, one line stood out immediately:

```
SFRCezIzbTQxbl9jNDFtXzRuZF9kMG43XzB2MzIyMzRjN30=
```

That's base64. Decoded it:

```bash
echo "SFRCezIzbTQxbl9jNDFtXzRuZF9kMG43XzB2MzIyMzRjN30=" | base64 -d
```

---

## Flag

```
HTB{23m41n_c41m_4nd_d0n7_0v32234c7}
```

---

## Takeaway

React Native bundles its JavaScript into a single file inside the `assets` folder. Developers sometimes leave sensitive strings in the JS bundle thinking they're obscured, but base64 is trivially reversible - it's encoding, not encryption. Anything hardcoded in the bundle is fair game for static analysis.
