# VA Custody Weighbridge – Prototype cân kiểm định Vạn An

Prototype mô hình **cân kiểm định (custody weighbridge)** cho dự án Vạn An. Đầu cân gửi chuỗi ASCII qua RS485 tới **PLC Siemens S7-1500**. PLC giải mã ra số cân và cung cấp qua **Modbus TCP/IP** cho **Node-RED**. Node-RED publish giá trị lên **MQTT broker (EMQX)** để các ứng dụng khác (SCADA, dashboard, database…) sử dụng.

---

## 1. Mục tiêu

- Dựng một mô hình cân thu nhỏ để kiểm thử toàn bộ chuỗi đo, từ load cell tới MQTT.
- **Tự làm phần decode ASCII** từ đầu cân trên PLC S7-1500, không phụ thuộc vào đơn vị thứ ba.
- Xuất số cân ra MQTT theo định dạng thống nhất, sẵn sàng tích hợp với hệ thống của nhà máy.

## 2. Bối cảnh

Nhà cung cấp (NCC) đang thuê ngoài (outsource) phần đọc số đầu cân qua một đơn vị khác. Vì vậy nhóm khó chủ động khi cần sửa lỗi, thay đổi định dạng dữ liệu hoặc tích hợp. Giải pháp là nhóm **tự đọc và giải mã dữ liệu ASCII** của đầu cân bằng PLC Siemens, rồi dùng Node-RED publish kết quả lên MQTT.

## 3. Danh sách thiết bị (prototype)

| STT | Thiết bị | Model | SL | Ghi chú |
|----|----------|-------|----|---------|
| 1 | Load Cell | SQB_D 250kg | *Theo mô hình* | Loại nhỏ gọn, tải 250 kg, dùng để test mô hình |
| 2 | Đầu cân (Weighing Indicator) | D2008FA | 2 | Xuất dữ liệu cân dạng chuỗi ASCII qua cổng nối tiếp |
| 3 | Hộp nối (Junction Box) | JXHG05-8-DD | 2 | Gom tín hiệu nhiều load cell về đầu cân |
| 4 | Bộ chuyển đổi nối tiếp | PLC Siemens S7-1500 + Remote IO (RS485) | 1 | Đọc chuỗi ASCII từ đầu cân qua RS485, decode ra số cân, cung cấp dữ liệu qua Modbus TCP/IP |
| 5 | Máy tính chạy Node-RED | PC | 1 | Đọc Modbus TCP/IP từ PLC, publish số cân lên MQTT (EMQX) |

> Chi tiết BOM nằm trong `02.BOM/`, catalog thiết bị nằm trong `03.Catalog/`.

## 4. Kiến trúc hệ thống

```
┌───────────┐  mV  ┌──────────────┐  mV  ┌──────────────┐
│ Load Cell │ ───► │   Hộp nối    │ ───► │   Đầu cân    │
│ SQB_D     │      │ JXHG05-8-DD  │      │   D2008FA    │
│ 250kg     │      └──────────────┘      └──────┬───────┘
└───────────┘                                   │ RS485 (ASCII)
                                                ▼
                                  ┌──────────────────────────┐
                                  │ PLC Siemens S7-1500      │
                                  │ + Remote IO (RS485)      │
                                  │ Decode ASCII             │
                                  └────────────┬─────────────┘
                                               │ Modbus TCP/IP
                                               ▼
                                  ┌──────────────────────────┐
                                  │ PC chạy Node-RED         │
                                  └────────────┬─────────────┘
                                               │ MQTT Publish
                                               ▼
                                  ┌──────────────────────────┐
                                  │      EMQX Broker         │
                                  └────────────┬─────────────┘
                                               ▼
                                 SCADA / Dashboard / Database
```

Luồng dữ liệu:
1. Load cell SQB_D 250kg tạo tín hiệu digital RS485 theo tải trọng.
2. Hộp nối JXHG05-8-DD gom và cân bằng tín hiệu từ các load cell.
3. Đầu cân D2008FA chuyển tín hiệu thành số cân và gửi ra cổng RS485 ASCII.
4. **PLC S7-1500 decode ASCII** (tách khung, lấy dấu, giá trị, trạng thái ổn định) và ghi số cân, mã lỗi vào thanh ghi.
5. PLC đóng vai trò **Modbus TCP Server**; Node-RED đọc các thanh ghi đó qua Modbus TCP/IP.
6. Node-RED đóng gói dữ liệu thành JSON và **publish lên EMQX Broker**.

## 5. Chi tiết giao tiếp

> ⚠️ Các thông số dưới đây theo manual trong `03.Catalog/` và cấu hình thực tế.

### 5.1. Đầu cân D2008FA → PLC (RS485, ASCII)

