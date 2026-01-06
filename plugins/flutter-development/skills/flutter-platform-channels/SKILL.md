---
name: flutter-platform-channels
description: Implement native iOS/Android integrations with Flutter platform channels, method channels, and event channels. Use when building native plugins, accessing platform APIs, or creating bidirectional native communication.
---

# Flutter Platform Channels

Production-ready patterns for native iOS and Android integrations using method channels, event channels, and basic message channels.

## When to Use This Skill

- Integrating native iOS/Android APIs not available in Flutter
- Creating custom Flutter plugins for native functionality
- Implementing bidirectional communication between Flutter and native code
- Streaming data from native platform to Flutter (sensors, location)
- Accessing platform-specific features (biometrics, NFC, Bluetooth)
- Building native UI components within Flutter

## Core Concepts

### 1. Channel Types

| Channel Type | Direction | Use Case |
|-------------|-----------|----------|
| **MethodChannel** | Bidirectional | Request-response calls (get data, execute action) |
| **EventChannel** | Native → Flutter | Streaming data (sensors, location updates) |
| **BasicMessageChannel** | Bidirectional | Simple data exchange with codecs |

### 2. Communication Flow

```
┌─────────────────────────────────────┐
│           Flutter (Dart)            │
│   MethodChannel / EventChannel      │
└──────────────────┬──────────────────┘
                   │ Platform Message
┌──────────────────▼──────────────────┐
│        Flutter Engine               │
│      (Message Serialization)        │
└──────────────────┬──────────────────┘
                   │ Binary Message
┌─────────┬────────▼────────┬─────────┐
│ iOS     │                 │ Android │
│ (Swift) │                 │(Kotlin) │
└─────────┴─────────────────┴─────────┘
```

## Quick Start

```yaml
# pubspec.yaml
flutter:
  plugin:
    platforms:
      android:
        package: com.example.my_plugin
        pluginClass: MyPlugin
      ios:
        pluginClass: MyPlugin
```

## Patterns

### Pattern 1: MethodChannel - Basic Communication

```dart
// lib/src/battery_channel.dart
import 'package:flutter/services.dart';

class BatteryChannel {
  static const MethodChannel _channel = MethodChannel('com.example.app/battery');

  /// Get current battery level (0-100)
  static Future<int> getBatteryLevel() async {
    try {
      final int level = await _channel.invokeMethod('getBatteryLevel');
      return level;
    } on PlatformException catch (e) {
      throw BatteryException('Failed to get battery level: ${e.message}');
    }
  }

  /// Check if device is charging
  static Future<bool> isCharging() async {
    try {
      final bool charging = await _channel.invokeMethod('isCharging');
      return charging;
    } on PlatformException catch (e) {
      throw BatteryException('Failed to get charging status: ${e.message}');
    }
  }

  /// Get battery health info (Android only)
  static Future<Map<String, dynamic>> getBatteryInfo() async {
    try {
      final result = await _channel.invokeMethod('getBatteryInfo');
      return Map<String, dynamic>.from(result);
    } on PlatformException catch (e) {
      throw BatteryException('Failed to get battery info: ${e.message}');
    }
  }
}

class BatteryException implements Exception {
  final String message;
  BatteryException(this.message);

  @override
  String toString() => 'BatteryException: $message';
}
```

