# Node-RED Vạn An - Weighing (VanAn_Weight)

[Tiếng Việt](#tiếng-việt) | [English](#english)

---

## Tiếng Việt

Chương trình Node-RED đọc khối lượng cân từ PLC Siemens S7-1500 qua Modbus TCP, kiểm tra kết nối PLC và Remote IO, rồi gửi dữ liệu lên MQTT.

### 1. Chương trình làm gì

| Bước | Node | Chi tiết |
|---|---|---|
| Đọc cân | `readWeight` (modbus-read) | Đọc 4 Holding Register từ địa chỉ 0, mỗi 1 giây, từ PLC `192.168.0.1:502`, Unit ID 1 |
| Xử lý | `dataWeight` (function) | Ghép 2 thanh ghi đầu thành khối lượng 32-bit, thanh ghi thứ 3 là mã lỗi (bảng mã 8101 đến 8701) |
| Kiểm tra kết nối | `ping` | Ping PLC `192.168.0.1` và Remote IO `192.168.0.2` mỗi 5 giây |
| Gửi đi | `pub-data-weight` (mqtt out) | Gửi lên broker `host.docker.internal:1883`, client id `VanAn_Weight` |

Các topic MQTT được gửi:

```
VANAN_weight/pub/weight
VANAN_weight/pub/error
VANAN_weight/pub/statusPLC
VANAN_weight/pub/statusIO
VANAN_weight/pub/timestamp
```

### 2. Cần chuẩn bị trên máy chạy

1. **Docker Desktop** (Windows) hoặc Docker Engine (Linux), đã bật và chạy.
2. **Git** (trên Windows dùng Git Bash) để tải code về.
3. **Mạng LAN tới PLC:** máy phải cùng mạng với PLC `192.168.0.1` và Remote IO `192.168.0.2`, ví dụ card mạng đặt IP `192.168.0.x`. Thử bằng lệnh `ping 192.168.0.1` trước.
4. **MQTT broker chạy trên chính máy này ở cổng 1883**, ví dụ Mosquitto. Node-RED trong Docker gọi tới máy chủ qua tên `host.docker.internal`. Nếu chưa có broker, chạy nhanh bằng Docker:
   ```bash
   docker run -d --name mosquitto --restart unless-stopped -p 1883:1883 eclipse-mosquitto:2 mosquitto -c /mosquitto-no-auth.conf
   ```
5. **Internet ở lần chạy đầu tiên**, để tải image Node-RED và cài các node bổ sung.
6. **Cổng 1881** trên máy chưa bị chương trình khác dùng.

Các node bổ sung (tự cài khi khởi động, khai báo trong `data/package.json`):

| Node | Phiên bản | Dùng để |
|---|---|---|
| `node-red-contrib-modbus` | ~5.60.2 | Đọc dữ liệu Modbus TCP từ PLC |
| `node-red-contrib-s7` | ~3.1.3 | Giao tiếp Siemens S7 (đã cài, flow hiện chưa dùng) |
| `node-red-node-ping` | ~0.3.3 | Kiểm tra kết nối PLC, Remote IO |

### 3. Cài đặt và chạy

```bash
git clone https://github.com/DatQuachM4D/<ten-repo>.git
cd <ten-repo>
docker compose up -d
```

Lần đầu sẽ mất vài phút để tải image và cài node. Xem tiến trình bằng:

```bash
docker compose logs -f
```

Khi thấy dòng `Server now running at http://127.0.0.1:1880/` thì mở trình duyệt vào **http://localhost:1881**.

Các node `debug 1`, `debug 2`, `debug 3` sẽ hiện dữ liệu cân ở thanh bên phải nếu PLC và broker kết nối được.

### 4. Cài các node bổ sung (node_modules)

Thư mục `data/node_modules` không được đưa lên Git (nằm trong `.gitignore`), nên máy mới tải về sẽ chưa có các node bổ sung. Có 3 cách cài, chọn một:

**Cách 1: Tự động (mặc định).** `entrypoint` trong `docker-compose.yml` tự chạy `npm install` mỗi lần container khởi động, nên chỉ cần:

```bash
docker compose up -d
docker compose logs -f   # chờ thấy "added ... packages" và "Server now running"
```

**Cách 2: Cài tay theo `data/package.json`** (khi cách 1 lỗi, hoặc đã bỏ `entrypoint`):

```bash
docker compose up -d
MSYS_NO_PATHCONV=1 docker compose exec VanAn_Weight sh -c "cd /data && npm install --omit=dev"
docker compose restart
```

**Cách 3: Cài từng node theo tên và phiên bản:**

```bash
MSYS_NO_PATHCONV=1 docker compose exec VanAn_Weight sh -c "cd /data && npm install node-red-contrib-modbus@~5.60.2 node-red-contrib-s7@~3.1.3 node-red-node-ping@~0.3.3"
docker compose restart
```

Kiểm tra đã cài đủ:

```bash
MSYS_NO_PATHCONV=1 docker compose exec VanAn_Weight sh -c "cd /data && npm ls --depth=0"
```

Kết quả phải có đủ 3 dòng `node-red-contrib-modbus`, `node-red-contrib-s7`, `node-red-node-ping`.

> `MSYS_NO_PATHCONV=1` chỉ cần khi chạy trong Git Bash trên Windows, để Git Bash không đổi `/data` thành đường dẫn Windows. Trên Linux, PowerShell hoặc CMD thì bỏ phần này đi.
>
> Khi cài thêm node mới qua **Manage palette** trong giao diện Node-RED, `data/package.json` tự cập nhật. Nhớ commit file này (và `data/package-lock.json`) để máy khác cài được giống hệt.

### 5. Các lệnh thường dùng

| Việc | Lệnh |
|---|---|
| Khởi động | `docker compose up -d` |
| Dừng | `docker compose down` |
| Khởi động lại | `docker compose restart` |
| Xem log | `docker compose logs -f` |
| Cập nhật Node-RED bản mới | `docker compose pull && docker compose up -d` |

Container tự chạy lại khi bật máy (`restart: unless-stopped`), miễn là Docker Desktop được bật cùng Windows.

### 6. Cấu trúc thư mục

```
.
├── docker-compose.yml   # Cấu hình container
├── README.md
└── data/                # Mount vào /data trong container
    ├── flows.json       # Toàn bộ flow
    ├── flows_cred.json  # Thông tin đăng nhập đã mã hoá
    ├── package.json     # Danh sách node bổ sung
    ├── settings.js      # Cấu hình Node-RED
    └── lib/
```

Mọi thay đổi khi bấm **Deploy** được lưu thẳng vào `data/flows.json`, nên chỉ cần commit thư mục này là sao lưu được flow.

`entrypoint` trong `docker-compose.yml` tự chạy `npm install` trong `/data` mỗi lần khởi động, để cài các node bổ sung (thư mục `node_modules` không được đưa lên Git). Nếu cài lỗi, Node-RED vẫn khởi động.

### 7. Thay đổi cấu hình cho nơi lắp đặt khác

- **IP PLC hoặc Remote IO khác:** mở Node-RED, sửa node cấu hình `S7-1500-master` (Modbus client) và hai node `ping`, rồi bấm Deploy.
- **Broker MQTT ở máy khác:** sửa node cấu hình MQTT broker `host.docker.internal` thành IP của broker.
- **Đổi cổng web:** sửa `"1881:1880"` trong `docker-compose.yml` thành `"<cổng mới>:1880"`.

### 8. Xử lý lỗi thường gặp

| Hiện tượng | Nguyên nhân và cách xử lý |
|---|---|
| Node viền đỏ, log báo `Waiting for missing types` | Chưa cài xong node bổ sung. Kiểm tra Internet rồi `docker compose restart`, hoặc cài tay: `MSYS_NO_PATHCONV=1 docker compose exec VanAn_Weight sh -c "cd /data && npm install"` rồi `docker compose restart` |
| `readWeight` báo timeout | Máy không tới được PLC `192.168.0.1:502`. Kiểm tra cáp mạng, IP card mạng, tường lửa |
| MQTT báo `disconnected` | Chưa có broker ở cổng 1883 trên máy, xem mục 2.4 |
| `port is already allocated` | Cổng 1881 đã bị dùng, đổi cổng như mục 7 |
| Mất thông tin đăng nhập trong các node | File `data/.config.runtime.json` hoặc `credentialSecret` trong `settings.js` không khớp với máy gốc |

---

## English

A Node-RED application that reads weight data from a Siemens S7-1500 PLC over Modbus TCP, monitors the PLC and Remote IO connections, and publishes the data to MQTT.

### 1. What it does

| Step | Node | Details |
|---|---|---|
| Read weight | `readWeight` (modbus-read) | Reads 4 Holding Registers from address 0, every 1 second, from PLC `192.168.0.1:502`, Unit ID 1 |
| Process | `dataWeight` (function) | Combines the first 2 registers into a 32-bit weight; the 3rd register is the error code (codes 8101 to 8701) |
| Connection check | `ping` | Pings the PLC `192.168.0.1` and Remote IO `192.168.0.2` every 5 seconds |
| Publish | `pub-data-weight` (mqtt out) | Publishes to broker `host.docker.internal:1883`, client id `VanAn_Weight` |

Published MQTT topics:

```
VANAN_weight/pub/weight
VANAN_weight/pub/error
VANAN_weight/pub/statusPLC
VANAN_weight/pub/statusIO
VANAN_weight/pub/timestamp
```

### 2. Requirements

1. **Docker Desktop** (Windows) or Docker Engine (Linux), installed and running.
2. **Git** (Git Bash on Windows) to download the code.
3. **LAN access to the PLC:** the machine must be on the same network as the PLC `192.168.0.1` and Remote IO `192.168.0.2`, for example with a network card set to `192.168.0.x`. Test with `ping 192.168.0.1` first.
4. **An MQTT broker running on this machine on port 1883**, such as Mosquitto. Node-RED inside Docker reaches the host through `host.docker.internal`. If you have no broker yet, start one with Docker:
   ```bash
   docker run -d --name mosquitto --restart unless-stopped -p 1883:1883 eclipse-mosquitto:2 mosquitto -c /mosquitto-no-auth.conf
   ```
5. **Internet access on the first start**, to pull the Node-RED image and install the extra nodes.
6. **Port 1881** must be free on the machine.

Extra nodes (installed automatically at startup, listed in `data/package.json`):

| Node | Version | Purpose |
|---|---|---|
| `node-red-contrib-modbus` | ~5.60.2 | Read Modbus TCP data from the PLC |
| `node-red-contrib-s7` | ~3.1.3 | Siemens S7 communication (installed, not used by the current flow) |
| `node-red-node-ping` | ~0.3.3 | Check PLC and Remote IO connectivity |

### 3. Install and run

```bash
git clone https://github.com/DatQuachM4D/<repo-name>.git
cd <repo-name>
docker compose up -d
```

The first start takes a few minutes to pull the image and install nodes. Follow progress with:

```bash
docker compose logs -f
```

When you see `Server now running at http://127.0.0.1:1880/`, open **http://localhost:1881** in a browser.

The `debug 1`, `debug 2` and `debug 3` nodes show the weight data in the right sidebar once the PLC and broker are reachable.

### 4. Installing the extra nodes (node_modules)

The `data/node_modules` folder is not committed to Git (it is in `.gitignore`), so a fresh clone has no extra nodes. Pick one of these 3 ways to install them:

**Option 1: Automatic (default).** The `entrypoint` in `docker-compose.yml` runs `npm install` on every container start, so you only need:

```bash
docker compose up -d
docker compose logs -f   # wait for "added ... packages" and "Server now running"
```

**Option 2: Manual install from `data/package.json`** (if option 1 failed, or the `entrypoint` was removed):

```bash
docker compose up -d
MSYS_NO_PATHCONV=1 docker compose exec VanAn_Weight sh -c "cd /data && npm install --omit=dev"
docker compose restart
```

**Option 3: Install each node by name and version:**

```bash
MSYS_NO_PATHCONV=1 docker compose exec VanAn_Weight sh -c "cd /data && npm install node-red-contrib-modbus@~5.60.2 node-red-contrib-s7@~3.1.3 node-red-node-ping@~0.3.3"
docker compose restart
```

Verify the install:

```bash
MSYS_NO_PATHCONV=1 docker compose exec VanAn_Weight sh -c "cd /data && npm ls --depth=0"
```

The output must list `node-red-contrib-modbus`, `node-red-contrib-s7` and `node-red-node-ping`.

> `MSYS_NO_PATHCONV=1` is only needed in Git Bash on Windows, to stop Git Bash from rewriting `/data` into a Windows path. Drop it on Linux, PowerShell or CMD.
>
> When you add a new node through **Manage palette** in the Node-RED editor, `data/package.json` is updated automatically. Commit it (and `data/package-lock.json`) so other machines install exactly the same nodes.

### 5. Common commands

| Task | Command |
|---|---|
| Start | `docker compose up -d` |
| Stop | `docker compose down` |
| Restart | `docker compose restart` |
| View logs | `docker compose logs -f` |
| Update Node-RED | `docker compose pull && docker compose up -d` |

The container restarts automatically after a reboot (`restart: unless-stopped`), as long as Docker Desktop starts with Windows.

### 6. Folder structure

```
.
├── docker-compose.yml   # Container configuration
├── README.md
└── data/                # Mounted to /data in the container
    ├── flows.json       # All flows
    ├── flows_cred.json  # Encrypted credentials
    ├── package.json     # Extra nodes list
    ├── settings.js      # Node-RED settings
    └── lib/
```

Every **Deploy** is saved directly to `data/flows.json`, so committing this folder backs up the flows.

The `entrypoint` in `docker-compose.yml` runs `npm install` in `/data` on every start to install the extra nodes (the `node_modules` folder is not committed to Git). Node-RED still starts if the install fails.

### 7. Adapting to another site

- **Different PLC or Remote IO IP:** open Node-RED, edit the `S7-1500-master` config node (Modbus client) and the two `ping` nodes, then Deploy.
- **MQTT broker on another machine:** change the MQTT broker config node from `host.docker.internal` to the broker's IP.
- **Different web port:** change `"1881:1880"` in `docker-compose.yml` to `"<new port>:1880"`.

### 8. Troubleshooting

| Symptom | Cause and fix |
|---|---|
| Nodes outlined in red, log shows `Waiting for missing types` | Extra nodes not installed. Check Internet and run `docker compose restart`, or install manually: `MSYS_NO_PATHCONV=1 docker compose exec VanAn_Weight sh -c "cd /data && npm install"` then `docker compose restart` |
| `readWeight` times out | The machine cannot reach the PLC `192.168.0.1:502`. Check cabling, network card IP and firewall |
| MQTT shows `disconnected` | No broker on port 1883 on this machine, see section 2.4 |
| `port is already allocated` | Port 1881 is in use, change it as in section 7 |
| Credentials in nodes are lost | `data/.config.runtime.json` or `credentialSecret` in `settings.js` does not match the original machine |
