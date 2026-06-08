# PHẦN 1: LÝ THUYẾT DOCKER & DOCKER-COMPOSE

## 1. Docker là gì?

Docker là một nền tảng phần mềm cho phép đóng gói một ứng dụng và toàn bộ môi trường chạy của nó (mã nguồn, thư viện, cấu hình...) vào một "chiếc hộp" tiêu chuẩn được gọi là Container. Các container này hoạt động độc lập, cách ly với nhau và có thể chạy nhất quán trên bất kỳ máy tính nào có cài đặt Docker (dù là Windows, Mac, hay Linux).

---

## 2. Các keyword trong docker-compose.yml

File `docker-compose.yml` dùng để định nghĩa và chạy nhiều container cùng lúc.

### version

Khai báo phiên bản cú pháp của file (thường dùng `version: '3.8'`).

### services

Nơi định nghĩa danh sách các container sẽ chạy. Mỗi dịch vụ con bên trong là một container độc lập.

### image

Chỉ định image nguồn (từ Docker Hub) để tạo container.

Ví dụ:

```yaml
image: mariadb:10.5
```

### build

Chỉ định thư mục chứa Dockerfile để Docker tự động build thành image mới.

Ví dụ:

```yaml
build: ./flask_api
```

### ports

Ánh xạ cổng (port) từ máy chủ thực (Host) vào cổng bên trong Container. Định dạng là `"Host_Port:Container_Port"`.

Ví dụ:

```yaml
ports:
  - "8080:80"
```

(Truy cập `localhost:8080` sẽ vào cổng `80` của container).

### environment

Khai báo các biến môi trường cấu hình cho ứng dụng trong container.

Ví dụ:

```yaml
environment:
  - MYSQL_ROOT_PASSWORD=123456
```

### volumes

Gắn thư mục của máy chủ vật lý vào container để lưu trữ dữ liệu vĩnh viễn (khi xóa container dữ liệu không bị mất).

Ví dụ:

```yaml
volumes:
  - ./mysql_data:/var/lib/mysql
```

### networks

Định nghĩa mạng nội bộ để các container có thể "nhìn thấy" và giao tiếp với nhau bằng tên (`service name`).

Ví dụ:

```yaml
networks:
  - my_network
```

### depends_on

Quy định thứ tự khởi động. Container này sẽ chờ container kia khởi động xong mới chạy.

Ví dụ:

```yaml
depends_on:
  - mariadb
```

### restart

Chính sách khởi động lại khi container bị lỗi hoặc khi khởi động lại máy chủ.

Ví dụ:

```yaml
restart: always
```

hoặc

```yaml
restart: unless-stopped
```

---

## 3. Ưu điểm khi triển khai app bằng Docker

### Nhất quán (Build once, run anywhere)

Code chạy được trên máy cá nhân chắc chắn sẽ chạy được trên máy chủ thực tế mà không lo lỗi "xung đột phiên bản thư viện".

### Tiết kiệm tài nguyên

Container chia sẻ chung nhân hệ điều hành (`Kernel`) nên cực kỳ nhẹ, khởi động chỉ mất vài giây thay vì vài phút như máy ảo (`VM`) truyền thống.

### Cô lập hoàn toàn

Các ứng dụng chạy trong container độc lập nhau, nếu một app bị lỗi hoặc bị hack cũng không lây lan sang các app khác.

### Triển khai & mở rộng cực nhanh

Chỉ cần một lệnh:

```bash
docker compose up -d
```

là dựng xong cả hệ thống phức tạp.

---

## 4. Triển khai app lên máy chủ KHÔNG có Internet

Đây là bài toán thực tế rất hay gặp trong các mạng nội bộ của doanh nghiệp/ngân hàng.

### Trên máy tính cá nhân (đang có internet)

Tải/Build tất cả các image cần thiết.

Dùng lệnh nén image thành file tar:

```bash
docker save -o my_app_images.tar nginx mariadb influxdb grafana nodered/node-red python:3.9
```

### Chuyển dữ liệu

