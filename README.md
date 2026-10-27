Dự án mã nguồn mở phát triển bởi 1mM.rt. Phiên bản hiện tại chưa phải là bản phát hành chính thức; toàn bộ nội dung trong kho lưu trữ này đóng vai trò như tài liệu kĩ thuật tham khảo về cách thức hoạt động của dự án.# Honda Winner 150 (2016) - K-Line ECU Telemetry Protocol

Dự án Reverse Engineering giao thức K-Line của ECU Keihin trên xe Honda Winner 150 (đời 2016). Tài liệu này cung cấp chi tiết về quá trình phần cứng, trình tự khởi tạo (Fast Init) và cách giải mã bảng dữ liệu trực tiếp (Live Data Table 0x11) để trích xuất các thông số RPM, TPS, ECT, IAT, MAP và điện áp ắc quy.

## 1. Thông số Giao tiếp (Communication Specs)
* **Giao thức vật lý:** K-Line (ISO 9141 / KWP2000 variant)
* **Mức điện áp Logic:** 12V (Yêu cầu IC L9637D hoặc TJA1021 để chuyển đổi mức logic xuống 3.3V/5V cho vi điều khiển).
* **Baudrate:** `10400 bps`
* **Định dạng UART:** `8 Data bits, No Parity, 1 Stop bit (8N1)`

## 2. Thuật toán Checksum
Honda sử dụng một biến thể Checksum đơn giản: `0x100 - (Sum & 0xFF)`.
(Ngoại trừ khung khởi tạo Wake-up không dùng checksum).

```cpp
uint8_t calculateChecksum(const uint8_t *data, size_t len) {
    uint16_t sum = 0;
    for (size_t i = 0; i < len; i++) {
        sum += data[i];
    }
    return (uint8_t)(0x100 - (sum & 0xFF));
}

```

## 3. Trình tự Đánh thức (Fast Initialization)

Để ECU chịu nói chuyện, bạn phải thực hiện chuỗi đánh thức phần cứng kết hợp phần mềm theo mốc thời gian cực kỳ nghiêm ngặt.

**Bước 1: Đánh thức bằng phần cứng (Kéo mass chân K-Line)**

* Kéo TX lên `HIGH` trong 300ms.
* Kéo TX xuống `LOW` trong 70ms.
* Kéo TX lên `HIGH` trong 130ms.
* *(Sau bước này, khởi tạo UART Serial ở mức 10400 baud).*

**Bước 2: Gửi khung phần mềm Wake-up**

* **Gửi đi (TX):** `FE 04 FF FF` (Không có Checksum).
* K-Line là đường truyền 1 dây, nên bạn sẽ nhận lại chính tiếng vọng (Echo) của chuỗi này.

**Bước 3: Gửi khung Khởi tạo (Init Frame)**

* **Gửi đi (TX):** `72 05 00 F0 99`
* **ECU Trả về (RX):** `02 04 00 FA`
* *Lưu ý: Nếu nhận được `02 04 00 FA`, ECU đã sẵn sàng để truy xuất dữ liệu.*

## 4. Truy xuất Dữ liệu Động (Polling Live Data) - Table 0x11

Bảng `0x11` chứa toàn bộ dữ liệu theo thời gian thực (Live Data) của xe khi đang bật khóa hoặc nổ máy.

**Lệnh yêu cầu (Request Table 0x11):**

* **Gửi đi (TX):** `72 05 71 11 07`
*(Trong đó 07 là Checksum của 72+05+71+11)*

**Khung dữ liệu ECU trả về (Response):**
Khung trả về sẽ có Header `02 xx 71 11` (với `xx` là chiều dài khung) và kết thúc bằng 1 byte Checksum. Phần Payload (Dữ liệu thực tế) dài 20 bytes (từ byte thứ 4 đến byte 23).

### Bản đồ Giải mã Payload 20-Bytes (Data Map)

| Vị trí Byte | Thông số đo đạc | Cách tính (Công thức chuyển đổi) | Đơn vị |
| :--- | :--- | :--- | :--- |
| `[0]`, `[1]` | **Vòng tua máy (RPM)** | `(Byte[0] << 8) \| Byte[1]` | RPM |
| `[2]` | **Góc bướm ga (TPS Voltage)** | `Byte[2] * (5.0 / 256.0)` | Volt |
| `[3]` | **Độ mở bướm ga (Throttle)**| `Byte[3] * 0.5` | % (Độ) |
| `[4]` | **Cảm biến Nước mát (ECT Volt)**| `Byte[4] * (5.0 / 256.0)` | Volt |
| `[5]` | **Nhiệt độ Nước mát (ECT Temp)**| `Byte[5] - 40` | °C |
| `[6]` | **Cảm biến Khí nạp (IAT Volt)**| `Byte[6] * (5.0 / 256.0)` | Volt |
| `[7]` | **Nhiệt độ Khí nạp (IAT Temp)**| `Byte[7] - 40` | °C |
| `[8]` | **Áp suất Buồng đốt (MAP Volt)**| `Byte[8] * (5.0 / 256.0)` | Volt |
| `[9]` | **MAP Raw (ADC)** | `Byte[9]` | Raw |
| `[12]` | **Điện áp Bình Ắc quy (BAT)**| `Byte[12] / 10.0` | Volt |
| `[14]`, `[15]`| **Thời gian phun (Injector)** | `(Byte[14] << 8) \| Byte[15]` | Raw |
| `[16]` | **Góc đánh lửa (Ignition)** | `Byte[16]` | Raw |
| `[17]` | **Tốc độ xe (Speed)** | `Byte[17]` | km/h |

*(Các byte còn lại: [10], [11], [13], [18], [19] chứa các thông số trạng thái mã lỗi (DTC), Mode hoặc biến dự phòng đang tiếp tục được nghiên cứu).*

## 5. Lưu ý Phần cứng (Hardware Notes)

* Việc đo dữ liệu trực tiếp trên xe đang nổ máy sẽ gặp nhiễu rất nặng từ mâm lửa và mô-bin sườn.
* **BẮT BUỘC** sử dụng mạch lọc LC (Cuộn cảm 47uH + Tụ 100uF + Tụ 104) cho đường nguồn 12V nuôi IC L9637D.
* Tuyệt đối sử dụng trở kéo (Pull-up Resistor) 1k Ohm tại chân K-Line kết nối qua một Diode M1/1N4001 để chống xung ngược.
  ⚠️ Note / Lưu ý: Mã nguồn mở này thuộc dự án của 1mM.rt (TikTok https://www.tiktok.com/@1mm.racingteam và đang trong giai đoạn phát triển (chưa phải bản phát hành chính thức). Toàn bộ tài liệu và code đi kèm được chia sẻ nhằm mục đích tham khảo kiến trúc cũng như nguyên lý vận hành của hệ thống.
  Toàn bộ nội dung trong tài liệu này được tổng hợp và tham khảo từ nhiều nguồn dữ liệu kỹ thuật khác nhau, chia sẻ nhằm mục đích nghiên cứu nguyên lý vận hành của hệ thống.
