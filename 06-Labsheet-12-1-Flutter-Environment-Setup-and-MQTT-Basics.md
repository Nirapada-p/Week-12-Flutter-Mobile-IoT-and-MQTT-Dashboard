# ใบงานการทดลองที่ 12.1
### การติดตั้งสภาพแวดล้อม Flutter และการพัฒนา MQTT Client พื้นฐานบนสมาร์ตโฟน

> [!NOTE] **คำชี้แจง**
> ในใบงานนี้นักศึกษาจะได้เริ่มต้นสร้างโปรเจกต์โมบายล์แอปพลิเคชันด้วย **Flutter SDK**, ติดตั้งคอมโพเนนต์ **`mqtt_client`**, และเขียนโค้ดสำหรับสร้างการเชื่อมต่อเครือข่ายไปยัง **Eclipse Mosquitto Broker** โดยมีหน้าจออินเทอร์เฟซแสดงผลสถานะการเชื่อมต่อ และปุ่มกดทดสอบการ Publish และ Subscribe ข้อความเบื้องต้น

---

## 1. วัตถุประสงค์การทดลอง
1. สามารถติดตั้งและตรวจสอบความพร้อมของ Flutter SDK ผ่านเครื่องมือ `flutter doctor` ได้
2. สามารถสร้างโปรเจกต์ Flutter ใหม่ และกำหนดค่าแพ็กเกจ `mqtt_client` ใน `pubspec.yaml` ได้
3. สามารถเขียนโค้ดภาษา Dart เพื่อเชื่อมต่อและตัดการเชื่อมต่อกับ Mosquitto Broker ผ่านพอร์ต 1883 ได้
4. สามารถส่งข้อความ (Publish) และบอกรับหัวข้อ (Subscribe) ทดสอบร่วมกับโปรแกรม Mosquitto CLI บนคอมพิวเตอร์ได้

---

## 2. โครงสร้างการเชื่อมต่อระบบ (System Architecture)

<p align="center"><img src="Images/Lab12.1-1_themed_academic.svg" ></p>

---

## 3. ขั้นตอนการทดลอง (Step-by-Step Activities)

### กิจกรรมที่ 12-1.1: ตรวจสอบและสร้างโปรเจกต์ Flutter

#### 1. ตรวจสอบความพร้อมของเครื่องมือ Flutter
เปิด PowerShell และตรวจสอบสถานะของ Flutter SDK:
```powershell
flutter doctor
```
*(ตรวจสอบว่า Flutter และ Windows Desktop หรือ Android Toolchain พร้อมใช้งาน)*

#### 2. สร้างโฟลเดอร์และโปรเจกต์ Flutter ใหม่
```powershell
flutter create iot_mqtt_basics
cd iot_mqtt_basics
```

#### 3. ติดตั้งแพ็กเกจ `mqtt_client`
รันคำสั่งเพิ่ม Dependency:
```powershell
flutter pub add mqtt_client
```
หรือตรวจสอบในไฟล์ `pubspec.yaml` ว่ามีส่วนต่อไปนี้:
```yaml
dependencies:
  flutter:
    sdk: flutter
  mqtt_client: ^10.3.0
```

---

### กิจกรรมที่ 12-1.2: การเขียนโปรแกรม MQTT Client (`lib/main.dart`)

เปิดไฟล์ `lib/main.dart` แล้วแทนที่ด้วยโค้ดสมบูรณ์ต่อไปนี้:

