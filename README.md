# 🛡️ Hệ thống giám sát an ninh mạng lai NIDS/HIDS

> **README tổng quát cho đề tài triển khai giám sát kết hợp Wazuh + Suricata + pfSense + Machine Learning**  
> Tài liệu này được viết theo hướng **dễ đọc, dễ trình bày trong báo cáo và bám sát code hiện có**.

---

## ✨ Tổng quan nhanh

| Thành phần | Công nghệ | Vai trò |
|---|---|---|
| **Host-based** | **Wazuh** | Giám sát sự kiện trên host, hỗ trợ correlation |
| **Network-based** | **Suricata + pfSense** | Thu thập lưu lượng, phát hiện theo rule/signature |
| **Machine Learning** | **XGBoost + VAE + Isolation Forest** | Phân loại tấn công đã biết và phát hiện bất thường |
| **Response** | **Block / Alert / Webhook / Email / Wazuh** | Tự động phản ứng khi phát hiện nguy cơ |

### 🎯 Điểm nhấn của repo

- Kết hợp **giám sát host + giám sát mạng** trong cùng một pipeline
- Hỗ trợ **phát hiện đa lớp**: signature-based + supervised ML + unsupervised ML
- Có sẵn **cơ chế phản ứng tự động**
- Có nền tảng để **correlate HIDS + NIDS** phục vụ thực nghiệm và báo cáo đề tài

---

## 🧭 Mục lục nhanh