Copy file `my_app_images.tar` cùng thư mục chứa mã nguồn (`docker-compose.yml`, code Flask, HTML...) qua máy chủ thực bằng USB hoặc ổ cứng ngoài.

### Trên máy chủ offline

Load các image vào Docker:

```bash
docker load -i my_app_images.tar
```

Vào thư mục chứa file cấu hình và chạy hệ thống:

```bash
docker compose up -d
```

# PHẦN 2: THỰC HÀNH ÁP DỤNG - APP MONITOR & ALERT DATA REALTIME

Với hệ thống này, sẽ lấy dữ liệu giá tiền ảo **Bitcoin (BTC)** qua API công khai của Binance:

```text
https://api.binance.com/api/v3/ticker/price?symbol=BTCUSDT
```

Dữ liệu này nhảy liên tục từng giây, cực kỳ phù hợp để test realtime và làm biểu đồ.

# BƯỚC 0: TẠO KHÔNG GIAN LÀM VIỆC ĐỘC LẬP

Chúng ta sẽ tạo một thư mục hoàn toàn mới, tách biệt với dự án cũ (nếu có).

Gõ lần lượt 3 lệnh này:

```bash
# 1. Đi về thư mục gốc của user
cd ~

# 2. Tạo thư mục chứa dự án số 5 và các thư mục con bên trong
mkdir -p bt5_realtime/flask_api bt5_realtime/html

# 3. Chui vào thư mục dự án vừa tạo để bắt đầu làm việc
cd bt5_realtime
```
<img width="1919" height="1079" alt="Screenshot 2026-06-07 204222" src="https://github.com/user-attachments/assets/666ee2e1-6732-4abd-9b91-6545a533be96" />

# BƯỚC 1: TẠO BẢN THIẾT KẾ docker-compose.yml CHỐNG XUNG ĐỘT

Bây giờ đang đứng ở thư mục `bt5_realtime`.

Gõ lệnh sau để tạo file:

```bash
nano docker-compose.yml
```

Copy toàn bộ đoạn code dưới đây dán vào:

```yaml
version: '3.8'

services:
  mariadb:
    image: mariadb:10.5
    container_name: mariadb_bt5
    environment:
      MYSQL_ROOT_PASSWORD: root
      MYSQL_DATABASE: realtime_db
    ports:
      - "3307:3306" 

  influxdb:
    image: influxdb:1.8
    container_name: influxdb_bt5
    ports:
      - "8086:8086"
    environment:
      - INFLUXDB_DB=history_db

  grafana:
    image: grafana/grafana:latest
    container_name: grafana_bt5
    ports:
      - "3000:3000"
    depends_on:
      - influxdb

  nodered:
    image: nodered/node-red:latest
    container_name: nodered_bt5
    ports:
      - "1880:1880"
    depends_on:
      - mariadb
      - influxdb

  flask_api:
    build: ./flask_api
    container_name: flask_api_bt5
    ports:
      - "5000:5000"
    depends_on:
      - mariadb

  nginx:
    image: nginx:latest
    container_name: nginx_bt5
    ports:
      - "8080:80" 
    volumes:
      - ./html:/usr/share/nginx/html
```
<img width="1919" height="1079" alt="Screenshot 2026-06-07 204623" src="https://github.com/user-attachments/assets/a19470dd-3a8d-4cd6-9ba8-77d083dd0e10" />

(Lưu file: Bấm `Ctrl + X`, gõ `Y`, rồi `Enter`).

---

# BƯỚC 2: TẠO CODE CHO BACKEND (API)

Gõ lệnh chui vào thư mục flask:

```bash
cd flask_api
```

## 1. Tạo file cài đặt thư viện

```bash
nano requirements.txt
```
<img width="1919" height="1079" alt="Screenshot 2026-06-07 204720" src="https://github.com/user-attachments/assets/eddafa57-f501-4bbe-99b5-919a33fdd37e" />

Dán nội dung này vào (`Ctrl+X -> Y -> Enter`):

```text
Flask==2.0.1
mysql-connector-python==8.0.28
flask-cors==3.0.10
```

## 2. Tạo file Dockerfile (để đóng gói Code)