```swift
// ios/Classes/BatteryPlugin.swift
import Flutter
import UIKit

public class BatteryPlugin: NSObject, FlutterPlugin {
    public static func register(with registrar: FlutterPluginRegistrar) {
        let channel = FlutterMethodChannel(
            name: "com.example.app/battery",
            binaryMessenger: registrar.messenger()
        )
        let instance = BatteryPlugin()
        registrar.addMethodCallDelegate(instance, channel: channel)
    }

    public func handle(_ call: FlutterMethodCall, result: @escaping FlutterResult) {
        switch call.method {
        case "getBatteryLevel":
            handleGetBatteryLevel(result: result)
        case "isCharging":
            handleIsCharging(result: result)
        case "getBatteryInfo":
            handleGetBatteryInfo(result: result)
        default:
            result(FlutterMethodNotImplemented)
        }
    }

    private func handleGetBatteryLevel(result: FlutterResult) {
        UIDevice.current.isBatteryMonitoringEnabled = true
        let batteryLevel = UIDevice.current.batteryLevel

        if batteryLevel < 0 {
            result(FlutterError(
                code: "UNAVAILABLE",
                message: "Battery level not available",
                details: nil
            ))
            return
        }

        result(Int(batteryLevel * 100))
    }

    private func handleIsCharging(result: FlutterResult) {
        UIDevice.current.isBatteryMonitoringEnabled = true
        let state = UIDevice.current.batteryState
        result(state == .charging || state == .full)
    }

    private func handleGetBatteryInfo(result: FlutterResult) {
        UIDevice.current.isBatteryMonitoringEnabled = true

        let info: [String: Any] = [
            "level": Int(UIDevice.current.batteryLevel * 100),
            "state": batteryStateToString(UIDevice.current.batteryState),
            "isLowPowerMode": ProcessInfo.processInfo.isLowPowerModeEnabled
        ]

        result(info)
    }

    private func batteryStateToString(_ state: UIDevice.BatteryState) -> String {
        switch state {
        case .unknown: return "unknown"
        case .unplugged: return "unplugged"
        case .charging: return "charging"
        case .full: return "full"
        @unknown default: return "unknown"
        }
    }
}
```

```kotlin
// android/src/main/kotlin/com/example/app/BatteryPlugin.kt
package com.example.app

import android.content.Context
import android.content.Intent
import android.content.IntentFilter
import android.os.BatteryManager
import android.os.Build
import io.flutter.embedding.engine.plugins.FlutterPlugin
import io.flutter.plugin.common.MethodCall
import io.flutter.plugin.common.MethodChannel
import io.flutter.plugin.common.MethodChannel.MethodCallHandler
import io.flutter.plugin.common.MethodChannel.Result

class BatteryPlugin : FlutterPlugin, MethodCallHandler {
    private lateinit var channel: MethodChannel
    private lateinit var context: Context

    override fun onAttachedToEngine(flutterPluginBinding: FlutterPlugin.FlutterPluginBinding) {
        channel = MethodChannel(flutterPluginBinding.binaryMessenger, "com.example.app/battery")
        channel.setMethodCallHandler(this)
        context = flutterPluginBinding.applicationContext
    }

    override fun onMethodCall(call: MethodCall, result: Result) {
        when (call.method) {
            "getBatteryLevel" -> handleGetBatteryLevel(result)
            "isCharging" -> handleIsCharging(result)
            "getBatteryInfo" -> handleGetBatteryInfo(result)
            else -> result.notImplemented()
        }
    }

    private fun handleGetBatteryLevel(result: Result) {
        val batteryLevel = getBatteryLevel()
        if (batteryLevel != -1) {
            result.success(batteryLevel)
        } else {
            result.error("UNAVAILABLE", "Battery level not available", null)
        }
    }

    private fun handleIsCharging(result: Result) {
        val intent = context.registerReceiver(null, IntentFilter(Intent.ACTION_BATTERY_CHANGED))
        val status = intent?.getIntExtra(BatteryManager.EXTRA_STATUS, -1) ?: -1
        val isCharging = status == BatteryManager.BATTERY_STATUS_CHARGING ||
                         status == BatteryManager.BATTERY_STATUS_FULL
        result.success(isCharging)
    }

    private fun handleGetBatteryInfo(result: Result) {
        val intent = context.registerReceiver(null, IntentFilter(Intent.ACTION_BATTERY_CHANGED))

        val level = getBatteryLevel()
        val status = intent?.getIntExtra(BatteryManager.EXTRA_STATUS, -1) ?: -1
        val health = intent?.getIntExtra(BatteryManager.EXTRA_HEALTH, -1) ?: -1
        val temperature = intent?.getIntExtra(BatteryManager.EXTRA_TEMPERATURE, -1) ?: -1

        val info = mapOf(
            "level" to level,
            "status" to batteryStatusToString(status),
            "health" to batteryHealthToString(health),
            "temperature" to temperature / 10.0 // Convert to Celsius
        )

        result.success(info)
    }

    private fun getBatteryLevel(): Int {
        return if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.LOLLIPOP) {
            val batteryManager = context.getSystemService(Context.BATTERY_SERVICE) as BatteryManager
            batteryManager.getIntProperty(BatteryManager.BATTERY_PROPERTY_CAPACITY)
        } else {
            val intent = context.registerReceiver(null, IntentFilter(Intent.ACTION_BATTERY_CHANGED))
            val level = intent?.getIntExtra(BatteryManager.EXTRA_LEVEL, -1) ?: -1
            val scale = intent?.getIntExtra(BatteryManager.EXTRA_SCALE, -1) ?: -1
            if (level >= 0 && scale > 0) (level * 100 / scale) else -1
        }
    }

    private fun batteryStatusToString(status: Int): String = when (status) {
        BatteryManager.BATTERY_STATUS_CHARGING -> "charging"
        BatteryManager.BATTERY_STATUS_DISCHARGING -> "discharging"
        BatteryManager.BATTERY_STATUS_FULL -> "full"
        BatteryManager.BATTERY_STATUS_NOT_CHARGING -> "not_charging"
        else -> "unknown"
    }

    private fun batteryHealthToString(health: Int): String = when (health) {
        BatteryManager.BATTERY_HEALTH_GOOD -> "good"
        BatteryManager.BATTERY_HEALTH_OVERHEAT -> "overheat"
        BatteryManager.BATTERY_HEALTH_DEAD -> "dead"
        BatteryManager.BATTERY_HEALTH_OVER_VOLTAGE -> "over_voltage"
        BatteryManager.BATTERY_HEALTH_COLD -> "cold"
        else -> "unknown"
    }

    override fun onDetachedFromEngine(binding: FlutterPlugin.FlutterPluginBinding) {
        channel.setMethodCallHandler(null)
    }
}
```

