# Mobile app protections (Flutter; notes apply to native too)

## Certificate pinning (pin the CA roots, not the leaf)

Pinning the leaf certificate breaks every renewal (Let's Encrypt renews every ~60–90 days). Pin
the **root CAs** instead: Let's Encrypt ISRG Root X1 (to 2035) and X2 (to 2040).

```dart
SecurityContext pinnedContext() => SecurityContext(withTrustedRoots: false)
  ..setTrustedCertificatesBytes(pinnedRootsPem.codeUnits); // ISRG X1 + X2 PEMs bundled in code

Dio pinned(Dio dio) {
  dio.httpClientAdapter = IOHttpClientAdapter(createHttpClient: () {
    final c = HttpClient(context: pinnedContext());
    c.badCertificateCallback = (_, __, ___) => false; // never accept anything else
    return c;
  });
  return dio;
}
```
- Pin only the API client, only for https (skip local dev servers in debug).
- **Before moving the API to another CA** (e.g. Cloudflare proxy), add that root and ship the app
  first, or old apps can't connect.
- Test with Charles/Burp + its CA installed on the phone: requests must fail.

## Android network security config

`android/app/src/main/res/xml/network_security_config.xml`, referenced from the manifest
(`android:networkSecurityConfig="@xml/network_security_config"`):
```xml
<network-security-config>
  <base-config cleartextTrafficPermitted="false">
    <trust-anchors><certificates src="system" /></trust-anchors>
  </base-config>
</network-security-config>
```
No user-installed CAs, no plain HTTP.

## RASP: freeRASP (talsec)

- Subscribe to **all** callbacks (23 in v8): root/jailbreak, hooks, malware, debugger, simulator,
  app integrity (repackaging), multi-instance, automation, unofficial store, device binding,
  device id, VPN, dev mode, ADB, no passcode, no secure hardware, obfuscation, screenshot, screen
  recording, unsecure Wi-Fi, time spoofing, location spoofing, bootloader.
- Config needs the **signing certificate SHA-256 (Base64)** — use the Play **app signing** key
  from Play Console, not the upload key, or every store install looks repackaged. iOS needs team
  id + bundle id.
- Send events to the server (`POST /security/events {events[], platform}`, anonymous allowed,
  rate-limited, enum-validated). The server only **logs**; never trust the client to decide.
- Act in the app only on strong signals (hooks, debugger, repackaged, root in release): show a
  block screen / sign out. Treat VPN, dev mode, no passcode etc. as "watch" only.

## Native anti-tamper (optional, stronger)

Small C code via method channel, compiled with `-O3 -fvisibility=hidden`:
ptrace / `P_TRACED` / `TracerPid` (debugger), scan `/proc/self/maps` and `_dyld_get_image_name`
for frida / substrate / xposed / cycript, root/jailbreak files. Re-run every few seconds and on
app resume (injectors attach after launch). Treat channel errors as clean (never lock out
because of a flaky check). Disable under `FLUTTER_TEST` (periodic timers break widget tests).

## Screenshot / recording block and app-switcher blur

- Android: `FLAG_SECURE` on the activity window.
- iOS: put the Flutter view inside a `UITextField` with `isSecureTextEntry = true` (the secure
  layer trick); on **iOS 17+** the secure sublayer lookup changed — find the `_UITextLayoutCanvasView`
  / first sublayer dynamically instead of a fixed index, and test on 17 and 18.
- Blur/cover the window on `applicationWillResignActive` and remove it on `didBecomeActive`.

## Storage and build

- Tokens and keys in Keychain/Keystore (`flutter_secure_storage`), never SharedPreferences.
- Release: `flutter build --obfuscate --split-debug-info=build/symbols`.
- Keep Firebase config files and keystores out of git.

## Testing tips

Anything with timers or platform channels must be off under `FLUTTER_TEST`, or widget tests fail
with "Timer is still pending". Check `Platform.environment.containsKey('FLUTTER_TEST')`.