```dart
import 'dart:io';
import 'package:flutter/material.dart';
import 'package:mqtt_client/mqtt_client.dart';
import 'package:mqtt_client/mqtt_server_client.dart';

void main() {
  runApp(const MqttBasicsApp());
}

class MqttBasicsApp extends StatelessWidget {
  const MqttBasicsApp({super.key});

  @override
  Widget build(BuildContext context) {
    return MaterialApp(
      title: 'Lab 12.1 MQTT Basics',
      theme: ThemeData.dark().copyWith(
        scaffoldBackgroundColor: const Color(0xFF0F172A),
        colorScheme: const ColorScheme.dark(
          primary: Color(0xFF00E5FF),
          secondary: Color(0xFF00E676),
        ),
      ),
      home: const MqttBasicsScreen(),
      debugShowCheckedModeBanner: false,
    );
  }
}

class MqttBasicsScreen extends StatefulWidget {
  const MqttBasicsScreen({super.key});

  @override
  State<MqttBasicsScreen> createState() => _MqttBasicsScreenState();
}

class _MqttBasicsScreenState extends State<MqttBasicsScreen> {
  // หมายเลข IP ของเครื่องคอมพิวเตอร์ที่รัน Mosquitto Broker
  final TextEditingController _brokerIpController =
      TextEditingController(text: '127.0.0.1');
  
  MqttServerClient? _client;
  String _statusText = 'ยังไม่ได้เชื่อมต่อ (Disconnected)';
  Color _statusColor = Colors.redAccent;
  final List<String> _receivedLogs = [];

  @override
  void dispose() {
    _brokerIpController.dispose();
    _client?.disconnect();
    super.dispose();
  }

  Future<void> _connectToBroker() async {
    setState(() {
      _statusText = 'กำลังเชื่อมต่อไปยัง Broker...';
      _statusColor = Colors.amber;
    });

    final broker = _brokerIpController.text.trim();
    final clientId = 'Flutter_Lab12_1_${DateTime.now().millisecondsSinceEpoch}';

    _client = MqttServerClient(broker, clientId);
    _client!.port = 1883;
    _client!.keepAlivePeriod = 20;
    _client!.autoReconnect = true;

    _client!.onConnected = () {
      setState(() {
        _statusText = 'เชื่อมต่อสำเร็จ (Connected)';
        _statusColor = const Color(0xFF00E676);
      });
      _subscribeToTestTopic();
    };

    _client!.onDisconnected = () {
      setState(() {
        _statusText = 'การเชื่อมต่อหลุด (Disconnected)';
        _statusColor = Colors.redAccent;
      });
    };

    final connMessage = MqttConnectMessage()
        .withClientIdentifier(clientId)
        .startClean()
        .withWillTopic('iot/flutter/lwt')
        .withWillMessage('Client crashed')
        .withWillQos(MqttQos.atLeastOnce);

    _client!.connectionMessage = connMessage;

    try {
      await _client!.connect();
    } on SocketException catch (e) {
      _appendLog('❌ Socket Error: ไม่สามารถต่อ Broker IP ได้ ($e)');
      _client?.disconnect();
    } catch (e) {
      _appendLog('❌ Error: $e');
      _client?.disconnect();
    }
  }

  void _disconnectFromBroker() {
    _client?.disconnect();
    _appendLog('ℹ️ ตัดการเชื่อมต่อเรียบร้อย');
  }

  void _subscribeToTestTopic() {
    const topic = 'test/broker/echo';
    _client?.subscribe(topic, MqttQos.atMostOnce);
    _appendLog('📢 Subscribed to: $topic');

    _client?.updates?.listen((List<MqttReceivedMessage<MqttMessage>> c) {
      final recMess = c[0].payload as MqttPublishMessage;
      final pt = MqttPublishPayload.bytesToStringAsString(
        recMess.payload.message,
      );
      _appendLog('📩 [${c[0].topic}] -> $pt');
    });
  }

  void _publishTestMessage() {
    if (_client?.connectionStatus?.state != MqttConnectionState.connected) {
      _appendLog('⚠️ ไม่สามารถส่งข้อความได้เนื่องจากยังไม่ได้เชื่อมต่อ');
      return;
    }

    const topic = 'test/flutter/ping';
    final builder = MqttClientPayloadBuilder();
    builder.addString('Hello from Flutter at ${DateTime.now().toIso8601String()}');

    _client?.publishMessage(
      topic,
      MqttQos.atMostOnce,
      builder.payload!,
    );

    _appendLog('🚀 Published -> $topic');
  }

  void _appendLog(String log) {
    setState(() {
      _receivedLogs.insert(0, '[${DateTime.now().toLocal().toString().substring(11, 19)}] $log');
      if (_receivedLogs.length > 50) _receivedLogs.removeLast();
    });
  }

  @override
  Widget build(BuildContext context) {
    return Scaffold(
      appBar: AppBar(
        title: const Text('Lab 12.1: MQTT Client Basics'),
        backgroundColor: const Color(0xFF1E293B),
      ),
      body: Padding(
        padding: const EdgeInsets.all(16.0),
        child: Column(
          children: [
            // ส่วนกรอก IP Broker
            TextField(
              controller: _brokerIpController,
              decoration: const InputDecoration(
                labelText: 'Broker IPv4 Address',
                border: OutlineInputBorder(),
                prefixIcon: Icon(Icons.dns),
              ),
            ),
            const SizedBox(height: 12),

            // แถบแสดงสถานะ
            Container(
              padding: const EdgeInsets.all(12),
              decoration: BoxDecoration(
                color: _statusColor.withOpacity(0.15),
                borderRadius: BorderRadius.circular(10),
                border: Border.all(color: _statusColor),
              ),
              child: Row(
                children: [
                  Icon(Icons.circle, color: _statusColor, size: 14),
                  const SizedBox(width: 8),
                  Text(
                    _statusText,
                    style: TextStyle(
                      color: _statusColor,
                      fontWeight: FontWeight.bold,
                    ),
                  ),
                ],
              ),
            ),
            const SizedBox(height: 12),

            // ปุ่มกดสั่งการ
            Row(
              children: [
                Expanded(
                  child: ElevatedButton.icon(
                    onPressed: _connectToBroker,
                    icon: const Icon(Icons.link),
                    label: const Text('Connect'),
                    style: ElevatedButton.styleFrom(
                      backgroundColor: const Color(0xFF00E5FF),
                      foregroundColor: Colors.black,
                    ),
                  ),
                ),
                const SizedBox(width: 8),
                Expanded(
                  child: OutlinedButton.icon(
                    onPressed: _disconnectFromBroker,
                    icon: const Icon(Icons.link_off),
                    label: const Text('Disconnect'),
                  ),
                ),
              ],
            ),
            const SizedBox(height: 8),
            SizedBox(
              width: double.infinity,
              child: ElevatedButton.icon(
                onPressed: _publishTestMessage,
                icon: const Icon(Icons.send),
                label: const Text('Publish Test Message (Ping)'),
                style: ElevatedButton.styleFrom(
                  backgroundColor: const Color(0xFF00E676),
                  foregroundColor: Colors.black,
                ),
              ),
            ),
            const SizedBox(height: 16),

            // กล่องข้อความ Logs
            const Align(
              alignment: Alignment.centerLeft,
              child: Text(
                'บันทึกการทำงาน (Transaction Logs):',
                style: TextStyle(fontWeight: FontWeight.bold),
              ),
            ),
            const SizedBox(height: 8),
            Expanded(
              child: Container(
                padding: const EdgeInsets.all(10),
                decoration: BoxDecoration(
                  color: Colors.black26,
                  borderRadius: BorderRadius.circular(8),
                  border: Border.all(color: Colors.white12),
                ),
                child: ListView.builder(
                  itemCount: _receivedLogs.length,
                  itemBuilder: (context, index) => Text(
                    _receivedLogs[index],
                    style: const TextStyle(
                      fontFamily: 'monospace',
                      fontSize: 12,
                      color: Colors.white70,
                    ),
                  ),
                ),
              ),
            ),
          ],
        ),
      ),
    );
  }
}
```

