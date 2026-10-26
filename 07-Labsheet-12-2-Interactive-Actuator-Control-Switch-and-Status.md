# ใบงานการทดลองที่ 12.2
### การสร้างสวิตช์ควบคุมอุปกรณ์สองทาง (Bi-directional Actuator Switch) พร้อมระบบ Feedback และสถานะ LWT

> [!NOTE] **คำชี้แจง**
> ในใบงานนี้นักศึกษาจะได้พัฒนาอินเทอร์เฟซควบคุมอุปกรณ์เปิด-ปิด (Actuator Switch) สไตล์ Glassmorphism ที่สมบูรณ์แบบ โดยมีระบบยืนยันสถานะสองทาง (State Synchronization): เมื่อแตะสวิตช์ แอปจะส่งคำสั่ง JSON ไปยัง Broker และรอรับข้อความยืนยันสถานะจริงจากบอร์ด พร้อมทั้งคอยดักฟังสถานะ Last Will and Testament (LWT) เพื่อแจ้งเตือนทันทีหากบอร์ดหลุดการเชื่อมต่อ

---

## 1. วัตถุประสงค์การทดลอง
1. สามารถออกแบบ UI สวิตช์เปิด-ปิดไฟที่มีเอฟเฟกต์แสงเรือง (Glow Effect) และการสั่นตอบสนอง (Haptic Feedback) ได้
2. สามารถส่งข้อความคำสั่ง JSON Payload ผ่านหัวข้อ `iot/esp32/control` ได้
3. สามารถดักฟังหัวข้อ `iot/esp32/status` เพื่อแสดงสถานะ Online/Offline ของบอร์ดตามกลไก LWT ได้
4. สามารถทดสอบการประสานสถานะ (State Sync) แบบจำลองสถานการณ์ความผิดพลาดของเครือข่ายได้

---

## 2. โครงสร้างหัวข้อ MQTT (Topic Architecture)

| ชื่อหัวข้อ (Topic) | ทิศทางการส่ง | ชนิดข้อมูล (Payload Format) | วัตถุประสงค์ |
| :--- | :--- | :--- | :--- |
| `iot/esp32/control` | Flutter -> ESP32 | `{"led": "ON"}` หรือ `{"led": "OFF"}` | ส่งคำสั่งควบคุมสวิตช์ไฟ |
| `iot/esp32/led_ack` | ESP32 -> Flutter | `{"led_state": true, "ack": true}` | ส่งการตอบรับสถานะจริงทางกายภาพ |
| `iot/esp32/status` | ESP32 -> Broker (LWT) | Plaintext: `"online"` หรือ `"offline"` | ตัวบ่งชี้ความพร้อมทำงานของบอร์ด |

---

## 3. ขั้นตอนการทดลอง (Step-by-Step Activities)

### กิจกรรมที่ 12-2.1: การเตรียมโปรเจกต์และการสร้างหน้าจอควบคุมสวิตช์

#### 1. คัดลอกโปรเจกต์จาก Lab 12-1
เพื่อประหยัดเวลาและไม่ต้องติดตั้งแพ็กเกจ `mqtt_client` ใหม่อีกครั้ง ให้นักศึกษาคัดลอกโฟลเดอร์โปรเจกต์จาก Lab 12-1 มาเป็นโฟลเดอร์สำหรับ Lab 12-2

**คำสั่ง PowerShell:**
```powershell
# 1. คัดลอกโฟลเดอร์โปรเจกต์จาก Lab12-1 มาเป็น Lab12-2
Copy-Item -Recurse -Path "..\Lab12-1\iot_mqtt_basics" -Destination "..\Lab12-2\iot_actuator_switch"

# 2. ย้ายเข้าไปยังโฟลเดอร์โปรเจกต์ใหม่
cd ..\Lab12-2\iot_actuator_switch

# 3. ล้างแคชการคอมไพล์เดิมออก (จำเป็นอย่างยิ่งเพื่อป้องกัน CMake Error จาก Path เก่า)
flutter clean
```
*(หรือคลิกขวา Copy โฟลเดอร์ `iot_mqtt_basics` ใน File Explorer แล้ว Paste วางในโฟลเดอร์ใหม่ และรันคำสั่ง `flutter clean` เสมอ)*

