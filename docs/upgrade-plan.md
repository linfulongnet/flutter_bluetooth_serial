# Flutter Bluetooth Serial 升级方案

## 目标

- Dart SDK >= 3.7, Flutter >= 3.28
- Android: compileSdk 36, targetSdk 36, minSdk 21, AGP 8.6.1, Gradle 8.7, JDK 17
- 支持 Android 12+ 新蓝牙权限模型（同时保留旧权限兼容 Android 11-）

## 当前状态 vs 实际目标

### Dart / Flutter

| 项目 | 当前 | 目标 |
|------|------|------|
| SDK 约束 | `>=2.12.0 <3.0.0` | `>=3.7.0 <4.0.0` |
| Flutter 约束 | `>=1.17.0` | `>=3.28.0` |

### Android 构建（插件）

| 项目 | 当前 | 目标 |
|------|------|------|
| AGP | 4.1.0 | 8.6.1 |
| Gradle | 6.7 | 8.7 |
| compileSdk | 30 | 36 |
| minSdk | 19 | 21 |
| targetSdk | 30 | 36 |
| JDK | 8/11 | 17 |
| source/targetCompatibility | Java 8 | Java 17 |
| appcompat | 1.3.0 | 1.7.0 |
| 仓库 | google() + jcenter() | google() + mavenCentral() |
| buildToolsVersion | 30.0.3 | 删除（AGP 自动管理） |
| group/version 声明 | 存在 | 删除 |
| rootProject.allprojects | 存在 | 删除 |
| lintOptions | `disable 'InvalidPackage'` | `lint { disable 'InvalidPackage' }` |
| 命名空间 | manifest package 属性 | build.gradle namespace 块 |

### Android 构建（Example App）

与插件构建配置同步升级，额外改动：

- `app/build.gradle` 改用声明式 `plugins {}` 块（Flutter 3.38 要求）
- `AndroidManifest.xml` 添加 `android:exported="true"`（Android 12+ 要求）
- 移除 `AndroidManifest.xml` 的 `package` 属性（改用 namespace）
- 添加 `ndkVersion "28.2.13676358"`（integration_test 要求）
- JVM heap 从 1536M 提升到 4096M

### AndroidManifest 权限

```xml
<!-- 新增 Android 12+ 权限 -->
<uses-permission android:name="android.permission.BLUETOOTH_SCAN"
    android:usesPermissionFlags="neverForLocation" />
<uses-permission android:name="android.permission.BLUETOOTH_CONNECT" />

<!-- 旧权限不变 -->
<uses-permission android:name="android.permission.BLUETOOTH" />
<uses-permission android:name="android.permission.BLUETOOTH_ADMIN" />
<uses-permission android:name="android.permission.ACCESS_COARSE_LOCATION" />
<uses-permission android:name="android.permission.ACCESS_FINE_LOCATION" />
```

### Java 代码改动

1. **AsyncTask → ExecutorService**（4 处）
   - 使用 `Executors.newCachedThreadPool()` 替代
   - `connect`、`write`（string/bytes）、`onCancel` 中的清理逻辑
   - `onDetachedFromEngine` 中调用 `executorService.shutdown()` 清理线程

2. **getAddress mService 反射删除**
   - 移除通过反射访问 `BluetoothAdapter.mService` 获取地址的代码
   - 保留 `Settings.Secure` 和 `NetworkInterface` 作为兜底方案

3. **isConnected / removeBond 反射** — 已有 try-catch，无需额外修改

### Dart 代码改动

- `BluetoothConnection.dart`: `StreamSink` 从 `extends` 改为 `implements`（Dart 3.x interface class）
- `FlutterBluetoothSerial.dart`: 移除不必要的 `as Uint8List` 强制转换
- `BluetoothConnection.dart`: 移除未使用的 `_id` 字段
- `flutter_bluetooth_serial.dart`: 移除不必要的 `dart:typed_data` import

### Example App 修复

- `LineChart.dart`: `TextTheme.caption` → `TextTheme.bodySmall`（Flutter 3.x 已移除）

## 验证结果

- `flutter pub get` — 通过
- `flutter analyze` — 0 error，0 warning（插件代码）；4 warning（example 预存）
- `flutter build apk --debug` — 通过，生成 `app-debug.apk`