```bash
nano Dockerfile
```

Dán nội dung này vào (`Ctrl+X -> Y -> Enter`):

```dockerfile
FROM python:3.9-slim
WORKDIR /app
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt
COPY app.py .
CMD ["python", "app.py"]
```
<img width="1919" height="1079" alt="Screenshot 2026-06-07 204757" src="https://github.com/user-attachments/assets/3e2833cb-406e-4693-8cb8-767fece52c2a" />

## 3. Tạo file Code logic của API

```bash
nano app.py
```

Dán nội dung này vào (`Ctrl+X -> Y -> Enter`):

```python
from flask import Flask, jsonify
from flask_cors import CORS
import mysql.connector

app = Flask(__name__)
CORS(app)

def get_db_connection():
    return mysql.connector.connect(
        host='mariadb', user='root', password='root', database='realtime_db'
    )

@app.route('/api/current_price', methods=['GET'])
def get_current_price():
    try:
        conn = get_db_connection()
        cursor = conn.cursor(dictionary=True)
        cursor.execute("SELECT value, timestamp FROM current_data WHERE id = 1")
        data = cursor.fetchone()
        cursor.close()
        conn.close()
        if data:
            return jsonify({"status": "success", "data": data})
        return jsonify({"status": "error", "message": "Chưa có dữ liệu"}), 404
    except Exception as e:
        return jsonify({"status": "error", "message": str(e)}), 500

if __name__ == '__main__':
    app.run(host='0.0.0.0', port=5000)
```
<img width="1919" height="1079" alt="Screenshot 2026-06-07 204833" src="https://github.com/user-attachments/assets/0aa2f482-a9bb-4f76-b29f-3f71abf26018" />

---

# BƯỚC 3: TẠO GIAO DIỆN WEB HTML (Nginx)

Quay lại thư mục gốc và chui vào thư mục html:

```bash
cd ~/bt5_realtime/html
nano index.html
```

Dán nội dung này vào (`Ctrl+X -> Y -> Enter`):

```html
<!DOCTYPE html>
<html lang="vi">
<head>
    <meta charset="UTF-8">
    <title>Giám sát Giá BTC Realtime</title>
    <style>
        body { font-family: Arial, sans-serif; text-align: center; background: #2c3e50; color: white;}
        .card { background: #34495e; padding: 20px; margin: 20px auto; width: 60%; border-radius: 10px; box-shadow: 0 4px 8px rgba(0,0,0,0.3); }
        .price { font-size: 50px; font-weight: bold; color: #2ecc71; margin: 10px 0;}
    </style>
</head>
<body>
    <div class="card">
        <h2>Giá Bitcoin Hiện Tại (Đổ về từ MariaDB)</h2>
        <div class="price" id="btc_price">Loading...</div>
        <p id="time" style="color: #bdc3c7;">Cập nhật lúc: --</p>
    </div>

    <iframe id="grafana" src="" width="80%" height="450" frameborder="0" style="background: #fff; border-radius: 10px;"></iframe>

    <script>
        function getPrice() {
            // Máy chủ web gọi thẳng vào API cổng 5000
            fetch('http://' + window.location.hostname + ':5000/api/current_price')
                .then(res => res.json())
                .then(data => {
                    if(data.status === 'success' && data.data) {
                        document.getElementById('btc_price').innerText = parseFloat(data.data.value).toLocaleString() + " USD";
                        document.getElementById('time').innerText = "Cập nhật lúc: " + new Date(data.data.timestamp).toLocaleTimeString();
                    }
                }).catch(err => console.log(err));
        }
        setInterval(getPrice, 2000); // Tự động load dữ liệu mỗi 2 giây
        getPrice();
    </script>
</body>
</html>
```
<img width="1919" height="1079" alt="Screenshot 2026-06-07 204922" src="https://github.com/user-attachments/assets/65eb1cd8-e8dd-492d-88b0-d60dd2eb02df" />

---

# BƯỚC 4: KHỞI ĐỘNG HỆ THỐNG VÀ TẠO BẢNG DỮ LIỆU

Bây giờ mọi File đã sẵn sàng.

