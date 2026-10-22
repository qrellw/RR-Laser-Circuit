# Tài liệu Kỹ thuật Dự án: Thiết bị Đo nhịp thở (RR-Laser-Circuit)

Tài liệu này cung cấp các thông số kỹ thuật, nguyên lý hoạt động và hướng dẫn cấu hình phần mềm cho hệ thống thiết bị đo nhịp thở ứng dụng cảm biến quang học.

## 1. Tổng quan Nguyên lý Hoạt động
Hệ thống đo nhịp thở dựa trên phương pháp đo chuyển vị quang học (Optical Displacement Sensing):
*   **Khối Phát:** Module Laser bán dẫn (bước sóng 650nm) chiếu tia sáng vào một gương phản xạ gắn trên lồng ngực.
*   **Khối Thu:** Hai photodiode (BPW34) được bố trí thành dạng bi-cell. Sự dịch chuyển của gương theo nhịp thở làm thay đổi vị trí tia phản xạ trên bề mặt bi-cell, dẫn đến sự thay đổi tương quan dòng quang điện giữa hai kênh.
*   **Khối Xử lý Tín hiệu (Analog):** Hai mạch Transimpedance Amplifier (TIA) sử dụng Op-amp MCP6002 chuyển đổi dòng quang điện (µA) thành điện áp (Va, Vb). Tín hiệu sau đó được đệm qua mạch Buffer để ổn định trở kháng trước khi đưa vào ADC.
*   **Khối Số hóa (Digital):** Vi điều khiển ESP32-C3 thu thập dữ liệu từ ADC (ADS1115) qua giao tiếp I2C, xử lý thuật toán và truyền dữ liệu qua giao thức Bluetooth Low Energy (BLE).

## 2. Sơ đồ Chân Vi điều khiển (Pinout)
*Thiết bị sử dụng module ESP32-C3 SuperMini.*

| Chức năng | Chân GPIO | Trạng thái Logic | Ghi chú |
| :--- | :--- | :--- | :--- |
| **I2C SDA** | GPIO8 | Bidirectional | Giao tiếp dữ liệu với ADS1115 |
| **I2C SCL** | GPIO9 | Output | Xung nhịp I2C cho ADS1115 |
| **BOOST_EN** | GPIO3 | Output | HIGH: Bật nguồn Boost 5V cho Laser |
| **LASER_EN** | GPIO4 | Output | HIGH: Bật công tắc MOSFET cấp dòng cho Laser |
| **VBAT_SENSE** | GPIO0 | Analog Input | Đọc điện áp Pin qua cầu phân áp (ADC1) |

## 3. Cấu hình ADC (ADS1115) và Quy trình Lấy mẫu
*   **Địa chỉ I2C (I2C Address):** `0x48` (Chân ADDR nối GND).
*   **Độ phân giải:** 16-bit.
*   **Phân bổ Kênh (Multiplexer):** Mặc dù ADS1115 chỉ có một lõi ADC, bộ dồn kênh (MUX) nội bộ cho phép đọc tuần tự 3 kênh thông qua giao thức I2C:
    *   `AIN0` = Điện áp **Va** (Tín hiệu TIA kênh 1).
    *   `AIN1` = Điện áp **Vb** (Tín hiệu TIA kênh 2).
    *   `AIN2` = Điện áp **Vbias** (Mức nền tham chiếu thực tế của mạch Analog, thường ~0.23V).

**Quy trình lấy mẫu (Sampling Sequence):**
Để triệt tiêu nhiễu môi trường và tối ưu hóa năng lượng, phần mềm cần thực hiện chu kỳ lấy mẫu theo trình tự nghiêm ngặt sau:

1.  Đặt `BOOST_EN` = HIGH (Khởi động mạch Boost).
2.  Trễ (Delay) **10ms** để điện áp 5V ổn định.
3.  Đặt `LASER_EN` = HIGH (Laser phát sáng).
4.  Trễ **50ms** để mạch lọc RC (fc ≈ 15Hz) xác lập giá trị (Settling time).
5.  Đọc giá trị ADC tại `AIN0`, `AIN1` và `AIN2` (Lưu vào biến `Va_on`, `Vb_on`, `Vbias`).
6.  Đặt `LASER_EN` = LOW (Tắt Laser).
7.  Trễ **50ms**.
8.  Đọc giá trị ADC tại `AIN0`, `AIN1` (Lưu vào biến `Va_off`, `Vb_off`).
9.  Đặt `BOOST_EN` = LOW (Tắt mạch Boost để tiết kiệm pin).

## 4. Công thức Tính toán Tín hiệu
Mức nền thực tế (Vbias) được đọc từ `AIN2` giúp bù trừ sai số do nhiệt độ và dung sai linh kiện. Phần mềm cần thực hiện các bước tính toán sau:

1.  **Trích xuất dòng quang điện (tỉ lệ với điện áp):**
    *   $I_a = V_{a\_on} - V_{a\_off}$
    *   $I_b = V_{b\_on} - V_{b\_off}$
    *(Việc trừ mẫu sáng cho mẫu tối giúp triệt tiêu hoàn toàn mức nền Vbias và ánh sáng môi trường).*

2.  **Tính tỉ số vị trí (Position Ratio - r):**
    $r = \frac{I_a - k \cdot I_b}{I_a + k \cdot I_b}$
    *   *Trong đó:* `k` là hệ số bù sai lệch giữa hai kênh (được xác định thực nghiệm). Giá trị `r` dao động từ -1 đến 1, phản ánh chính xác vị trí dịch chuyển của tia Laser.
## 5. Quản lý Năng lượng và Cảnh báo Phần cứng
*   **Nguồn cấp:** Pin Lithium-Ion 3.7V (có mạch bảo vệ PCM).
*   **Sạc:** IC TP4056 (giới hạn dòng sạc ~120mA qua trở PROG 10kΩ).
*   **Đo Pin:** Điện áp Pin được chia áp 1:2 qua cầu trở 100kΩ trước khi vào `GPIO0`. Công thức quy đổi: `V_pin = ADC_Value * 2`. Cảnh báo pin yếu nên được thiết lập ở ngưỡng 3.5V.
*   **Cảnh báo Phần cứng:** Cổng USB trên module ESP32-C3 chỉ dùng để nạp firmware (đã ngắt kết nối nguồn 5V từ USB vào LDO nội bộ để tránh xung đột với mạch nguồn ngoài). Do đó, **không thực hiện đo nhịp thở trong quá trình đang sạc pin**.

## 6. Ghi chú Phát triển Firmware
*   **Thư viện đề xuất:** `Adafruit_ADS1X15` (đọc ADC), `NimBLE-Arduino` (giao tiếp BLE).
*   **Xử lý tín hiệu số (DSP):** Tín hiệu `r` thô chứa nhiễu tần số thấp. Cần áp dụng bộ lọc thông dải (Bandpass Filter) trong dải tần **0.05 Hz đến 1 Hz** để tách xuất tín hiệu nhịp thở. Khuyến nghị sử dụng thuật toán lọc trung bình trượt (Moving Average) hoặc bộ lọc IIR (Infinite Impulse Response).
*   **Chu kỳ lấy mẫu:** Tần số lấy mẫu khuyến nghị từ 10Hz - 20Hz để đảm bảo độ phân giải cho tín hiệu nhịp thở (0.2 - 0.3Hz) mà không gây quá tải bộ nhớ.