<h2 align="center">
    <a href="https://dainam.edu.vn/vi/khoa-cong-nghe-thong-tin">
    🎓 Faculty of Information Technology (DaiNam University)
    </a>
</h2>
<h2 align="center">
   CẢM BIẾN ĐO LƯU LƯỢNG NƯỚC & CẢNH BÁO RÒ RỈ NƯỚC 
</h2>
<div align="center">
    <p align="center">
        <img src="https://github.com/user-attachments/assets/ee72b1c4-04c7-4e4b-8d7a-8cf16932804a"width="170" />
        <img src="https://github.com/user-attachments/assets/1459f5bf-7fc9-4462-996d-eb1ef7633a97"width="180" />
        <img src="https://github.com/user-attachments/assets/f081d02c-b644-4e87-a40c-fcb8383c2985"width="200" />
    </p>

[![AIoTLab](https://img.shields.io/badge/AIoTLab-green?style=for-the-badge)](https://www.facebook.com/DNUAIoTLab)
[![Faculty of Information Technology](https://img.shields.io/badge/Faculty%20of%20Information%20Technology-blue?style=for-the-badge)](https://dainam.edu.vn/vi/khoa-cong-nghe-thong-tin)
[![DaiNam University](https://img.shields.io/badge/DaiNam%20University-orange?style=for-the-badge)](https://dainam.edu.vn)

</div>

## 1. Giới thiệu  
Đây là một project môn **Lập trình IoT** sử dụng cảm biến lưu lượng nước để đo **lưu lượng tức thời (L/min)**, **tổng thể tích (Lít)**, đồng thời đưa ra **cảnh báo rò rỉ** khi lưu lượng quá thấp hoặc quá cao bất thường.  
Dữ liệu được hiển thị qua **Serial Monitor** và tổng thể tích nước sử dụng được lưu trữ trong **EEPROM** để tránh mất dữ liệu khi reset hoặc mất điện.

---

## 2. Mô tả bài toán  
Trong thực tế, việc giám sát lưu lượng nước giúp:  
- Theo dõi lượng nước tiêu thụ.  
- Phát hiện sự cố **rò rỉ nước** khi lưu lượng thấp bất thường.  
- Phát hiện sự cố **dòng chảy bất thường** khi lưu lượng cao hơn mức cho phép.  

👉 Yêu cầu:  
- Đọc xung từ cảm biến lưu lượng nước (hall sensor).  
- Tính toán lưu lượng (L/min) và tổng thể tích (L).  
- Hiển thị kết quả qua Serial Monitor.  
- Lưu dữ liệu tổng vào EEPROM.  
- Phát cảnh báo khi:  
  - **Lưu lượng > 10 L/min** → Cảnh báo "cao".  
  - **Lưu lượng < 0.5 L/min** → Cảnh báo "thấp".  

---

## 3. Công nghệ và ngôn ngữ lập trình sử dụng  
- **Phần cứng**:  
  - ESP32 (hoặc Arduino có hỗ trợ EEPROM).  
  - Cảm biến lưu lượng nước YF-S201 (hoặc tương tự).  
- **Ngôn ngữ**: C++ (Arduino IDE).  
- **Thư viện**:  
  - `EEPROM.h` để lưu trữ dữ liệu.  
  - Hàm ngắt `attachInterrupt()` để đếm xung chính xác.  
## Code
#ifndef IRAM_ATTR
#define IRAM_ATTR
#endif

#include <EEPROM.h>

volatile int pulseCount = 0;             
unsigned long prevMillis = 0;            
const float calibrationFactor = 5880.0;  

// Lưu lượng trung bình
const int averagingInterval = 60; // tính trung bình mỗi 60 giây
int pulseHistory[averagingInterval] = {0};
int historyIndex = 0;

void IRAM_ATTR pulseCounter() {
  pulseCount++;
}

void setup() {
  Serial.begin(115200);
  pinMode(27, INPUT_PULLUP);
  attachInterrupt(digitalPinToInterrupt(27), pulseCounter, RISING);
  prevMillis = millis();

  // Khởi tạo EEPROM để lưu tổng lượng
  EEPROM.begin(512);
}

void loop() {
  if (millis() - prevMillis >= 1000) { 
    detachInterrupt(digitalPinToInterrupt(27));
    int pulses = pulseCount;             
    pulseCount = 0;

    // Tính lưu lượng L/min
    float flowLperMin = (pulses * 60.0) / calibrationFactor;

    // Tính tổng thể tích (Lít)
    static float totalLitres = 0;
    totalLitres += (flowLperMin / 60.0);

    // Lưu vào lịch sử để tính trung bình
    pulseHistory[historyIndex] = pulses;
    historyIndex = (historyIndex + 1) % averagingInterval;

    // Tính lưu lượng trung bình
    float avgPulses = 0;
    for (int i = 0; i < averagingInterval; i++) avgPulses += pulseHistory[i];
    float avgFlow = (avgPulses / averagingInterval) * 60.0 / calibrationFactor;

    // Hiển thị ra Serial
    Serial.print("Xung/s: "); Serial.print(pulses);
    Serial.print(" | Luu luong: "); Serial.print(flowLperMin, 3); Serial.print(" L/min");
    Serial.print(" | Tong: "); Serial.print(totalLitres, 3); Serial.print(" L");
    Serial.print(" | Trung binh: "); Serial.print(avgFlow, 3); Serial.println(" L/min");

    // Cảnh báo lưu lượng thấp hoặc cao
    if (flowLperMin > 10.0) Serial.println("CANH BAO: Luu luong cao!");
    if (flowLperMin < 0.5) Serial.println("CANH BAO: Luu luong thap!");

    // Lưu tổng lượng vào EEPROM mỗi 1 phút
    static int saveCounter = 0;
    saveCounter++;
    if (saveCounter >= averagingInterval) {
      saveCounter = 0;
      EEPROM.put(0, totalLitres);
      EEPROM.commit();
      Serial.println("Da luu tong luong vao EEPROM");
    }

    attachInterrupt(digitalPinToInterrupt(27), pulseCounter, RISING);
    prevMillis = millis();
  }
}
---

## 4. Hướng dẫn cài đặt và sử dụng  
### ⚙️ Cài đặt  
1. Cài đặt **Arduino IDE**: [Download tại đây](https://www.arduino.cc/en/software).  
2. Cài đặt **ESP32 board** trong Arduino IDE (nếu dùng ESP32).  
   - Vào **File → Preferences**.  
   - Thêm URL vào `Additional Board Manager URLs`:  
     ```
     https://dl.espressif.com/dl/package_esp32_index.json
     ```
   - Vào **Tools → Board → Board Manager** và cài đặt **ESP32 by Espressif Systems**.  
3. Kết nối ESP32/Arduino với máy tính.  

### 🔌 Kết nối phần cứng  
- Cảm biến lưu lượng **YF-S201**:  
  - **Dây đỏ** → 5V/3.3V (tùy loại board).  
  - **Dây đen** → GND.  
  - **Dây vàng** → GPIO 27 (hoặc pin digital khác).  

### ▶️ Upload & chạy chương trình  
1. Mở code trong Arduino IDE.  
2. Chọn **Board** và **Port** đúng.  
3. Upload code lên board.  
4. Mở **Serial Monitor** (115200 baud).  

### 📊 Kết quả hiển thị  
Ví dụ Serial Monitor:  Xung/s: 1671 | Luu Luong: 17.051/min | Tong: 10.525 L | Trung binh: 10.525 L/min.
Xung/s: 1697 | Luu Luong: 17.316/min | Tong: 10.814 L | Trung binh: 10.814 L/min.
Xung/s: 1697 | Luu Luong: 17.316/min | Tong: 10.814 L | Trung binh: 10.814 L/min. 