Ra lệnh nổ máy:

### Bash

```bash
cd ~/bt5_realtime
docker compose up -d --build
```
<img width="1919" height="1079" alt="Screenshot 2026-06-07 205656" src="https://github.com/user-attachments/assets/341b28df-b527-4aa6-b06a-d2d4d62fea85" />

(Quá trình này mất khoảng 2-5 phút để tải InfluxDB, Grafana, Node-RED và Build Flask API. Hãy kiên nhẫn đợi chữ Started màu xanh).

Sau khi chạy xong, phải vào MariaDB tạo bảng để chứa dữ liệu.

Gõ lệnh:

```bash
docker exec -it mariadb_bt5 mysql -uroot -proot
```

Khi màn hình hiện `MariaDB [(none)]>`, dán 4 lệnh này vào (`Enter` sau mỗi lệnh):

```sql
USE realtime_db;
CREATE TABLE current_data (id INT PRIMARY KEY, value FLOAT, timestamp DATETIME DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP);
INSERT INTO current_data (id, value) VALUES (1, 0);
exit;
```
<img width="1919" height="1079" alt="Screenshot 2026-06-07 205851" src="https://github.com/user-attachments/assets/2703cee9-e21b-4919-b130-a625e9b21b5f" />

## Kiểm tra thành quả Giai đoạn 1

Bây giờ mở trình duyệt web lên, gõ:

```text
http://<IP_MÁY_ẢO>:8080
```

Sẽ thấy giao diện web hiện lên và chưa có giá trị nào  (Vì Node-RED chưa bơm dữ liệu vào).
<img width="1919" height="1079" alt="Screenshot 2026-06-07 210108" src="https://github.com/user-attachments/assets/093755b0-86b0-4066-9f6c-3574e2efa4da" />

---

# BƯỚC 5: CẤU HÌNH NODE-RED (Não bộ tự động)

Mở trình duyệt vào:

```text
http://<IP_MÁY_ẢO>:1880
```

Bấm `Menu (3 dấu gạch góc phải) -> Manage palette -> tab Install`.

Tìm và cài 3 gói này:

- node-red-node-mysql
- node-red-contrib-influxdb
- node-red-contrib-telegrambot
<img width="1919" height="1025" alt="Screenshot 2026-06-07 210456" src="https://github.com/user-attachments/assets/cc2592bb-3a8d-4ad2-8bd8-f765dfcda6d9" />

Cài xong, tự kéo thả các Node theo luồng sau:

- Kéo node `Inject` (Cài đặt: `Repeat interval -> every 5 seconds`).
<img width="1919" height="1021" alt="Screenshot 2026-06-07 210602" src="https://github.com/user-attachments/assets/2aff5482-4f8c-4b43-8b65-ecc42dcd8748" />

- Nối sang node `HTTP Request` (Cài đặt: `Method GET`, URL: `https://api.binance.com/api/v3/ticker/price?symbol=BTCUSDT`, Return: `a parsed JSON object`).
<img width="1915" height="1020" alt="Screenshot 2026-06-07 210740" src="https://github.com/user-attachments/assets/2fb7ee29-c79f-4b2b-8fb5-ff639cb1f708" />

- Nối sang node `Function` (Lọc lấy giá), gõ code:
<img width="1919" height="1019" alt="Screenshot 2026-06-07 210840" src="https://github.com/user-attachments/assets/75755cc1-2907-43b5-a0df-632874eb459f" />

```javascript
msg.payload = parseFloat(msg.payload.price);
return msg;
```

Từ Node Function trên, kéo rẽ ra 3 đường:

## Đường 1 (Vào MariaDB)

Kéo thêm node Function, gõ code:

```javascript
msg.topic = "UPDATE current_data SET value = " + msg.payload + " WHERE id = 1;";
return msg;
```
<img width="1919" height="1023" alt="Screenshot 2026-06-07 211305" src="https://github.com/user-attachments/assets/06ea849b-7e7f-4717-a3da-f25036bc7dfb" />

Nối vào node mysql.