### Pattern 2: EventChannel - Streaming Data

```dart
// lib/src/sensor_channel.dart
import 'dart:async';
import 'package:flutter/services.dart';

class SensorChannel {
  static const EventChannel _accelerometerChannel =
      EventChannel('com.example.app/accelerometer');

  static const EventChannel _gyroscopeChannel =
      EventChannel('com.example.app/gyroscope');

  /// Stream accelerometer data
  static Stream<AccelerometerEvent> get accelerometerEvents {
    return _accelerometerChannel.receiveBroadcastStream().map((event) {
      final data = Map<String, dynamic>.from(event);
      return AccelerometerEvent(
        x: data['x'] as double,
        y: data['y'] as double,
        z: data['z'] as double,
        timestamp: data['timestamp'] as int,
      );
    });
  }

  /// Stream gyroscope data
  static Stream<GyroscopeEvent> get gyroscopeEvents {
    return _gyroscopeChannel.receiveBroadcastStream().map((event) {
      final data = Map<String, dynamic>.from(event);
      return GyroscopeEvent(
        x: data['x'] as double,
        y: data['y'] as double,
        z: data['z'] as double,
        timestamp: data['timestamp'] as int,
      );
    });
  }
}

class AccelerometerEvent {
  final double x;
  final double y;
  final double z;
  final int timestamp;

  AccelerometerEvent({
    required this.x,
    required this.y,
    required this.z,
    required this.timestamp,
  });
}

class GyroscopeEvent {
  final double x;
  final double y;
  final double z;
  final int timestamp;

  GyroscopeEvent({
    required this.x,
    required this.y,
    required this.z,
    required this.timestamp,
  });
}
```

