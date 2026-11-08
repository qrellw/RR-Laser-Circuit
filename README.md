# Tài liệu Kỹ thuật Dự án: Thiết bị Đo nhịp thở (RR-Laser-Circuit)

Tài liệu này cung cấp các thông số kỹ thuật, nguyên lý hoạt động và hướng dẫn cấu hình phần mềm cho hệ thống thiết bị đo nhịp thở ứng dụng cảm biến quang học.

## 1. Tổng quan Nguyên lý Hoạt động
Hệ thống đo nhịp thở dựa trên phương pháp đo chuyển vị quang học (Optical Displacement Sensing):
*   **Khối Phát:** Module Laser bán dẫn (bước sóng 650nm) chiếu tia sáng vào một gương phản xạ gắn trên lồng ngực.
*   **Khối Thu:** Hai photodiode (BPW34) được bố trí thành dạng bi-cell. Sự dịch chuyển của gương theo nhịp thở làm thay đổi vị trí tia phản xạ trên bề mặt bi-cell, dẫn đến sự thay đổi tương quan dòng quang điện giữa hai kênh.
*   **Khối Xử lý Tín hiệu (Analog):** Sử dụng duy nhất 1 Op-amp **MCP6004** (4 kênh). Hai kênh được cấu hình làm mạch Transimpedance Amplifier (TIA) để chuyển đổi dòng quang điện (µA) thành điện áp (Va, Vb). Hai kênh còn lại làm mạch Buffer để ổn định trở kháng trước khi đưa vào ADC.
*   **Khối Số hóa (Digital):** Vi điều khiển ESP32-C3 thu thập dữ liệu từ ADC (ADS1115) qua giao tiếp I2C, xử lý thuật toán và truyền dữ liệu qua giao thức Bluetooth Low Energy (BLE).

## 2. Sơ đồ Chân Vi điều khiển (Pinout)
*Thiết bị sử dụng module ESP32-C3 SuperMini.*

| Chức năng | Chân GPIO | Trạng thái Logic | Ghi chú |
| :--- | :--- | :--- | :--- |
| **I2C SDA** | GPIO8 | Bidirectional | Giao tiếp dữ liệu với ADS1115 |
| **I2C SCL** | GPIO9 | Output | Xung nhịp I2C cho ADS1115 |
| **LASER_EN** | GPIO4 | Output | HIGH: Bật công tắc MOSFET cấp dòng cho Laser (3.3V) |
| **VBAT_SENSE** | GPIO0 | Analog Input | Đọc điện áp Pin qua cầu phân áp 100k/100k (ADC1) |

*(Lưu ý: Thiết kế V1 đã loại bỏ hoàn toàn mạch Boost 5V. Laser được cấp trực tiếp nguồn 3.3V từ LDO. Chân `BOOST_EN` cũ đã bị xóa khỏi phần mềm).*

## 3. Cấu hình ADC (ADS1115) và Quy trình Lấy mẫu
*   **Địa chỉ I2C:** `0x48` (Chân ADDR nối GND).
*   **Độ phân giải:** 16-bit.
*   **Phân bổ Kênh (Multiplexer):**
    *   `AIN0` = Điện áp **Va** (Tín hiệu TIA kênh 1).
    *   `AIN1` = Điện áp **Vb** (Tín hiệu TIA kênh 2).
    *   `AIN2` = Điện áp **Vbias** (Mức nền tham chiếu thực tế, thường ~0.23V).

**Quy trình lấy mẫu (Sampling Sequence):**
Để triệt tiêu nhiễu môi trường và ánh sáng phòng, phần mềm cần thực hiện chu kỳ lấy mẫu theo trình tự nghiêm ngặt sau:
1.  Đặt `LASER_EN` = HIGH (Laser phát sáng).
2.  Trễ (Delay) **50ms** để mạch lọc RC (fc ≈ 15Hz) xác lập giá trị.
3.  Đọc giá trị ADC tại `AIN0`, `AIN1` và `AIN2` (Lưu vào biến `Va_on`, `Vb_on`, `Vbias`).
4.  Đặt `LASER_EN` = LOW (Tắt Laser).
5.  Trễ **50ms**.
6.  Đọc giá trị ADC tại `AIN0`, `AIN1` (Lưu vào biến `Va_off`, `Vb_off`).

## 4. Công thức Tính toán Tín hiệu
Mức nền thực tế (Vbias) được đọc từ `AIN2` giúp bù trừ sai số do nhiệt độ và dung sai linh kiện. Phần mềm cần thực hiện các bước tính toán sau:

1.  **Trích xuất dòng quang điện (tỉ lệ với điện áp):**
    *   `Ia = Va_on - Va_off`
    *   `Ib = Vb_on - Vb_off`
    *(Việc trừ mẫu sáng cho mẫu tối giúp triệt tiêu hoàn toàn mức nền Vbias và ánh sáng môi trường).*

2.  **Tính tỉ số vị trí (Position Ratio - r):**
    \[
 r = \frac{I_a - k \cdot I_b}{I_a + k \cdot I_b} 
\]
    *Trong đó:* `k` là hệ số bù sai lệch giữa hai kênh (được xác định thực nghiệm). Giá trị `r` dao động từ -1 đến 1, phản ánh chính xác vị trí dịch chuyển của tia Laser.

## 5. Quản lý Năng lượng và Cảnh báo Phần cứng
*   **Nguồn cấp:** Pin Lithium-Ion 3.7V (có mạch bảo vệ PCM, dung lượng 800mAh).
*   **Sạc:** IC TP4056 (giới hạn dòng sạc ~300mA qua trở PROG).
*   **Nguồn Analog:** LDO MCP1700-3302 (siêu sạch, cấp cho MCP6004 và ADS1115).
*   **Nguồn Digital:** LDO AP2112K-3.3 (cấp cho ESP32-C3).
*   **Đo Pin:** Điện áp Pin được chia áp 1:2 qua cầu trở 100kΩ trước khi vào GPIO0. Công thức quy đổi: `V_pin = ADC_Value * 2`. Cảnh báo pin yếu nên được thiết lập ở ngưỡng **3.5V**.
*   **Cảnh báo Phần cứng:** Cổng USB trên module ESP32-C3 chỉ dùng để nạp firmware (đã ngắt kết nối nguồn 5V từ USB vào LDO nội bộ để tránh xung đột). Do đó, **không thực hiện đo nhịp thở trong quá trình đang sạc pin**.

## 6. Ghi chú Phát triển Firmware (Software)
*   **Thư viện đề xuất:** `Adafruit_ADS1X15` (đọc ADC), `NimBLE-Arduino` (giao tiếp BLE).
*   **Xử lý tín hiệu số (DSP):** Tín hiệu `r` thô chứa nhiễu tần số thấp và các dao động chậm. Cần áp dụng bộ lọc thông dải (Bandpass Filter):
    *   Dải tần nhịp thở (RR): **0.1 Hz đến 0.4 Hz** (tương đương 6 - 24 nhịp/phút).
    *   Khuyến nghị sử dụng thuật toán lọc trung bình trượt (Moving Average) hoặc bộ lọc IIR (Butterworth).
*   **Chu kỳ lấy mẫu:** Tần số lấy mẫu khuyến nghị từ **10Hz - 20Hz** để đảm bảo độ phân giải cho tín hiệu nhịp thở mà không gây quá tải bộ nhớ.