#### 2. สร้างไฟล์หน้าจอควบคุมสวิตช์ (`lib/actuator_control_screen.dart`)
ในโฟลเดอร์ `lib/` ให้สร้างไฟล์ใหม่ชื่อ `actuator_control_screen.dart` และใส่โค้ดต่อไปนี้

```dart
import 'dart:convert';
import 'package:flutter/material.dart';
import 'package:flutter/services.dart';
import 'package:mqtt_client/mqtt_client.dart';
import 'package:mqtt_client/mqtt_server_client.dart';

class ActuatorControlScreen extends StatefulWidget {
  final String brokerIp;
  const ActuatorControlScreen({super.key, required this.brokerIp});

  @override
  State<ActuatorControlScreen> createState() => _ActuatorControlScreenState();
}

class _ActuatorControlScreenState extends State<ActuatorControlScreen> {
  MqttServerClient? _client;
  bool _isLedOn = false;
  bool _isDeviceOnline = false;
  bool _isWaitingAck = false;
  String _networkStatus = 'กำลังเชื่อมต่อ Broker...';

  @override
  void initState() {
    super.initState();
    _connectMqtt();
  }

  @override
  void dispose() {
    _client?.disconnect();
    super.dispose();
  }

  Future<void> _connectMqtt() async {
    final clientId = 'Flutter_Switch_${DateTime.now().millisecondsSinceEpoch}';
    _client = MqttServerClient(widget.brokerIp, clientId);
    _client!.port = 1883;
    _client!.keepAlivePeriod = 20;

    _client!.onConnected = () {
      setState(() => _networkStatus = 'เชื่อมต่อ Broker เรียบร้อย');
      _subscribeTopics();
    };

    _client!.onDisconnected = () {
      setState(() {
        _networkStatus = 'ขาดการเชื่อมต่อ Broker!';
        _isDeviceOnline = false;
      });
    };

    try {
      await _client!.connect();
    } catch (e) {
      setState(() => _networkStatus = 'เชื่อมต่อล้มเหลว: $e');
    }
  }

  void _subscribeTopics() {
    // 1. บอกรับหัวข้อสถานะบอร์ด (LWT)
    _client?.subscribe('iot/esp32/status', MqttQos.atMostOnce);
    // 2. บอกรับหัวข้อยืนยันการทำงานของหลอดไฟ
    _client?.subscribe('iot/esp32/led_ack', MqttQos.atMostOnce);

    _client?.updates?.listen((List<MqttReceivedMessage<MqttMessage>> c) {
      final recMess = c[0].payload as MqttPublishMessage;
      final pt = MqttPublishPayload.bytesToStringAsString(recMess.payload.message);
      final topic = c[0].topic;

      setState(() {
        if (topic == 'iot/esp32/status') {
          _isDeviceOnline = (pt.trim().toLowerCase() == 'online');
        } else if (topic == 'iot/esp32/led_ack') {
          _isWaitingAck = false;
          try {
            final data = jsonDecode(pt);
            _isLedOn = data['led_state'] == true;
          } catch (_) {}
        }
      });
    });
  }

  void _toggleLed(bool targetState) {
    if (_client?.connectionStatus?.state != MqttConnectionState.connected) {
      ScaffoldMessenger.of(context).showSnackBar(
        const SnackBar(content: Text('⚠️ กรุณาเชื่อมต่อ Broker ก่อนสั่งการ')),
      );
      return;
    }

    if (!_isDeviceOnline) {
      ScaffoldMessenger.of(context).showSnackBar(
        const SnackBar(content: Text('⚠️ อุปกรณ์ ESP32 ออฟไลน์อยู่ ไม่สามารถส่งคำสั่งได้')),
      );
      return;
    }

    // แตะแล้วสั่งสั่น Haptic Feedback
    HapticFeedback.lightImpact();

    setState(() {
      _isWaitingAck = true;
    });

    final payloadMap = {'led': targetState ? 'ON' : 'OFF'};
    final jsonStr = jsonEncode(payloadMap);
    final builder = MqttClientPayloadBuilder();
    builder.addString(jsonStr);

    _client?.publishMessage(
      'iot/esp32/control',
      MqttQos.atLeastOnce,
      builder.payload!,
    );
  }

  @override
  Widget build(BuildContext context) {
    return Scaffold(
      backgroundColor: const Color(0xFF0F172A),
      appBar: AppBar(
        title: const Text('Actuator Control Center'),
        backgroundColor: const Color(0xFF1E293B),
      ),
      body: Padding(
        padding: const EdgeInsets.all(20.0),
        child: Column(
          children: [
            // แถบแสดงสถานะ Device Online/Offline
            Container(
              padding: const EdgeInsets.symmetric(horizontal: 16, vertical: 12),
              decoration: BoxDecoration(
                color: _isDeviceOnline ? const Color(0x1A00E676) : const Color(0x1AFF5252),
                borderRadius: BorderRadius.circular(16),
                border: Border.all(
                  color: _isDeviceOnline ? const Color(0xFF00E676) : const Color(0xFFFF5252),
                ),
              ),
              child: Row(
                children: [
                  Icon(
                    Icons.circle,
                    size: 12,
                    color: _isDeviceOnline ? const Color(0xFF00E676) : const Color(0xFFFF5252),
                  ),
                  const SizedBox(width: 10),
                  Text(
                    _isDeviceOnline ? 'ESP32 ฮาร์ดแวร์: ออนไลน์ (ONLINE)' : 'ESP32 ฮาร์ดแวร์: ออฟไลน์ (OFFLINE)',
                    style: TextStyle(
                      fontWeight: FontWeight.bold,
                      color: _isDeviceOnline ? const Color(0xFF00E676) : const Color(0xFFFF5252),
                    ),
                  ),
                ],
              ),
            ),
            const SizedBox(height: 30),

            // การ์ดควบคุมหลอดไฟแบบ Glassmorphism
            Container(
              padding: const EdgeInsets.all(24),
              decoration: BoxDecoration(
                color: const Color(0xFF1E293B).withOpacity(0.8),
                borderRadius: BorderRadius.circular(24),
                border: Border.all(
                  color: _isLedOn ? const Color(0xFF00E676).withOpacity(0.6) : Colors.white12,
                  width: 2,
                ),
                boxShadow: _isLedOn
                    ? [
                        BoxShadow(
                          color: const Color(0xFF00E676).withOpacity(0.25),
                          blurRadius: 24,
                          spreadRadius: 4,
                        )
                      ]
                    : [],
              ),
              child: Column(
                children: [
                  // ไอคอนหลอดไฟพร้อมเอฟเฟกต์แสง
                  AnimatedContainer(
                    duration: const Duration(milliseconds: 300),
                    padding: const EdgeInsets.all(24),
                    decoration: BoxDecoration(
                      shape: BoxShape.circle,
                      color: _isLedOn ? const Color(0x3300E676) : Colors.black26,
                    ),
                    child: Icon(
                      Icons.lightbulb,
                      size: 72,
                      color: _isLedOn ? const Color(0xFF00E676) : Colors.white24,
                    ),
                  ),
                  const SizedBox(height: 20),
                  const Text(
                    'หลอดไฟ LED ทดสอบ (GPIO 2)',
                    style: TextStyle(fontSize: 18, fontWeight: FontWeight.bold, color: Colors.white),
                  ),
                  const SizedBox(height: 6),
                  Text(
                    _isWaitingAck
                        ? 'กำลังรอการยืนยันจากบอร์ด (Waiting ACK)...'
                        : _isLedOn
                            ? 'สถานะ: เปิดสว่าง (Active)'
                            : 'สถานะ: ปิดการทำงาน (Standby)',
                    style: TextStyle(
                      color: _isWaitingAck ? Colors.amber : (_isLedOn ? const Color(0xFF00E676) : Colors.white38),
                    ),
                  ),
                  const SizedBox(height: 24),
                  // สวิตช์ควบคุม
                  Transform.scale(
                    scale: 1.4,
                    child: Switch(
                      value: _isLedOn,
                      activeColor: const Color(0xFF00E676),
                      onChanged: _isWaitingAck ? null : _toggleLed,
                    ),
                  ),
                ],
              ),
            ),
            const Spacer(),
            Text(
              _networkStatus,
              style: const TextStyle(fontSize: 12, color: Colors.white38),
            ),
          ],
        ),
      ),
    );
  }
}
```