```swift
// ios/Classes/SensorPlugin.swift
import Flutter
import CoreMotion

public class SensorPlugin: NSObject, FlutterPlugin, FlutterStreamHandler {
    private var motionManager: CMMotionManager?
    private var accelerometerSink: FlutterEventSink?
    private var gyroscopeSink: FlutterEventSink?

    public static func register(with registrar: FlutterPluginRegistrar) {
        let instance = SensorPlugin()

        let accelerometerChannel = FlutterEventChannel(
            name: "com.example.app/accelerometer",
            binaryMessenger: registrar.messenger()
        )
        accelerometerChannel.setStreamHandler(instance)

        let gyroscopeChannel = FlutterEventChannel(
            name: "com.example.app/gyroscope",
            binaryMessenger: registrar.messenger()
        )
        gyroscopeChannel.setStreamHandler(instance)
    }

    public func onListen(
        withArguments arguments: Any?,
        eventSink events: @escaping FlutterEventSink
    ) -> FlutterError? {
        if motionManager == nil {
            motionManager = CMMotionManager()
        }

        guard let manager = motionManager else {
            return FlutterError(
                code: "UNAVAILABLE",
                message: "Motion manager not available",
                details: nil
            )
        }

        // Determine which sensor based on channel
        let channelName = (arguments as? [String: Any])?["channel"] as? String

        if channelName == "accelerometer" || arguments == nil {
            accelerometerSink = events
            startAccelerometer(manager: manager)
        }

        return nil
    }

    private func startAccelerometer(manager: CMMotionManager) {
        guard manager.isAccelerometerAvailable else { return }

        manager.accelerometerUpdateInterval = 0.1 // 10 Hz
        manager.startAccelerometerUpdates(to: .main) { [weak self] data, error in
            guard let data = data, error == nil else { return }

            let event: [String: Any] = [
                "x": data.acceleration.x * 9.81,
                "y": data.acceleration.y * 9.81,
                "z": data.acceleration.z * 9.81,
                "timestamp": Int(Date().timeIntervalSince1970 * 1000)
            ]

            self?.accelerometerSink?(event)
        }
    }

    public func onCancel(withArguments arguments: Any?) -> FlutterError? {
        motionManager?.stopAccelerometerUpdates()
        motionManager?.stopGyroUpdates()
        accelerometerSink = nil
        gyroscopeSink = nil
        return nil
    }
}
```

```kotlin
// android/src/main/kotlin/com/example/app/SensorPlugin.kt
package com.example.app

import android.content.Context
import android.hardware.Sensor
import android.hardware.SensorEvent
import android.hardware.SensorEventListener
import android.hardware.SensorManager
import io.flutter.embedding.engine.plugins.FlutterPlugin
import io.flutter.plugin.common.EventChannel

class SensorPlugin : FlutterPlugin {
    private lateinit var context: Context
    private var accelerometerChannel: EventChannel? = null
    private var gyroscopeChannel: EventChannel? = null

    override fun onAttachedToEngine(flutterPluginBinding: FlutterPlugin.FlutterPluginBinding) {
        context = flutterPluginBinding.applicationContext

        accelerometerChannel = EventChannel(
            flutterPluginBinding.binaryMessenger,
            "com.example.app/accelerometer"
        )
        accelerometerChannel?.setStreamHandler(
            SensorStreamHandler(context, Sensor.TYPE_ACCELEROMETER)
        )

        gyroscopeChannel = EventChannel(
            flutterPluginBinding.binaryMessenger,
            "com.example.app/gyroscope"
        )
        gyroscopeChannel?.setStreamHandler(
            SensorStreamHandler(context, Sensor.TYPE_GYROSCOPE)
        )
    }

    override fun onDetachedFromEngine(binding: FlutterPlugin.FlutterPluginBinding) {
        accelerometerChannel?.setStreamHandler(null)
        gyroscopeChannel?.setStreamHandler(null)
    }
}

class SensorStreamHandler(
    private val context: Context,
    private val sensorType: Int
) : EventChannel.StreamHandler, SensorEventListener {

    private var sensorManager: SensorManager? = null
    private var sensor: Sensor? = null
    private var eventSink: EventChannel.EventSink? = null

    override fun onListen(arguments: Any?, events: EventChannel.EventSink?) {
        eventSink = events
        sensorManager = context.getSystemService(Context.SENSOR_SERVICE) as SensorManager
        sensor = sensorManager?.getDefaultSensor(sensorType)

        sensor?.let {
            sensorManager?.registerListener(
                this,
                it,
                SensorManager.SENSOR_DELAY_UI
            )
        } ?: run {
            events?.error("UNAVAILABLE", "Sensor not available", null)
        }
    }

    override fun onCancel(arguments: Any?) {
        sensorManager?.unregisterListener(this)
        eventSink = null
    }

    override fun onSensorChanged(event: SensorEvent?) {
        event?.let {
            val data = mapOf(
                "x" to it.values[0].toDouble(),
                "y" to it.values[1].toDouble(),
                "z" to it.values[2].toDouble(),
                "timestamp" to System.currentTimeMillis()
            )
            eventSink?.success(data)
        }
    }

    override fun onAccuracyChanged(sensor: Sensor?, accuracy: Int) {
        // Handle accuracy changes if needed
    }
}
```

