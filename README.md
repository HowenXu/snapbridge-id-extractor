# snapbridge-id-extractor

自动恢复尼康相机中已存储的 **SnapBridge 设备标识**（DeviceID），使第三方 App 可以无缝切换为 SnapBridge 连接相机，无需在相机和手机上反复删除配对记录。

本算法被 [nsg - 尼康智能 GPS（安卓版）](https://github.com/HowenXu/nsg) 使用：当相机已有 SnapBridge 配对记录时，App 会自动运行该算法破解标识并连接。

配对协议本身来自 [gkoh/furble](https://github.com/gkoh/furble) 的逆向成果。

## 原理

通过逆向 SnapBridge APK（`snapbridge.backend.D6`），其设备标识（8 字节）的生成逻辑为：

```java
// 仅在 SharedPreferences 中没有 "DeviceID" 时执行一次
byte[] id = new byte[8];
new Random(new Date().getTime()).nextBytes(id); // 种子 = 首次运行的毫秒时间戳
```

关键事实：
- 标识**只生成一次**（首次运行/首次配对），并永久保存；
- 相机会在 BLE 广播的厂商数据中（尼康公司 ID `0x0399` 之后）暴露该标识的**前 4 字节**；
- `java.util.Random.nextBytes` 按小端序填充，因此前 4 字节 = 第一个 `nextInt()` 的低 4 字节。

于是可以数学反解：
1. 读取相机广播的 4 字节前缀（LE 小端 uint32）；
2. 读取 SnapBridge 的首次安装时间（`PackageManager.firstInstallTime`）；
3. Java `Random` 的 48 位状态是 LCG：`state = (state * 0x5DEECE66D + 0xB) mod 2^48`，给定第一个 `nextInt()` 的输出（高 32 位），低 16 位未知 → 共 `2^16` 个候选种子；
4. 筛选出落在合理时间窗内的候选（2015 年至今），通常只有 0~3 个；
5. 对每个候选，连接相机写入配对阶段 1（`stage + timestamp + device + nonce`），相机接受（返回 stage 2）即为正确标识。

## 用法

### 作为独立算法

`SnapBridgeIdSolver.kt` 是纯 Kotlin 实现，无 Android 依赖，可直接复制使用：

```kotlin
// 相机广播的 4 字节前缀（小端 uint32）
val advertisedDevice: Long = 0x244B5D44L
// 时间窗：SnapBridge 首次安装时间 .. 当前时间
val candidates = SnapBridgeIdSolver.candidatesFor(advertisedDevice, installTimeMs, System.currentTimeMillis())
for (candidate in candidates) {
    // candidate.deviceIdHex: 完整 16 位十六进制标识，如 "445D4B24981064F7"
    // 用该标识连接相机并写 stage1，相机接受即成功
}
```

### 在安卓 App 中集成

集成方式参见 [nsg 安卓版](https://github.com/HowenXu/nsg) 的 `android/app/src/main/java/info/skyblond/nsp/ble/protocol/SnapBridgeIdSolver.kt`：
1. 扫描相机，从厂商数据（company `0x0399`）读取 4 字节设备 ID 前缀；
2. 通过 `PackageManager.getPackageInfo("com.nikon.snapbridge.cmru", 0).firstInstallTime` 获取 SnapBridge 安装时间；
3. 调用 `candidatesFor` 得到候选；
4. 逐个连接相机写配对 stage1，相机接受则保存为固定标识，之后无需再删除配对记录。

## 测试

`SnapBridgeIdSolverTest.kt` 使用 JUnit4，包含真实验证值：
- `new Random(1785587115763L).nextBytes(8)` 的实测输出；
- 真实配对种子 `1777120641934`（2026-04-25）对应 `445D4B24981064F7`。

## 许可

[AGPL-3.0](LICENSE)
