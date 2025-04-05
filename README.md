# 🚗 SMART CAR PARKING PTIT 🚗

### 📌 Giới thiệu

   Dự án này xây dựng mô hình bãi đỗ xe thông minh sử dụng ESP32 và các cảm biến để xác định vị trí còn trống. Hệ thống có thể tự động mở/đóng cổng bằng RFID và servo, đồng thời kiểm tra xe bằng cảm biến hồng ngoại, siêu âm, và trọng lượng. Kết quả được hiển thị trên LCD và có thể truy cập từ Web Server.

![image](https://github.com/user-attachments/assets/f8b82753-964e-4cea-937d-c342ede6e6d2)

### 🎯 Chức năng chính

   ✅ Xác định vị trí trống: Sử dụng cảm biến trọng lượng, cảm biến vật cản và cảm biến khoảng cách.

   ✅ Tự động mở/đóng cổng: Dùng RFID quét thẻ để vào/ra, điều khiển servo.

   ✅ Hiển thị thông tin: Trên màn hình LCD và giao diện WebServer.

   ✅ Cập nhật thời gian thực: Đồng bộ với NTP Server để hiển thị thời gian chính xác.

   ✅ Hệ thống quản lý từ xa: Điều khiển trạng thái cổng và xem log từ trình duyệt.

### 🔧 Phần cứng sử dụng

   🖥 ESP32 - Bộ điều khiển chính

  📡 RFID RC522 (x2) - Nhận diện thẻ mở cổng

  ⚖ Cảm biến trọng lượng HX711 (x2) - Xác định xe có đỗ hay không

  🚦 Cảm biến vật cản hồng ngoại - Kiểm tra có vật cản không

  📏 Cảm biến khoảng cách HC-SR04 - Định vị xe trong phạm vi 5-10m

  🔄 Servo (x2) - Điều khiển mở/đóng cổng

  📟 LCD 16x2 I2C - Hiển thị thông tin trạng thái

  💡 LED - Báo hiệu trạng thái bãi đỗ

### 📜 Sơ đồ kết nối

![image](https://github.com/user-attachments/assets/f2d93157-46ba-41f5-a450-92673835f067)


### 🚀 Cách sử dụng

1️⃣ Cài đặt thư viện cần thiết

Mở Arduino IDE, vào Library Manager, tìm và cài đặt các thư viện sau:

  📶 WiFiManager - Quản lý kết nối WiFi cho ESP32

  🎫 MFRC522 - Thư viện đọc thẻ RFID

  ⚙ ESP32Servo - Điều khiển servo với ESP32

  ⏳ NTPClient - Đồng bộ thời gian từ NTP Server

  ⚖ HX711 - Đọc giá trị từ cảm biến trọng lượng

  📟 LiquidCrystal_I2C - Điều khiển LCD qua giao tiếp I2C

2️⃣ Kết nối phần cứng

  🔌 Kết nối ESP32 với các cảm biến theo sơ đồ trên.

  ⚡ Cấp nguồn cho mạch.

3️⃣ Nạp chương trình

  📂 Mở file .ino trên Arduino IDE.

  🔧 Chọn board ESP32 Dev Module.

  🔌 Chọn cổng COM phù hợp và Upload chương trình.

4️⃣ Kết nối WiFi

  📶 Khi ESP32 khởi động, nó sẽ tạo một WiFi AP có tên "ESP32-TMT".

  📱 Kết nối điện thoại/laptop với WiFi này.

  🌍 Truy cập 192.168.4.1 để thiết lập WiFi.

5️⃣ Sử dụng hệ thống

  🎫 Quét thẻ RFID để mở cổng 🚗

  📟 Kiểm tra vị trí trống trên LCD

  💻 Truy cập http://[IP-ESP32] để quản lý bãi đỗ trên Web

## 📜 Các hàm quan trọng

### 🎫 Xử lý RFID quét thẻ  

```cpp
void handleRFID(MFRC522 &rfid, Servo &servo, String gateName, String action) {
    if (rfid.PICC_IsNewCardPresent() && rfid.PICC_ReadCardSerial()) {
        String uid = "";
        for (byte i = 0; i < rfid.uid.size; i++) {
            uid += String(rfid.uid.uidByte[i], HEX);
        }
        uid.toUpperCase();
        Serial.println("UID: " + uid);

        String entryTime = getTimeStamp();

        if (gateName == "Cổng vào") {
            addUID(uid);
            Serial.println("Thẻ được thêm vào danh sách tại Cổng vào");

            logData += "<tr><td>" + gateName + "</td><td>" + action + "</td><td>" + entryTime + "</td></tr>";

            displayMessage("-- OPEN --"); // ngược chiều
            moveServo(servo, 5, 5000); // mở 
            moveServo(servo, 85, 0); // đóng
            displayMessage("-- CLOSE --");
        } 
        else if (gateName == "Cổng ra") {
            if (isUIDAllowed(uid)) {
                removeUID(uid);
                Serial.println("Thẻ hợp lệ, đã xóa khỏi danh sách");

                logData += "<tr><td>" + gateName + "</td><td>" + action + "</td><td>" + entryTime + "</td></tr>";

                displayMessage("-- OPEN --"); // cùng chiều 
                moveServo(servo, 85, 5000); // mở 85
                moveServo(servo, 3, 0); //  đóng 3
                displayMessage("-- CLOSE --");
            } else {
                Serial.println("!!!ERROR CARD!!!");
                displayMessage("!!!ERROR CARD!!!");
                delay(2000);
                updateLCD();
            }
        }

        rfid.PICC_HaltA();
        rfid.PCD_StopCrypto1();
    }
}
```

### ⚖️ Kiểm tra cảm biến trọng lượng  

```cpp
void checkWeightSensors() {
    // Cảm biến 1
    if (scale.is_ready()) {
        float weight1 = scale.get_units(10); // Trung bình 10 lần đo
        if (abs(weight1) < 2) weight1 = 0; // Ngưỡng nhiễu

        Serial.print("Khối lượng cảm biến 1: ");
        Serial.print(weight1);
        Serial.println(" g");

        bool isOccupied1 = weight1 > 5;
        if (isOccupied1 && !wasOccupied1) {
            if (availableSpots > 0) {
                availableSpots--;
                updateLCD();
            }
        } else if (!isOccupied1 && wasOccupied1) {
            if (availableSpots < 4) {
                availableSpots++;
                updateLCD();
            }
        }
        wasOccupied1 = isOccupied1;
    } else {
        Serial.println("Cảm biến 1 chưa sẵn sàng!");
    }

    // Cảm biến 2
    if (scale2.is_ready()) {
        float weight2 = scale2.get_units(10);
        if (abs(weight2) < 2) weight2 = 0;

        Serial.print("Khối lượng cảm biến 2: ");
        Serial.print(weight2);
        Serial.println(" g");

        bool isOccupied2 = weight2 > 5;
        if (isOccupied2 && !wasOccupied2) {
            if (availableSpots > 0) {
                availableSpots--;
                updateLCD();
            }
        } else if (!isOccupied2 && wasOccupied2) {
            if (availableSpots < 4) {
                availableSpots++;
                updateLCD();
            }
        }
        wasOccupied2 = isOccupied2;
    } else {
        Serial.println("Cảm biến 2 chưa sẵn sàng!");
    }

    // Cảm biến 3
    if (scale3.is_ready()) {
        float weight3 = scale3.get_units(10);
        if (abs(weight3) < 2) weight3 = 0;

        Serial.print("Khối lượng cảm biến 3: ");
        Serial.print(weight3);
        Serial.println(" g");

        bool isOccupied3 = weight3 > 6; 
        if (isOccupied3 && !wasOccupied3) {
            if (availableSpots > 0) {
                availableSpots--;
                updateLCD();
            }
        } else if (!isOccupied3 && wasOccupied3) {
            if (availableSpots < 4) {
                availableSpots++;
                updateLCD();
            }
        }
        wasOccupied3 = isOccupied3;
    } else {
        Serial.println("Cảm biến 3 chưa sẵn sàng!");
    }
}
```
### 🚦 Xác định xe và điều khiển LED

```cpp

void checkForVehicle() {
    int irState = digitalRead(IR_SENSOR_PIN); 
    distance = measureDistance();             
    bool isVehicleDetected = (irState == LOW && distance >= 400 && distance <= 1000);
    bool allScalesOccupied = wasOccupied1 && wasOccupied2 && wasOccupied3;
    if (isVehicleDetected && allScalesOccupied && !ledIsOn) {
        digitalWrite(LED_PIN, HIGH); 
        ledIsOn = true;             
        Serial.println("Phát hiện xe và cả ba cảm biến trọng lượng có vật! LED sáng liên tục.");
        if (availableSpots > 0) {
            availableSpots--;
            updateLCD(); 
        }
    }
    if (actualAvailableSpots < availableSpots) {  
        availableSpots = actualAvailableSpots;
        updateLCD(); 
        if (ledIsOn) {
            digitalWrite(LED_PIN, LOW); 
            ledIsOn = false;            
            Serial.println("Số chỗ trống giảm, LED tắt.");
        }
    }
}

```

### 🌐 Giao diện Web Server

ESP32 cung cấp một giao diện Web để quản lý trạng thái bãi đỗ:

🚗 Trang chính: Hiển thị số lượng vị trí trống trong bãi xe với giao diện trực quan.

📜 Lịch sử xe vào/ra: Xem log các phương tiện đã quét thẻ vào/ra bãi.

🔓 Mở/đóng cổng: Nhấn nút trên giao diện để điều khiển servo.

📶 Kết nối trạng thái: Hiển thị tín hiệu WiFi và thời gian thực.

📊 Cập nhật dữ liệu: Dữ liệu cảm biến và trạng thái bãi xe được cập nhật liên tục

### 📷 Hình ảnh thực tế

![image](https://github.com/user-attachments/assets/1cb388e5-ed3c-4c19-81ef-7091cb498b2b)


### 🏆 Kết quả & Đánh giá

✅ Hệ thống nhận diện vị trí trống chính xác ...%.

✅ Giao diện web giúp giám sát dễ dàng từ xa.

✅ Cổng tự động mở/đóng ổn định và nhanh chóng.

🔧 Cần cải thiện khả năng lọc nhiễu của cảm biến siêu âm.


💡 Nếu bạn thích dự án này, hãy ⭐️ trên GitHub nhé!

📩 Liên hệ: trannguyentuankhanh0612@gmail.com | 

📌 Tác giả: 🚀 Nhom 1 - PTIT! 🚀