### Pattern 3: Pigeon - Type-Safe Code Generation

```dart
// pigeons/messages.dart
import 'package:pigeon/pigeon.dart';

@ConfigurePigeon(PigeonOptions(
  dartOut: 'lib/src/messages.g.dart',
  kotlinOut: 'android/src/main/kotlin/com/example/app/Messages.g.kt',
  kotlinOptions: KotlinOptions(package: 'com.example.app'),
  swiftOut: 'ios/Classes/Messages.g.swift',
))
class UserData {
  String? id;
  String? name;
  String? email;
  int? age;
}

class CreateUserRequest {
  String? name;
  String? email;
  int? age;
}

@HostApi()
abstract class UserApi {
  @async
  UserData getUser(String id);

  @async
  List<UserData> getUsers();

  @async
  UserData createUser(CreateUserRequest request);

  @async
  void deleteUser(String id);
}

@FlutterApi()
abstract class UserEventApi {
  void onUserUpdated(UserData user);
  void onUserDeleted(String id);
}
```

```bash
# Generate type-safe code
dart run pigeon --input pigeons/messages.dart
```

```dart
// lib/src/user_service.dart
import 'messages.g.dart';

class UserService {
  final UserApi _api = UserApi();

  Future<UserData> getUser(String id) async {
    return await _api.getUser(id);
  }

  Future<List<UserData?>> getUsers() async {
    return await _api.getUsers();
  }

  Future<UserData> createUser({
    required String name,
    required String email,
    int? age,
  }) async {
    final request = CreateUserRequest()
      ..name = name
      ..email = email
      ..age = age;

    return await _api.createUser(request);
  }
}
```

### Pattern 4: Platform Views (Native UI in Flutter)

```dart
// lib/src/native_map_view.dart
import 'package:flutter/foundation.dart';
import 'package:flutter/gestures.dart';
import 'package:flutter/material.dart';
import 'package:flutter/rendering.dart';
import 'package:flutter/services.dart';

class NativeMapView extends StatefulWidget {
  final double initialLatitude;
  final double initialLongitude;
  final double initialZoom;
  final void Function(double lat, double lng)? onMapTap;

  const NativeMapView({
    super.key,
    required this.initialLatitude,
    required this.initialLongitude,
    this.initialZoom = 14.0,
    this.onMapTap,
  });

  @override
  State<NativeMapView> createState() => _NativeMapViewState();
}

class _NativeMapViewState extends State<NativeMapView> {
  late MethodChannel _channel;

  @override
  Widget build(BuildContext context) {
    const String viewType = 'com.example.app/native_map';

    final Map<String, dynamic> creationParams = {
      'latitude': widget.initialLatitude,
      'longitude': widget.initialLongitude,
      'zoom': widget.initialZoom,
    };

    switch (defaultTargetPlatform) {
      case TargetPlatform.android:
        return AndroidView(
          viewType: viewType,
          layoutDirection: TextDirection.ltr,
          creationParams: creationParams,
          creationParamsCodec: const StandardMessageCodec(),
          onPlatformViewCreated: _onPlatformViewCreated,
          gestureRecognizers: <Factory<OneSequenceGestureRecognizer>>{
            Factory<OneSequenceGestureRecognizer>(
              () => EagerGestureRecognizer(),
            ),
          },
        );

      case TargetPlatform.iOS:
        return UiKitView(
          viewType: viewType,
          layoutDirection: TextDirection.ltr,
          creationParams: creationParams,
          creationParamsCodec: const StandardMessageCodec(),
          onPlatformViewCreated: _onPlatformViewCreated,
          gestureRecognizers: <Factory<OneSequenceGestureRecognizer>>{
            Factory<OneSequenceGestureRecognizer>(
              () => EagerGestureRecognizer(),
            ),
          },
        );

      default:
        return const Center(
          child: Text('Platform not supported'),
        );
    }
  }

  void _onPlatformViewCreated(int viewId) {
    _channel = MethodChannel('com.example.app/native_map_$viewId');
    _channel.setMethodCallHandler(_handleMethodCall);
  }

  Future<dynamic> _handleMethodCall(MethodCall call) async {
    switch (call.method) {
      case 'onMapTap':
        final args = Map<String, dynamic>.from(call.arguments);
        widget.onMapTap?.call(
          args['latitude'] as double,
          args['longitude'] as double,
        );
        break;
    }
  }

  /// Move map to location
  Future<void> moveTo(double latitude, double longitude, {double? zoom}) async {
    await _channel.invokeMethod('moveTo', {
      'latitude': latitude,
      'longitude': longitude,
      'zoom': zoom ?? widget.initialZoom,
    });
  }

  /// Add marker to map
  Future<void> addMarker(String id, double lat, double lng, String title) async {
    await _channel.invokeMethod('addMarker', {
      'id': id,
      'latitude': lat,
      'longitude': lng,
      'title': title,
    });
  }
}
```