| Thông số | Giá trị | Trạng thái |
|----------|---------|------------|
| Chuẩn cổng | RS485 | Đã xác nhận |
| Baud rate | 9600 (thường gặp) | Đã xác nhận |
| Data bits / Parity / Stop bits | 8 / None / 1 | Đã xác nhận |
| Chế độ xuất | theo lệnh | Đã xác nhận |
| Module RS485 trên Remote IO | Module truyền thông nối tiếp (PtP) | Đã xác nhận model |

Các bước decode trong PLC dự kiến:
1. Nhận chuỗi từ module RS485 vào bộ đệm.
2. Tách khung theo ký tự bắt đầu/kết thúc (ví dụ `STX`/`ETX` hoặc `CR LF`).
3. Lấy dấu (+/-), giá trị số, vị trí dấu thập phân và trạng thái ổn định (nếu có).
4. Loại bỏ khung lỗi (sai độ dài, sai checksum nếu có).
5. Ghi kết quả vào vùng dữ liệu được map ra Modbus.

### 5.2. PLC → Node-RED (Modbus TCP/IP)

PLC S7-1500 chạy khối Modbus TCP Server (`MB_SERVER`). Node-RED dùng node Modbus (ví dụ `node-red-contrib-modbus`) để đọc định kỳ.

Bảng thanh ghi **đề xuất**:

| Holding Register | Kiểu | Nội dung |
|------------------|------|----------|
| 40001–40002 | REAL (float32) | Số cân đầu cân 1 (kg) |
| 40003 | WORD | ErrorCode |

| Thông số | Giá trị đề xuất |
|----------|-----------------|
| IP PLC | *(điền)* |
| Port | 502 |
| Unit ID | 1 |
| Chu kỳ đọc | 500 ms – 1 s |

## 6. Cấu trúc thư mục

```
1.Van-An-Weight/
├── 02.BOM/                  # Danh sách vật tư, thiết bị
├── 03.Catalog/              # Catalog, datasheet, manual thiết bị (D2008FA, JXHG05, load cell…)
├── 04.Structure-System/     # Sơ đồ cấu trúc hệ thống
├── 05.Drawings/             # Bản vẽ mô hình, đấu nối
├── 06.Programs/             # Chương trình (PLC, Node-RED, MQTT)
│   └── 03.emqx/             # Cấu hình và môi trường test MQTT broker EMQX
├── 07.Documents/            # Tài liệu dự án, báo cáo
└── README.md
```

## 7. Chạy môi trường MQTT (EMQX) để test

Yêu cầu: đã cài **Docker Desktop**.

```bash
cd 06.Programs/03.emqx
docker compose up -d
```

- Dashboard EMQX: http://localhost:18083
- Cổng MQTT: `1883`

Test nhanh bằng MQTT client (ví dụ MQTTX):
- Subscribe topic `vanan/weighbridge/#`
- Publish thử một payload như mục 6 để kiểm tra.

> Nếu thư mục `03.emqx` dùng lệnh `docker run` thay cho `docker compose`, làm theo hướng dẫn trong thư mục đó.

## 8. Quy trình làm việc với Git

- Nhánh `main` chứa phiên bản ổn định và được bảo vệ, chỉ cập nhật qua Pull Request đã được duyệt.
- Mỗi tính năng làm trên một nhánh riêng, ví dụ `feature/mqtt_emqx`, `feature/ascii_decoder`.

```bash
git switch -c feature/ten-tinh-nang
git add <file hoặc thư mục>
git commit -m "Mô tả thay đổi"
git push -u origin feature/ten-tinh-nang
```

Sau đó tạo **Pull Request** vào `main` trên GitHub và chờ người phụ trách duyệt.

**Lưu ý**
- GitHub chặn file trên 100 MB. Không đẩy file cài đặt, video, file CAD quá lớn; đưa chúng vào `.gitignore` hoặc lưu ở nơi khác.
- Không đẩy mật khẩu, API key hay file `.env`.

## 9. Việc cần làm

-  Dựng môi trường test MQTT broker (EMQX)
-  Chốt model load cell cho mô hình (SQB_D 250kg)
-  Lắp mô hình: load cell, hộp nối JXHG05-8-DD, đầu cân D2008FA
-  Xác nhận cấu hình RS485 và định dạng khung ASCII của D2008FA
-  Đấu nối D2008FA với module RS485 trên Remote IO của PLC S7-1500
-  Viết chương trình PLC decode ASCII → số cân
-  Cấu hình Modbus TCP Server trên PLC và chốt bảng thanh ghi
-  Viết flow Node-RED: đọc Modbus TCP → publish MQTT
-  Publish số cân lên EMQX 
-  Hiệu chuẩn và test với quả cân chuẩn