---

### กิจกรรมที่ 12-1.3: การทดสอบร่วมกับ Mosquitto CLI

#### 1. เปิด Mosquitto Broker บนเครื่องคอมพิวเตอร์
```powershell
mosquitto -v -p 1883
```

> [!NOTE]
> ใน Mosquitto เวอร์ชัน 2.x ขึ้นไป การรันด้วยคำสั่ง `mosquitto -v -p 1883` โดยไม่มีไฟล์คอนฟิก โบรกเกอร์จะดักฟังเฉพาะ Loopback Interface (**`127.0.0.1`**) เท่านั้น ดังนั้นหากทดสอบแอป Flutter บน Windows Desktop เครื่องเดียวกันนี้ ให้กรอก IP เป็น **`127.0.0.1`**  
> *(หากต้องการให้เชื่อมต่อผ่าน IP วง LAN เช่น `192.168.0.x` หรือจากบอร์ด ESP32 ภายนอก จะต้องรันผ่านไฟล์คอนฟิกที่มีการระบุ `listener 1883 0.0.0.0` และ `allow_anonymous true`)*

#### 2. เปิดหน้าต่าง PowerShell เพื่อคอยดักฟังข้อความจาก Flutter
```powershell
mosquitto_sub -h 127.0.0.1 -p 1883 -t "test/flutter/ping" -v
```