### Pattern 5: Background Execution

```dart
// lib/src/background_service.dart
import 'dart:async';
import 'dart:ui';
import 'package:flutter/services.dart';

class BackgroundService {
  static const MethodChannel _channel =
      MethodChannel('com.example.app/background_service');

  static const String _backgroundChannelName =
      'com.example.app/background_service_background';

  /// Initialize the background service
  static Future<void> initialize({
    required Future<void> Function() onStart,
    int intervalMinutes = 15,
  }) async {
    final CallbackHandle? handle = PluginUtilities.getCallbackHandle(onStart);

    if (handle == null) {
      throw Exception('Failed to get callback handle');
    }

    await _channel.invokeMethod('initialize', {
      'callbackHandle': handle.toRawHandle(),
      'intervalMinutes': intervalMinutes,
    });
  }

  /// Start the background service
  static Future<void> start() async {
    await _channel.invokeMethod('start');
  }

  /// Stop the background service
  static Future<void> stop() async {
    await _channel.invokeMethod('stop');
  }

  /// Check if service is running
  static Future<bool> isRunning() async {
    return await _channel.invokeMethod('isRunning');
  }
}

// Entry point for background execution
@pragma('vm:entry-point')
Future<void> backgroundCallback() async {
  WidgetsFlutterBinding.ensureInitialized();

  const MethodChannel channel =
      MethodChannel('com.example.app/background_service_background');

  channel.setMethodCallHandler((call) async {
    switch (call.method) {
      case 'onBackgroundFetch':
        // Perform background work
        await performBackgroundSync();
        return true;
    }
    return false;
  });
}

Future<void> performBackgroundSync() async {
  // Your background sync logic here
  print('Background sync executed at ${DateTime.now()}');
}
```

## Best Practices

### Do's
- **Use consistent channel naming** - `com.company.app/feature`
- **Handle errors on both sides** - Return proper error codes
- **Validate arguments** - Check for null and type safety
- **Clean up resources** - Implement proper disposal
- **Use Pigeon for complex APIs** - Type-safe code generation
- **Test on real devices** - Emulators may not support all features

### Don'ts
- **Don't block the main thread** - Use async operations
- **Don't forget platform checks** - Some APIs are platform-specific
- **Don't hardcode values** - Pass configuration from Flutter
- **Don't ignore lifecycle** - Handle app suspend/resume
- **Don't skip error handling** - Always handle PlatformException
- **Don't mix channels** - Keep each feature in its own channel

## Resources

- [Platform Channels Documentation](https://docs.flutter.dev/development/platform-integration/platform-channels)
- [Pigeon Package](https://pub.dev/packages/pigeon)
- [Platform Views](https://docs.flutter.dev/development/platform-integration/platform-views)
- [Writing Custom Platform-Specific Code](https://docs.flutter.dev/development/platform-integration/writing-platform-specific-code)
