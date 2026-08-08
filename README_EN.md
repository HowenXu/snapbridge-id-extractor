# snapbridge-id-extractor

Recovers the **SnapBridge DeviceID** a Nikon camera has stored, so third-party apps can connect to the camera as SnapBridge itself — no more deleting pairing records on both sides every time you switch apps.

This algorithm is used by [nsg - Nikon Smart GPS (Android)](https://github.com/HowenXu/nsg): when a camera already has a SnapBridge pairing, the app runs this algorithm automatically and connects.

The pairing protocol itself comes from the reverse engineering work of [gkoh/furble](https://github.com/gkoh/furble).

## How it works

By reverse-engineering the SnapBridge APK (`snapbridge.backend.D6`), the 8-byte device identity is generated as:

```java
// runs only once, when "DeviceID" is not yet in SharedPreferences
byte[] id = new byte[8];
new Random(new Date().getTime()).nextBytes(id); // seed = first-run millisecond timestamp
```

Key facts:
- The identity is generated **exactly once** (first run / first pairing) and stored forever;
- The camera advertises the **first 4 bytes** of that identity in its BLE manufacturer data (after Nikon company id `0x0399`);
- `java.util.Random.nextBytes` fills little-endian, so the first 4 bytes equal the low 4 bytes of the first `nextInt()`.

So we can invert it mathematically:
1. Read the 4-byte prefix (LE uint32) from the camera advertisement;
2. Read SnapBridge's first-install time (`PackageManager.firstInstallTime`);
3. Java `Random`'s 48-bit state is an LCG: `state = (state * 0x5DEECE66D + 0xB) mod 2^48`. Given the first `nextInt()` output (high 32 bits), the low 16 bits are unknown → `2^16` candidate seeds;
4. Filter candidates to a plausible time window (2015-present), usually leaving 0-3;
5. For each candidate, connect to the camera and write pairing stage 1 (`stage + timestamp + device + nonce`); the camera accepting it (returning stage 2) confirms the identity.

## Usage

### Standalone algorithm

`SnapBridgeIdSolver.kt` is pure Kotlin with no Android dependencies:

```kotlin
// 4-byte prefix advertised by the camera (LE uint32)
val advertisedDevice: Long = 0x244B5D44L
// window: SnapBridge first-install time .. now
val candidates = SnapBridgeIdSolver.candidatesFor(advertisedDevice, installTimeMs, System.currentTimeMillis())
for (candidate in candidates) {
    // candidate.deviceIdHex: full 16-hex identity, e.g. "445D4B24981064F7"
    // connect with it and write stage1; the camera accepting means success
}
```

### Integration in an Android app

See [nsg (Android)](https://github.com/HowenXu/nsg), `android/app/src/main/java/info/skyblond/nsp/ble/protocol/SnapBridgeIdSolver.kt`:
1. Scan for the camera and read the 4-byte device ID prefix from manufacturer data (company `0x0399`);
2. Get SnapBridge's install time via `PackageManager.getPackageInfo("com.nikon.snapbridge.cmru", 0).firstInstallTime`;
3. Call `candidatesFor` to get the candidates;
4. Connect with each candidate and write pairing stage 1; when the camera accepts, save it as the fixed identity.

## Tests

`SnapBridgeIdSolverTest.kt` (JUnit4) includes verified values:
- Real output of `new Random(1785587115763L).nextBytes(8)`;
- Real pairing seed `1777120641934` (2026-04-25) → `445D4B24981064F7`.

## License

[AGPL-3.0](LICENSE)