(Cài đặt mysql: host `mariadb`, port `3306`, user `root`, pass `root`, db `realtime_db`).
<img width="1919" height="1025" alt="Screenshot 2026-06-07 211557" src="https://github.com/user-attachments/assets/e397fca3-dc8c-4d19-84ae-f80ae93ce193" />

## Đường 2 (Vào InfluxDB)

Kéo node `influxdb out`.

(Cài đặt: Version `1.x`, URL `http://influxdb:8086`, Database `history_db`, Measurement `btc_price`).
<img width="1919" height="1026" alt="Screenshot 2026-06-07 212026" src="https://github.com/user-attachments/assets/83ea113a-1ef9-4535-9e9d-809fd897b4bf" />

## Đường 3 (Báo động Telegram)

Kéo node `Switch`, cấu hình rẽ 2 nhánh:

```text
< 60000
```

và

```text
> 65000
```
<img width="1919" height="1024" alt="Screenshot 2026-06-07 212324" src="https://github.com/user-attachments/assets/27c305d2-1f11-43d4-adde-7fbdaee28c68" />

Nối vào node `Function` (viết câu cảnh báo:

```javascript
msg.payload = "Báo động! Giá BTC: " + msg.payload
```

)
<img width="1919" height="1019" alt="Screenshot 2026-06-07 212411" src="https://github.com/user-attachments/assets/d46674eb-a979-4d41-a2b7-bf5eb31a56ec" />

Nối vào node `Telegram Sender`.
<img width="957" height="484" alt="image" src="https://github.com/user-attachments/assets/1571d92e-0587-42d3-a8c6-bbc447c41a7f" />

# Nhìn vào màn hình Web sẽ thấy có một khoảng trắng to đùng ở bên dưới. Đó chính là chỗ sẽ đặt biểu đồ lịch sử giá. Để lấp đầy khoảng trắng đó, đi đến bước cuối cùng: Cấu hình Grafana.
<img width="1919" height="1026" alt="Screenshot 2026-06-07 221654" src="https://github.com/user-attachments/assets/1254363f-2c84-437d-a6e5-38decc09c38f" />

## BƯỚC 1: ĐĂNG NHẬP GRAFANA

Mở một tab trình duyệt mới, truy cập vào cổng `3000`:

👉 `http://192.168.227.128:3000`

Đăng nhập với tài khoản mặc định:

```text
Username: admin
Password: admin
```

Nó sẽ yêu cầu đổi mật khẩu mới, có thể gõ mật khẩu mới hoặc bấm `Skip (Bỏ qua)` cũng được.
<img width="1919" height="1022" alt="Screenshot 2026-06-07 221924" src="https://github.com/user-attachments/assets/c54ea68b-e333-4cae-bac0-a5464e8f3c9c" />

---

## BƯỚC 2: KẾT NỐI VỚI INFLUXDB (Nơi chứa dữ liệu lịch sử)

Ở thanh menu bên trái, tìm biểu tượng bánh răng (`Configuration`) -> Chọn `Data Sources`.

Bấm nút màu xanh `Add data source`.

Tìm và chọn `InfluxDB`.

Ở bảng cấu hình hiện ra, điền 2 thông tin cực kỳ quan trọng sau (các ô khác bỏ trống):

```text
URL: http://influxdb:8086
```

(Bắt buộc dùng tên service `influxdb`, không dùng `localhost`).

Cuộn xuống dưới cùng tìm mục `Database`, gõ vào:

```text
history_db
```
<img width="1919" height="1021" alt="Screenshot 2026-06-07 222142" src="https://github.com/user-attachments/assets/f30d188f-95dc-4045-be6b-ad72e055a43b" />

Bấm nút `Save & test`.

Nếu hiện thông báo màu xanh lá cây:

```text
Data source is working
```

là thành công.

---

## BƯỚC 3: VẼ BIỂU ĐỒ

Nhìn menu bên trái, bấm vào biểu tượng 4 ô vuông (`Dashboards`) -> Chọn `New dashboard`.

Bấm nút `Add a new panel`.

Ở khu vực nửa dưới màn hình (phần `Query - chữ A`), cấu hình như sau để lấy dữ liệu:

- Chỗ `FROM`, bấm vào `select measurement` và chọn `btc_price`.
- Chỗ `SELECT`, bấm vào `field(value)` và đảm bảo nó đang chọn `mean()`.

(Ngay lúc này, ở nửa trên màn hình sẽ thấy đường line biểu đồ giá BTC bắt đầu xuất hiện).

Nhìn sang cột bên phải, chỗ `Panel title`, đặt tên là:

```text
Biểu đồ lịch sử giá BTC
```

Bấm nút `Apply` (góc trên bên phải).

Sau đó bấm tiếp biểu tượng cái đĩa mềm (`Save dashboard`) để lưu lại.
<img width="1919" height="1024" alt="Screenshot 2026-06-07 222556" src="https://github.com/user-attachments/assets/c4c3314f-eb1a-46a7-9926-b9aefbf672b0" />

---

## BƯỚC 4: NHÚNG BIỂU ĐỒ VÀO WEB TRẮNG

Ngay tại cái biểu đồ vừa lưu, bấm vào biểu tượng `3 dấu chấm` ở góc phải của cái biểu đồ đó -> Chọn `Share`.

Chuyển sang tab `Embed`.

Sẽ thấy một đoạn code HTML bắt đầu bằng:

```html
<iframe ...>
```

Bấm nút `Copy` đoạn code đó.
<img width="1919" height="1024" alt="Screenshot 2026-06-07 223458" src="https://github.com/user-attachments/assets/6318c544-ede3-4d71-bdc0-c5710223711b" />

Mở Terminal Ubuntu lên, vào sửa file HTML:

```bash
nano ~/bt5_realtime/html/index.html
```

Tìm đến chỗ thẻ:

```html
<iframe id="grafana"...>
```

cũ mà để trống, xóa nó đi và dán (`Paste`) đoạn code vừa copy từ Grafana vào đó.

Lưu file lại:

```text
Ctrl + X -> Y -> Enter
```

Bây giờ ra ngoài trình duyệt, `F5` lại trang web `8080`.
<img width="1919" height="1024" alt="Screenshot 2026-06-07 235537" src="https://github.com/user-attachments/assets/6cae3a78-90f5-4f60-add6-6471520ba8b9" />

## TEST THÔNG BÁO
<img width="1919" height="1079" alt="Screenshot 2026-06-07 232534" src="https://github.com/user-attachments/assets/5dedf7f0-cc9c-40a6-96be-df46c17fc824" />

# PUBLIC WEB
<img width="1918" height="1019" alt="Screenshot 2026-06-08 001733" src="https://github.com/user-attachments/assets/8fe20582-8895-4946-972d-4ffe4ee99378" />
<img width="1919" height="1028" alt="Screenshot 2026-06-08 002039" src="https://github.com/user-attachments/assets/3c66d26e-a717-432a-b6a4-f5f20086a18b" />

# Xuất tất cả các container ra file nén.
# xoá mọi container đang chạy
# load lại các container từ file nén để khôi phục các container đã xoá

## BƯỚC 1: XUẤT TẤT CẢ CÁC CONTAINER RA FILE NÉN (.tar.gz)

Thay vì ngồi gõ tay từng container một rất mất thời gian, chúng ta sẽ dùng một vòng lặp tự động quét qua toàn bộ các container đang chạy trong bài BT5 (Node-RED, MariaDB, Flask, Grafana, Nginx, InfluxDB) để tự động đóng gói và nén lại thành file `.tar.gz`.

Copy nguyên cụm lệnh này dán vào Terminal rồi ấn Enter:

```bash
# 1. Tạo một thư mục riêng để chứa các file sao lưu
mkdir -p ~/backup_docker && cd ~/backup_docker

# 2. Chạy vòng lặp tự động đóng gói toàn bộ container
for container in $(docker ps --format "{{.Names}}"); do
    echo "--------------------------------------------------"
    echo "Đang đóng gói container: $container ..."
    
    # Commit container hiện tại thành một Image tạm thời
    docker commit $container ${container}_backup
    
    # Xuất Image đó ra file và nén lại thành định dạng .tar.gz
    docker save ${container}_backup | gzip > ${container}.tar.gz
    
    # Xóa cái image tạm thời đi cho đỡ chật ổ cứng
    docker rmi ${container}_backup
done

# 3. Kiểm tra lại danh sách các file nén đã xuất ra
echo "=== DANH SÁCH FILE NÉN CONTAINER ĐÃ XUẤT THÀNH CÔNG ==="
ls -lh ~/backup_docker
```

Sau khi chạy xong, sẽ thấy trong thư mục xuất hiện các file như `nodered.tar.gz`, `mariadb_bt5.tar.gz`, `grafana_bt5.tar.gz`... rất gọn gàng.
<img width="1919" height="1079" alt="Screenshot 2026-06-08 002321" src="https://github.com/user-attachments/assets/c0758211-8720-490c-abb0-2d8f3a8edd82" />

---

## BƯỚC 2: XÓA SẠCH MỌI CONTAINER ĐANG CHẠY

Bây giờ là bước "thử thách lòng dũng cảm" theo đúng đề bài: Xóa sạch sẽ không chừa một cái container nào để chứng minh file nén ở Bước 1 có khả năng khôi phục.

```bash
# Xóa cưỡng chế toàn bộ container đang tồn tại trên hệ thống
docker rm -f $(docker ps -aq)
```
<img width="1917" height="292" alt="Screenshot 2026-06-08 002441" src="https://github.com/user-attachments/assets/675b7523-3189-46d4-81a0-98c385d601b9" />

Để chắc chắn hệ thống đã trống trơn, gõ lệnh kiểm tra:

```bash
docker ps -a
```
<img width="1915" height="154" alt="Screenshot 2026-06-08 002516" src="https://github.com/user-attachments/assets/039203a2-0636-4228-9145-488ffd1c35d6" />

(Màn hình không hiện ra dòng nào nữa tức là đã xóa sạch thành công).

---

## BƯỚC 3: LOAD LẠI CÁC CONTAINER TỪ FILE NÉN ĐỂ KHÔI PHỤC

Bây giờ, chúng ta sẽ dùng lệnh `docker load` (đúng chuẩn từ khóa "load lại") để nạp các file nén `.tar.gz` quay trở lại hệ thống thành các Image sẵn sàng khởi chạy.

```bash
cd ~/backup_docker

# Chạy vòng lặp giải nén và load lại tất cả các file .tar.gz
for file in *.tar.gz; do
    echo "--------------------------------------------------"
    echo "Đang tiến hành LOAD (khôi phục) file: $file ..."
    docker load -i $file
done

# Kiểm tra xem các Image đã quay trở lại kho lưu trữ chưa
echo "=== DANH SÁCH IMAGE ĐÃ ĐƯỢC LOAD LẠI THÀNH CÔNG ==="
docker images | grep backup
```
<img width="1919" height="1079" alt="Screenshot 2026-06-08 002609" src="https://github.com/user-attachments/assets/d9a72de8-b178-4817-b61d-1195541af600" />

Tuy nhiên, nhìn vào cái bảng thì thấy toàn bộ "bộ sậu" 6 cái Image (Flask, Grafana, Node-RED, InfluxDB, MariaDB, Nginx) đều đã được khôi phục đầy đủ.

Nếu muốn dựng lại toàn bộ hệ thống đồ sộ này chạy lại như cũ, chỉ cần về lại thư mục gốc và gọi bản vẽ ra:

```bash
cd ~/bt5_realtime
docker compose up -d
```
<img width="1919" height="361" alt="Screenshot 2026-06-08 003235" src="https://github.com/user-attachments/assets/722ceb71-a190-4914-97ae-cc2c6087e299" />

Lúc này Docker sẽ tự động lấy các Image vừa load lên để lắp ráp lại thành hệ thống hoàn chỉnh mà không cần tải lại từ mạng!
<img width="1140" height="572" alt="image" src="https://github.com/user-attachments/assets/b0a00c35-8aa8-487e-94e1-86971e2fd6b5" />