#### 3. ปรับปรุงไฟล์ `lib/main.dart` เพื่อเรียกใช้งานหน้าจอใหม่
เปิดไฟล์ `lib/main.dart` และแก้ไขโค้ดทั้งหมดให้เรียกใช้งาน `ActuatorControlScreen` เป็นหน้าจอเริ่มต้น (Home Screen)

```dart
import 'package:flutter/material.dart';
import 'actuator_control_screen.dart';

void main() {
  runApp(const ActuatorApp());
}

class ActuatorApp extends StatelessWidget {
  const ActuatorApp({super.key});

  @override
  Widget build(BuildContext context) {
    return MaterialApp(
      title: 'Lab 12.2 Actuator Switch',
      theme: ThemeData.dark().copyWith(
        scaffoldBackgroundColor: const Color(0xFF0F172A),
        colorScheme: const ColorScheme.dark(
          primary: Color(0xFF00E5FF),
          secondary: Color(0xFF00E676),
        ),
      ),
      // กำหนด IP ของ Mosquitto Broker (ใช้ 127.0.0.1 หากรันบนคอมพิวเตอร์เครื่องเดียวกัน)
      home: const ActuatorControlScreen(brokerIp: '127.0.0.1'),
      debugShowCheckedModeBanner: false,
    );
  }
}
```