1. [Giới thiệu](#-giới-thiệu)
2. [Mục tiêu hệ thống](#-mục-tiêu-hệ-thống)
3. [Cấu trúc workspace](#-cấu-trúc-workspace)
4. [Kiến trúc code hiện có](#-kiến-trúc-code-hiện-có)
5. [Yêu cầu môi trường](#-yêu-cầu-môi-trường)
6. [Cài đặt](#-cài-đặt)
7. [Cách sử dụng](#-cách-sử-dụng)
8. [Tích hợp Wazuh](#-tích-hợp-wazuh)
9. [Thực nghiệm theo yêu cầu đề tài](#-thực-nghiệm-theo-yêu-cầu-đề-tài)
10. [Đối chiếu với yêu cầu nghiên cứu](#-đối-chiếu-với-yêu-cầu-nghiên-cứu)
11. [Mức độ hoàn thiện](#-mức-độ-hoàn-thiện)
12. [Quy trình đề xuất cho báo cáo và thực nghiệm](#-quy-trình-đề-xuất-cho-báo-cáo-và-thực-nghiệm)
13. [Tài liệu nên đọc tiếp](#-tài-liệu-nên-đọc-tiếp)
14. [Kết luận](#-kết-luận)

---

## 📌 Giới thiệu

Đây là mã nguồn cho một hệ thống giám sát an ninh mạng lai, kết hợp:

- **Host-based monitoring** qua **Wazuh**
- **Network-based monitoring** qua **Suricata + pfSense**
- **Machine Learning** để phát hiện bất thường và phân loại tấn công
- **Cơ chế phản ứng tự động** như log, cảnh báo, chặn IP, webhook, email và đẩy sự kiện sang Wazuh

Workspace hiện tại gồm hai lớp tài liệu chính:

- `readme.txt`: hướng dẫn cài đặt/ngắn gọn ban đầu
- `Source/`: mã nguồn, test, notebook và tài liệu kỹ thuật chi tiết

> ✅ README này được dùng như **bản tổng hợp chính thức ở mức workspace**, giúp bạn:
> - đọc nhanh toàn bộ dự án
> - hiểu phần nào đã triển khai thật trong code
> - đối chiếu được với yêu cầu nghiên cứu/thực nghiệm của đề tài

---

## 🎯 Mục tiêu hệ thống

Hệ thống được xây dựng để:

1. Thu thập lưu lượng mạng từ **pfSense**.
2. Trích xuất flow từ PCAP bằng **CICFlowMeter**.
3. Phát hiện tấn công bằng nhiều lớp:
   - **Suricata**: phát hiện theo signature/rule
   - **XGBoost**: phân loại tấn công đã biết
   - **VAE / Isolation Forest**: phát hiện bất thường, hỗ trợ nhận diện mẫu chưa biết
4. Kích hoạt phản ứng tự động:
   - ghi log
   - in cảnh báo
   - chặn IP trên pfSense
   - gửi webhook
   - gửi email
   - gửi hoặc correlate sự kiện với Wazuh
5. Hỗ trợ đánh giá thực nghiệm cho các tình huống tấn công trong đề tài.

### 🧠 Tư tưởng thiết kế

```text
Traffic → Capture → Flow Extraction → Multi-layer Detection → Response → Correlation
```

Hệ thống không chỉ nhằm **phát hiện**, mà còn hướng đến **ngăn chặn, ghi nhận và hỗ trợ cải tiến vòng lặp thực nghiệm**.

---

## 🗂️ Cấu trúc workspace

```text
source_code/
├── readme.txt
├── README.md                  # Tài liệu tổng quát này
└── Source/
    ├── app.py                 # Flask app mẫu nhận webhook
    ├── README.md              # README nhanh của source
    ├── requirements.txt
    ├── config/                # File cấu hình, service, Wazuh rules/decoder
    ├── data/                  # Models và dữ liệu test
    ├── docs/                  # Tài liệu kỹ thuật chi tiết
    ├── notebooks/             # Notebook huấn luyện
    ├── scripts/               # Detection pipeline và package nids/
    └── tests/                 # Script kiểm thử
```

### 📍 Những file quan trọng nhất

| File | Vai trò |
|---|---|
| `Source/scripts/detection_pipeline.py` | Entry point chạy phát hiện |
| `Source/scripts/nids/pipeline.py` | Điều phối toàn bộ pipeline |
| `Source/scripts/nids/layers.py` | Suricata, XGBoost, VAE, Isolation Forest |
| `Source/scripts/nids/actions.py` | Các hành động phản ứng |
| `Source/scripts/nids/wazuh.py` | Tích hợp và correlation với Wazuh |
| `Source/tests/test_flows_detection.py` | Test bằng file flow CSV |
| `Source/tests/test_actions.py` | Test webhook, email, Wazuh |

---

## 🏗️ Kiến trúc code hiện có

### 🔄 Luồng xử lý chính

1. pfSense capture gói tin bằng `tcpdump` qua SSH
2. File PCAP được kéo về máy chạy pipeline
3. `CICFlowMeter` chuyển PCAP thành flow CSV
4. Pipeline phân tích qua các lớp:
   - **Layer 1 - Suricata**: đọc alert từ `eve.json` trên pfSense
   - **Layer 2 - XGBoost**: nhận diện tấn công đã biết
   - **Layer 3 - VAE hoặc Isolation Forest**: phát hiện bất thường
5. Detection được lọc whitelist, gộp trùng
6. `ActionHandler` thực hiện phản ứng tương ứng
7. Nếu bật Wazuh, hệ thống có thể:
   - đẩy detection sang Wazuh
   - đọc HIDS alerts từ Wazuh
   - correlate NIDS + HIDS

### 🧩 Các action đang hỗ trợ

| Action | Mục đích |
|---|---|
| `log` | Ghi log kết quả phát hiện |
| `alert` | In cảnh báo ra console |
| `block` | Chặn IP trên pfSense |
| `webhook` | Gửi cảnh báo sang hệ thống ngoài |
| `email` | Gửi email cảnh báo |
| `wazuh` | Đẩy detection sang Wazuh |

### 🤖 Các mô hình phát hiện đang có

| Thành phần | Kiểu phát hiện | Vai trò |
|---|---|---|
| **Suricata** | Rule / Signature | Phát hiện nhanh theo luật có sẵn |
| **XGBoost** | Supervised Learning | Phân loại tấn công đã biết |
| **VAE** | Unsupervised / Anomaly | Phát hiện bất thường nâng cao |
| **Isolation Forest** | Unsupervised / Anomaly | Phương án anomaly detection thay thế |

---

## 💻 Yêu cầu môi trường

### Môi trường khuyến nghị để chạy thật

- Ubuntu/Debian Linux
- Python 3.10+
- pfSense có SSH và Suricata
- Wazuh Manager/Agent nếu muốn HIDS correlation
- CICFlowMeter
- GPU là tùy chọn, chủ yếu hữu ích khi huấn luyện VAE

### Lưu ý khi làm việc trên Windows

Bạn đang mở workspace trên Windows, nhưng phần runtime của hệ thống được thiết kế chủ yếu cho môi trường Linux/pfSense. Vì vậy:

- có thể **đọc code, sửa code, xem tài liệu** trên Windows
- có thể chạy một số test offline nếu cài đủ Python packages
- nhưng phần **capture thực tế, block IP trên pfSense, Wazuh socket/log, service production** phù hợp hơn khi triển khai trên Linux

> ⚠️ Nếu mục tiêu là **chạy demo thật**, nên triển khai trên Ubuntu/Debian.  
> Nếu mục tiêu là **đọc code, chỉnh sửa, viết báo cáo**, môi trường Windows vẫn ổn.

---

## ⚙️ Cài đặt

### 1) Di chuyển vào thư mục mã nguồn

```bash
cd Source
```

### 2) Tạo môi trường ảo và cài thư viện

```bash
python -m venv .venv
source .venv/bin/activate
pip install -r requirements.txt
pip install cicflowmeter
```

### 3) Nếu dùng Windows PowerShell

```powershell
python -m venv .venv
.\.venv\Scripts\Activate.ps1
pip install -r requirements.txt
pip install cicflowmeter
```

### 4) Sao chép file cấu hình môi trường

```bash
cp config/action_config.env.example .env
```

### 5) Các biến thường cần cấu hình

- `PFSENSE_HOST`
- `PFSENSE_USER`
- `PFSENSE_INTERFACE`
- `WEBHOOK_URL`
- `EMAIL_*`
- `WAZUH_API_URL`
- `WAZUH_API_USER`
- `WAZUH_API_PASSWORD`

---

## 🚀 Cách sử dụng

### 1. Phân tích một file PCAP

```bash
python scripts/detection_pipeline.py --pcap data/sample.pcap --action alert --action log
```

### 2. Giám sát liên tục từ pfSense

```bash
python scripts/detection_pipeline.py --continuous --interval 15 --action alert --action log
```

### 3. Giám sát liên tục và tự chặn IP

```bash
python scripts/detection_pipeline.py --continuous --interval 15 --action alert --action block
```

### 4. Bật streaming mode

```bash
python scripts/detection_pipeline.py --continuous --streaming --interval 15 --action alert
```

### 5. Gửi cảnh báo sang webhook / email / Wazuh

```bash
python scripts/detection_pipeline.py --continuous \
  --action alert \
  --action webhook \
  --action email \
  --action wazuh
```

### 6. Kiểm thử bằng file flow CSV

```bash
python tests/test_flows_detection.py --flows data/test/test_flows_20260106_153916.csv
```

### 7. Kiểm thử action

```bash
python tests/test_actions.py --test all --verbose
```

### 8. Chạy webhook receiver mẫu

```bash
python app.py
```

Mặc định Flask app mẫu nhận POST tại:

- `http://localhost:5000/webhook`

### 📝 Gợi ý sử dụng nhanh

| Mục tiêu | Lệnh nên chạy |
|---|---|
| Test pipeline offline | `python tests/test_flows_detection.py --flows ...` |
| Test action | `python tests/test_actions.py --test all --verbose` |
| Demo phát hiện online | `python scripts/detection_pipeline.py --continuous --action alert` |
| Demo phản ứng tự động | `python scripts/detection_pipeline.py --continuous --action alert --action block` |

---

## 🔗 Tích hợp Wazuh

Repo đã có sẵn nền tảng tích hợp Wazuh gồm:

- custom decoder: `Source/config/wazuh/hybrid_nids_decoder.xml`
- custom rules: `Source/config/wazuh/hybrid_nids_rules.xml`
- cấu hình integration: `Source/config/wazuh/ossec_integration.xml`
- module Python xử lý correlation: `Source/scripts/nids/wazuh.py`

### Các mode tích hợp mà code hỗ trợ

1. **Log file**
2. **Wazuh socket**
3. **Wazuh API**

### Ngoài việc đẩy alert sang Wazuh, code còn có logic

- đọc HIDS alerts từ Wazuh
- correlate với NIDS detections
- ghép tương quan theo:
  - trùng IP nguồn
  - trùng nhóm hành vi tấn công
  - chuỗi sự kiện nâng mức độ nguy hiểm

> ✅ Đây là phần rất đáng giá của repo vì nó bám sát hướng **host-based + network-based** trong đề tài.

---

## 🧪 Thực nghiệm theo yêu cầu đề tài

Theo mô tả đề tài, môi trường thực nghiệm cần triển khai:

- **Host-based**: Wazuh
- **Network-based**: Suricata + pfSense

Các kịch bản thử nghiệm mong muốn:

- quét cổng bằng **Nmap**
- tấn công **DoS**
- tấn công tầng ứng dụng
- social engineering như **phishing**, lừa người dùng cài malware hoặc chạy script
- đánh giá khả năng phát hiện và ngăn chặn
- nếu chưa chặn tốt thì điều chỉnh rules hoặc bổ sung ML rồi thử lại

### ✅ Những gì code hiện tại đã hỗ trợ tốt

#### 1) Hạ tầng giám sát lai

Code đã có đầy đủ khối chính để xây dựng hệ giám sát lai:

- lấy lưu lượng từ pfSense
- đọc Suricata alert
- phân tích flow bằng ML
- đẩy sự kiện sang Wazuh
- đọc Wazuh alert để correlation
- block IP trên pfSense

➡️ Nghĩa là phần **khung triển khai host-based + network-based** đã có nền tảng tương đối đầy đủ.

#### 2) Kịch bản Nmap / Port Scan

Code **đã hỗ trợ phát hiện port scan** theo 2 hướng:

- Suricata alert nếu pfSense/Suricata có rule tương ứng
- XGBoost nếu model đã học nhãn `PortScan` hoặc tương đương

#### 3) Kịch bản DoS / DDoS

Code **đã hỗ trợ tốt về mặt kiến trúc**:

- XGBoost có logic phân loại DoS/DDoS
- Threat level cho DoS/DDoS được đẩy lên cao
- Có thể kích hoạt `block`, `alert`, `webhook`, `email`, `wazuh`

➡️ Nếu model đã được huấn luyện đúng và pfSense/Suricata có dữ liệu đủ, đây là nhóm kịch bản có thể thực nghiệm trực tiếp.

#### 4) Kịch bản tấn công tầng ứng dụng

Code **hỗ trợ một phần**:

- tài liệu và nhãn model có nhắc tới web attacks như XSS/SQL Injection
- Suricata có thể phát hiện nếu có rule
- Wazuh correlation có thể hỗ trợ ghép nối với host events

#### 5) Kịch bản social engineering / phishing / malware chạy script

Code **mới hỗ trợ gián tiếp**, chưa có pipeline chuyên biệt đầu cuối:

- Wazuh integration có mô tả và code correlation cho các HIDS events
- có thể correlate giữa host alert và network anomaly
- có thể xem đây là nền tảng để phát hiện hậu quả của phishing/malware, ví dụ tiến trình lạ + outbound traffic bất thường

---

## 📊 Đối chiếu với yêu cầu nghiên cứu

### 🟢 Phần đã làm được trong code

1. **Kết hợp host-based và network-based**
   - Có Wazuh integration + correlation
   - Có Suricata + pfSense capture + ML pipeline

2. **Phát hiện nhiều loại tấn công mạng phổ biến**
   - Port scan
   - DoS/DDoS
   - một số web attacks nếu model/rules hỗ trợ
   - bất thường chưa biết qua VAE / Isolation Forest

3. **Cơ chế phản ứng tự động**
   - log, alert, block, webhook, email, Wazuh

4. **Chu trình phân tích lại sau detection**
   - có nền tảng để ghi log, xem alert, test lại trên CSV/PCAP
   - có notebook/model artifacts để tiếp tục tinh chỉnh ML

5. **Khả năng correlate sự kiện đa nguồn**
   - đây là phần rất phù hợp với hướng đề tài vì cho phép gộp host events và network events

### 🟡 Phần mới đạt một phần

1. **Thử nghiệm tầng ứng dụng**
   - có thể hỗ trợ về detection nếu rule/model có
   - nhưng chưa có harness hay scenario automation riêng

2. **Thử nghiệm social engineering / phishing / malware**
   - mới ở mức correlation / định hướng sử dụng Wazuh + anomaly
   - chưa có pipeline thực nghiệm trọn vẹn trong repo

3. **Vòng lặp điều chỉnh rồi thử nghiệm lại**
   - có thể làm thủ công dựa trên logs, notebook, Wazuh rules
   - chưa có workflow tự động đóng vòng `phát hiện → phân tích → tinh chỉnh → thử nghiệm lại`

### 🔴 Phần chưa thấy được hiện thực đầy đủ trong code

1. **Tự động sinh rules mới bằng Machine Learning**
   - chưa thấy module sinh Suricata/Wazuh rules tự động từ model output
   - chưa thấy cơ chế xếp hạng hoặc ưu tiên rules dựa trên feature như IP, payload patterns, port

2. **Kết hợp unsupervised + supervised để sinh rules**
   - code có cả unsupervised và supervised detection
   - nhưng chưa có tầng chuyển detection thành rule generation hoặc rule recommendation tự động

3. **Bootstrap khi thiếu log thực tế**
   - chưa thấy pipeline cho synthetic data
   - chưa thấy tích hợp threat intelligence feeds
   - chưa thấy transfer learning workflow rõ ràng trong code hiện có

4. **Tập lệnh huấn luyện đầy đủ như tài liệu mô tả**
   - tài liệu nhắc tới `scripts/train_anomaly_detector.py`
   - nhưng trong repo hiện tại **không có file này**
   - notebook huấn luyện XGBoost thì có, nhưng phần train anomaly detector bằng script chưa thấy trong source hiện tại

5. **Bộ thực nghiệm tự động cho từng kịch bản**
   - chưa có script riêng để sinh Nmap/DoS/web attack/phishing traffic một cách có kiểm soát
   - chưa có báo cáo benchmark hoặc bảng metric thực nghiệm kèm repo

### 📌 Ma trận đánh giá nhanh

| Hạng mục yêu cầu | Mức đáp ứng | Ghi chú |
|---|---|---|
| Giám sát host-based bằng Wazuh | **Tốt** | Có integration và correlation |
| Giám sát network-based bằng Suricata + pfSense | **Tốt** | Có capture, đọc alert, block IP |
| Thực nghiệm Port Scan / Nmap | **Tốt** | Phù hợp nhất với repo hiện tại |
| Thực nghiệm DoS / DDoS | **Tốt** | Có logic phát hiện và phản ứng |
| Thực nghiệm web attack | **Một phần** | Phụ thuộc rule/model/môi trường lab |
| Social engineering / phishing / malware | **Một phần** | Chủ yếu qua correlation, chưa có pipeline chuyên trách |
| ML tự sinh rules | **Chưa hoàn chỉnh** | Chưa thấy module rule generation |
| Bootstrap dữ liệu khi thiếu log | **Chưa hoàn chỉnh** | Chưa có synthetic / threat-intel / transfer learning workflow |

---

## 🔍 Mức độ hoàn thiện

Repo hiện tại **mạnh ở phần khung phát hiện và phản ứng**, cụ thể là:

- pipeline giám sát thời gian thực
- tích hợp pfSense
- đọc Suricata alert
- ML detection
- action handling
- Wazuh correlation

Tuy nhiên, nếu đối chiếu sát với yêu cầu nghiên cứu bạn mô tả, thì repo đang ở trạng thái:

- **đã có nền tảng triển khai tốt**
- **đã làm được một số ca thực nghiệm mạng phổ biến**
- **chưa hoàn thiện phần tự động cải tiến rules bằng ML**
- **chưa hiện thực đầy đủ các ca social engineering/phishing/malware dưới dạng kịch bản test trọn gói**

> **Tóm lại:**  
> Code hiện tại phù hợp để chứng minh một hệ thống giám sát lai có ML và phản ứng tự động.  
> Nhưng để khớp hoàn toàn với định hướng **“ML tự sinh rules, bootstrap dữ liệu, và vòng lặp tối ưu hóa thực nghiệm”**, vẫn cần bổ sung thêm các module nghiên cứu/phát triển.

---

## 🧭 Quy trình đề xuất cho báo cáo và thực nghiệm

### Giai đoạn 1 — Dựng hệ thống

1. Cài Python dependencies
2. Cấu hình pfSense SSH
3. Cấu hình Suricata trên pfSense
4. Cấu hình Wazuh agent/manager
5. Nạp `.env` và thử action webhook/email/Wazuh

### Giai đoạn 2 — Kiểm tra cơ bản

1. Chạy test action:

   ```bash
   python tests/test_actions.py --test all --verbose
   ```

2. Chạy test flow CSV:

   ```bash
   python tests/test_flows_detection.py --flows data/test/test_flows_20260106_153916.csv
   ```

3. Chạy detection với PCAP mẫu

### Giai đoạn 3 — Thực nghiệm tấn công

1. Thử **Nmap scan**
2. Thử **DoS/DDoS** trong lab
3. Thử một số **web attack** nếu có môi trường web đích
4. Kết hợp **Wazuh host events** để mô phỏng malware hoặc script execution

### Giai đoạn 4 — Tinh chỉnh

1. Xem log detection và Wazuh alerts
2. Điều chỉnh Suricata/Wazuh rules
3. Tái huấn luyện hoặc thay đổi threshold của model
4. Chạy lại cùng kịch bản để so sánh

---

## 📚 Tài liệu nên đọc tiếp

Trong thư mục `Source/docs/` nên ưu tiên đọc:

- `PROJECT_GUIDE.md`
- `DEPLOYMENT_GUIDE.md`
- `WAZUH_INTEGRATION.md`
- `ACTION_TRIGGERS.md`

---

## ✅ Kết luận

Nếu câu hỏi là **“code này đã làm được gì so với yêu cầu đề tài?”**, thì câu trả lời ngắn gọn là:

- **Đã làm được:** khung giám sát lai Wazuh + Suricata/pfSense, ML detection, phản ứng tự động, correlation HIDS/NIDS, hỗ trợ thực nghiệm Nmap/DoS và một phần web attack
- **Làm được một phần:** thực nghiệm tầng ứng dụng nâng cao, các ca malware/phishing qua correlation
- **Chưa làm trọn vẹn:** tự sinh rules bằng ML, bootstrap dữ liệu bằng synthetic/threat intel/transfer learning, và workflow khép kín tự động `phát hiện → phân tích → tinh chỉnh rules/ML → thử nghiệm lại`