#### 3. สั่งรัน Flutter App
> [!WARNING]
> **ข้อควรระวังในการเลือกแพลตฟอร์มทดสอบ:**  
> โค้ดในแล็บนี้ใช้คลาส `MqttServerClient` เพื่อสร้าง Raw TCP Socket ไปยังพอร์ต 1883 ผ่าน `dart:io` ซึ่ง Web Browser (Chrome) ไม่รองรับ (หากรันด้วย `flutter run -d chrome` จะเกิดข้อผิดพลาด `Unsupported operation: default SecurityContext getter` และแอปจะค้าง) ดังนั้น **แนะนำให้รันบน Windows Desktop หรือ Android Device/Emulator เท่านั้น**

```powershell
# รันบน Windows Desktop (แนะนำสำหรับการทดสอบบนคอมพิวเตอร์)
flutter run -d windows

# หรือรันบน Android Device / Emulator
flutter run -d <device-id>
```

#### 4. ทดสอบส่งข้อความสองทาง
* บนหน้าจอ Flutter: กรอก IP เป็น **`127.0.0.1`** (หรือ IP ของ Broker ที่เปิดรับ) แล้วกดปุ่ม **Connect**
* เมื่อสถานะเปลี่ยนเป็นสีเขียว ให้กดปุ่ม **Publish Test Message**
* สังเกตหน้าต่าง PowerShell `mosquitto_sub` ว่าได้รับข้อความ Ping หรือไม่
* ในอีกหน้าต่าง PowerShell ส่งข้อความตอบกลับไปยังหัวข้อที่ Flutter รอฟัง:
```powershell
mosquitto_pub -h 127.0.0.1 -p 1883 -t "test/broker/echo" -m "Echo response from CLI"
```
* ตรวจสอบว่าในช่อง Logs ของ Flutter App แสดงข้อความที่ได้รับอย่างถูกต้อง


<img width="1577" height="888" alt="image" src="https://github.com/user-attachments/assets/b41cfbc4-4bc0-426b-a321-9fea0a3ab316" />


<img width="1577" height="887" alt="image" src="https://github.com/user-attachments/assets/577f7d5d-f42e-45ea-9b41-4a868e70a80c" />


<img width="1587" height="890" alt="image" src="https://github.com/user-attachments/assets/d5a28116-d426-4ab1-b96e-0755e4a905b5" />


<img width="1475" height="437" alt="image" src="https://github.com/user-attachments/assets/7d9a763f-1fa7-403c-ab5e-3b3fa538c33e" />


<img width="1487" height="347" alt="image" src="https://github.com/user-attachments/assets/17eb28b8-6b07-46da-bf40-d272ce3bfa26" />


<img width="1477" height="426" alt="image" src="https://github.com/user-attachments/assets/bc6849b2-a539-4a4e-8b8d-eae627287067" />


---

## 4. คำถามท้ายการทดลอง (Review Questions)
1. เพราะเหตุใดเราจึงต้องสุ่มชื่อ `clientId` เช่นการใส่ Timestamp ต่อท้าย แทนที่จะตั้งชื่อแบบคงที่?
2. หากหน้าจอ Flutter รันบนอุปกรณ์จริง (Physical Android Phone) เหตุใดการกรอก IP `127.0.0.1` จึงไม่สามารถเชื่อมต่อ Broker ได้? และต้องใช้ IP ใด?
3. การตั้งค่า `keepAlivePeriod` มีความจำเป็นอย่างไรในสภาพแวดล้อมเครือข่ายไร้สาย?