#### 4. สั่งรัน Flutter App
```powershell
# รันบน Windows Desktop
flutter run -d windows

# หรือรันบน Android Device / Emulator
flutter run -d <device-id>
```

---

### กิจกรรมที่ 12-2.2: การทดสอบร่วมกับสคริปต์จำลองฮาร์ดแวร์ (Mock Simulation)

หากยังไม่ได้ต่อบอร์ด ESP32 จริง สามารถใช้ PowerShell ทดสอบจำลองพฤติกรรมของฮาร์ดแวร์ได้

#### 1. จำลองส่งสถานะ Online (เสมือน ESP32 บูตเสร็จ)
```powershell
mosquitto_pub -h 127.0.0.1 -p 1883 -t "iot/esp32/status" -m "online" -r
```
*สังเกตหน้าจอ Flutter แถบสถานะด้านบนจะเปลี่ยนเป็นสีเขียวทันที*


<img width="1296" height="990" alt="image" src="https://github.com/user-attachments/assets/d1a092f9-0fdc-4daf-8c2b-6504e584d44b" />


#### 2. เฝ้าดูคำสั่งที่ส่งมาจากแอป Flutter
```powershell
mosquitto_sub -h 127.0.0.1 -p 1883 -t "iot/esp32/control" -v
```
*เมื่อแตะสวิตช์บน Flutter จะเห็นแพ็กเก็ต `{"led":"ON"}` หรือ `{"led":"OFF"}` ปรากฏในเทอร์มินัล*

<img width="865" height="118" alt="image" src="https://github.com/user-attachments/assets/0756dcf8-06af-44e8-bf7d-c680d4dbd5bd" />


#### 3. ส่งคำสั่งยืนยันกลับไปยังแอป (เสมือนบอร์ดเปลี่ยนสถานะขา GPIO สำเร็จ)
```powershell
mosquitto_pub -h 127.0.0.1 -p 1883 -t "iot/esp32/led_ack" -m '{"led_state": true, "ack": true}'
```
*สังเกตสวิตช์และไอคอนหลอดไฟบนมือถือจะเรืองแสงสีเขียวทันที*

#### 4. จำลองเหตุการณ์บอร์ดดับหรือเน็ตหลุด (เสมือน Broker ส่ง LWT)
```powershell
mosquitto_pub -h 127.0.0.1 -p 1883 -t "iot/esp32/status" -m "offline" -r
```
*สังเกตว่าหน้าจอจะแจ้งเตือนว่า ESP32 ออฟไลน์ทันที และสวิตช์จะไม่ยอมให้สั่งการ*

---

## 4. สรุปผลและคำถามท้ายการทดลอง
1. ประโยชน์ของการรอสัญญาณ ACK จากหัวข้อ `iot/esp32/led_ack` แทนการเปลี่ยน UI ทันที (Optimistic Update) คืออะไร?
2. ทำไมเราจึงควรตั้งแฟล็ก Retain (`-r`) สำหรับหัวข้อสถานะ `iot/esp32/status`?
