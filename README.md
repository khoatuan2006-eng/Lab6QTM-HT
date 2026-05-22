# HƯỚNG DẪN TỔNG THỂ & PHÂN CHIA NHIỆM VỤ LAB 6
## Đề tài: Cấu hình Hệ thống mạng VLAN, OSPF, NAT, DHCP & Security ACLs

Tài liệu này đóng vai trò là **Trung tâm Điều phối (Master Guide)** cho nhóm 4 thành viên thực hiện bài Lab 6 trên môi trường **Cisco Packet Tracer Online**. 

---

## 1. Sơ đồ Topology Chi tiết (L2/L3)

Dưới đây là sơ đồ chi tiết biểu diễn đầy đủ các cổng giao tiếp (interface), địa chỉ IP và cách phân vùng VLAN/DMZ/Internet.

```text
                                     +--------------------+
                                     |    ISP Internet    |
                                     +----------+---------+
                                                | e0/0 (DHCP/Public IP)
                                                |
                                                | e0/0 (IP Nhận từ ISP - DHCP)
                                     +----------+---------+
                                     |      Router R1     |
                                     | (Gateway/NAT Edge) |
                                     +----+----------+----+
                  e0/1 (192.168.100.1/30) |          | e0/2 (192.168.100.5/30)
                                          |          |
      +-----------------------------------+          +-----------------------------------+
      |                                                                                  |
      | e0/0 (192.168.100.2/30)                                  e0/0 (192.168.100.6/30) |
+-----+--------------+                                                             +-----+--------------+
|     Router R2      |                                                             |     Router R3      |
|   (Core Router)    | e0/2 (192.168.100.9/30)             e0/3 (192.168.100.10/30)        | (Services/Security)|
| [Router-on-Stick]  +=============================================================+ [DHCP/DMZ Gateway] |
+-----+--------------+                         (ĐƯỜNG DỰ PHÒNG - COST 100)         +-----+--------------+
      | e0/1 (Sub-interfaces)                                                            | e0/1 (192.168.1.1/24)
      |                                                                                  |
      | Trunking                                                                         | Access
      |                                                                                  |
+-----+--------------+                                                             +-----+--------------+
|     Switch SW1     | e0/0 (Trunking)                               e0/0 (Trunking) |     Switch SW3     |
|   (VTP Server)     +---------------------------------------------------------------+     (DMZ Switch)   |
+--+-------+-------+-+                                                             +-----+--------------+
   |       |       |                                                                     |
   | e1/0  | e2/0  | e3/0                                                                | eth0 (192.168.1.2/24)
   |       |       |                                                               +-----+--------------+
+--+--+ +--+--+ +--+--+                                                            |     Web Server     |
| PC1 | | PC2 | | PC3 |                                                            |     (DMZ Zone)     |
+-----+ +-----+ +-----+                                                            +--------------------+
VLAN10   VLAN20  VLAN20
(Static) (DHCP)  (DHCP)
                 
                                     +--------------------+
                                     |     Switch SW2     |
                                     |    (VTP Client)    |
                                     +----------+---------+
                                                | e3/0
                                                |
                                                | eth0 (DHCP: .40)
                                     +----------+---------+
                                     |         PC4        |
                                     |      (VLAN 30)     |
                                     +--------------------+
```

### Phân tích đường đi của dữ liệu (Traffic Flow):
- **Đường đi chính (Primary Path)**: R2 ↔ R1 ↔ R3 (Được ưu tiên nhờ điều chỉnh thông số OSPF Cost).
- **Đường đi dự phòng (Backup Path)**: R2 ↔ R3 (Chỉ kích hoạt khi đường chính qua R1 bị sự cố).

---

## 2. Bảng Phân bổ Địa chỉ IP Toàn hệ thống

| Thiết bị | Cổng (Interface) | Địa chỉ IP | Subnet Mask | Default Gateway | Ghi chú |
|---|---|---|---|---|---|
| **R1** | e0/0 | DHCP từ ISP | - | - | Cổng kết nối Internet / NAT Outside |
| | e0/1 | `192.168.100.1` | `255.255.255.252` | - | Nối sang R2 e0/0 |
| | e0/2 | `192.168.100.5` | `255.255.255.252` | - | Nối sang R3 e0/0 |
| **R2** | e0/0 | `192.168.100.2` | `255.255.255.252` | - | Nối sang R1 e0/1 |
| | e0/2 | `192.168.100.9` | `255.255.255.252` | - | Nối sang R3 e0/3 (Đường dự phòng) |
| | e0/1.10 | `192.168.10.1` | `255.255.255.0` | - | Gateway VLAN 10 |
| | e0/1.20 | `172.20.0.1` | `255.255.0.0` | - | Gateway VLAN 20 |
| | e0/1.30 | `172.30.0.1` | `255.255.0.0` | - | Gateway VLAN 30 |
| **R3** | e0/0 | `192.168.100.6` | `255.255.255.252` | - | Nối sang R1 e0/2 |
| | e0/3 | `192.168.100.10`| `255.255.255.252` | - | Nối sang R2 e0/2 (Đường dự phòng) |
| | e0/1 | `192.168.1.1` | `255.255.255.0` | - | Gateway vùng DMZ |
| **Server**| eth0 | `192.168.1.2` | `255.255.255.0` | `192.168.1.1` | Máy chủ dịch vụ Web |
| **PC1** | eth0 | `192.168.10.2` | `255.255.255.0` | `192.168.10.1` | Thiết bị VLAN 10 (IP Tĩnh) |
| **PC2** | eth0 | DHCP | `255.255.0.0` | `172.20.0.1` | Thiết bị VLAN 20 (IP Động) |
| **PC3** | eth0 | DHCP (Đặt tĩnh) | `255.255.0.0` | `172.20.0.1` | Thiết bị VLAN 20 (Nhận IP .2) |
| **PC4** | eth0 | DHCP (Bắt buộc) | `255.255.0.0` | `172.30.0.1` | Thiết bị VLAN 30 (Nhận IP `172.30.0.40`) |

---

## 3. Phân chia Nhiệm vụ Thành viên

Hệ thống được chia nhỏ thành 4 phân vùng độc lập để tránh xung đột cấu hình trực tiếp trên Packet Tracer Online:

| Thành viên | Vai trò chính | Thiết bị phụ trách | Địa chỉ IP được gán | Tài liệu hướng dẫn |
|---|---|---|---|---|
| **Thành viên 1** | **Switching Specialist** | SW1, SW2 | *Không có IP Lớp 3 (Chỉ chuyển mạch L2)* | [thanh_vien_1_switching.md](file:///d:/Lab6QTM/thanh_vien_1_switching.md) |
| **Thành viên 2** | **Core Routing Specialist** | R2 | - `e0/0` (Nối R1): `192.168.100.2/30`<br>- `e0/2` (Nối R3): `192.168.100.9/30`<br>- `e0/1.10` (Gateway VL10): `192.168.10.1/24`<br>- `e0/1.20` (Gateway VL20): `172.20.0.1/16`<br>- `e0/1.30` (Gateway VL30): `172.30.0.1/16` | [thanh_vien_2_core_routing.md](file:///d:/Lab6QTM/thanh_vien_2_core_routing.md) |
| **Thành viên 3** | **Gateway & NAT Specialist**| R1 | - `e0/0` (Nối ISP): DHCP (Public IP)<br>- `e0/1` (Nối R2): `192.168.100.1/30`<br>- `e0/2` (Nối R3): `192.168.100.5/30` | [thanh_vien_3_gateway_nat.md](file:///d:/Lab6QTM/thanh_vien_3_gateway_nat.md) |
| **Thành viên 4** | **Services & Security** | R3, SW3, Server | - **R3** `e0/0` (Nối R1): `192.168.100.6/30`<br>- **R3** `e0/3` (Nối R2): `192.168.100.10/30`<br>- **R3** `e0/1` (Gateway DMZ): `192.168.1.1/24`<br>- **Server** (Web Server): `192.168.1.2/24` | [thanh_vien_4_services_security.md](file:///d:/Lab6QTM/thanh_vien_4_services_security.md) |

### 3.1 Bản đồ Đối chiếu Nhiệm vụ Theo 12 Yêu cầu của Đề bài

Để đảm bảo không bỏ sót bất kỳ yêu cầu nào của giáo viên, dưới đây là bảng ánh xạ chi tiết 12 yêu cầu trong đề bài vào nhiệm vụ của từng thành viên:

| Số yêu cầu | Nội dung yêu cầu | Thành viên chịu trách nhiệm | Cách giải quyết kỹ thuật | Tài liệu hướng dẫn chi tiết |
|---|---|---|---|---|
| **Yêu cầu 1** | Cấu hình Hostname & MOTD Banner nhóm | **Cả 4 thành viên** | Thực hiện trên các thiết bị mình phụ trách (`SW1`, `SW2`, `R2`, `R1`, `R3`, `SW3`). | Có trong hướng dẫn của từng thành viên |
| **Yêu cầu 2** | Đặt IP (tĩnh/động) cho PC, Router, Server, Client | **Cả 4 thành viên** | Cấu hình IP tĩnh cho `PC1` & `Server`, IP động cho `PC2`, `PC3`, `PC4`, và IP cổng WAN/LAN của các Router. | [README.md (IP Table)](#2-bang-phan-bo-dia-chi-ip-toan-he-thong) |
| **Yêu cầu 3** | SW1 tạo VLAN 10,20,30. R2 chia sub-interfaces. Trunk SW1-R2 & Router-on-stick | **Thành viên 1 & 2** | - **TV 1**: Tạo VLAN 10,20,30 trên SW1 và cấu hình Trunking cổng `e0/1`. <br>- **TV 2**: Cấu hình sub-interfaces `e0/1.10`, `.20`, `.30` kèm encapsulation `dot1q` trên R2. | [TV 1 (Mục 3)](file:///d:/Lab6QTM/thanh_vien_1_switching.md)<br>[TV 2 (Mục 3)](file:///d:/Lab6QTM/thanh_vien_2_core_routing.md) |
| **Yêu cầu 4** | Định tuyến động OSPF. Ưu tiên đường R2-R1-R3, link R2-R3 dự phòng | **Thành viên 2 & 4** | Cấu hình OSPF Area 0 trên R1, R2, R3. Thiết lập **`ip ospf cost 100`** trên cả hai đầu link R2-R3 (`e0/2` của R2 và `e0/3` của R3) để ép dữ liệu đi đường R1. | [TV 2 (Mục 3)](file:///d:/Lab6QTM/thanh_vien_2_core_routing.md)<br>[TV 4 (Mục 3)](file:///d:/Lab6QTM/thanh_vien_4_services_security.md) |
| **Yêu cầu 5** | R3 làm DHCP Server cho VLAN 20,30. DHCP Relay trên R2. PC4 nhận IP `.40` cố định | **Thành viên 2 & 4** | - **TV 2**: Bật `ip helper-address 192.168.100.10` trên sub-interface R2. <br>- **TV 4**: Cấu hình DHCP Pools trên R3, tạo pool tĩnh riêng khớp MAC của PC4 để cấp đúng `.40`. | [TV 2 (Mục 3)](file:///d:/Lab6QTM/thanh_vien_2_core_routing.md)<br>[TV 4 (Mục 3)](file:///d:/Lab6QTM/thanh_vien_4_services_security.md) |
| **Yêu cầu 6** | Cấu hình NAT Overload trên R1. Đảm bảo toàn mạng ping được Internet (8.8.8.8) | **Thành viên 3** | Cấu hình `ip nat inside/outside` trên R1. Tạo ACL 10 để PAT ra cổng `e0/0`. Bật lệnh quảng bá định tuyến `default-information originate` trong OSPF. | [TV 3 (Mục 3)](file:///d:/Lab6QTM/thanh_vien_3_gateway_nat.md) |
| **Yêu cầu 7** | Cấu hình SW1-SW2 Trunking. VTP domain/password đồng bộ tự động. Tạo VLAN 40,50 | **Thành viên 1** | Cấu hình Trunking trên SW1-SW2 cổng `e0/0`. Cấu hình VTP Server (`SW1`) và Client (`SW2`). Tạo VLAN 40, 50 trên SW1 để tự động đồng bộ sang SW2. | [TV 1 (Mục 3)](file:///d:/Lab6QTM/thanh_vien_1_switching.md) |
| **Yêu cầu 8** | ACL cấm PC4 (`172.30.0.40`) truy cập vùng DMZ | **Thành viên 4** | Tạo Extended ACL `DMZ_SECURITY` trên R3, cấu hình luật: `deny ip host 172.30.0.40 192.168.1.0 0.0.0.255` và áp dụng outbound trên `e0/1`. | [TV 4 (Mục 3)](file:///d:/Lab6QTM/thanh_vien_4_services_security.md) |
| **Yêu cầu 9** | VLAN 20 chỉ được phép truy cập Web Server bằng giao thức HTTP/HTTPS | **Thành viên 4** | Thêm luật vào ACL `DMZ_SECURITY` trên R3: cho phép TCP port 80/443 từ VLAN 20 đến Server `192.168.1.2`, sau đó cấm toàn bộ giao thức IP khác. | [TV 4 (Mục 3)](file:///d:/Lab6QTM/thanh_vien_4_services_security.md) |
| **Yêu cầu 10** | R1 chỉ cho phép truy cập Telnet từ mạng VLAN 10 | **Thành viên 3** | Tạo ACL 20 trên R1 chỉ `permit 192.168.10.0 0.0.0.255`. Cấu hình cổng VTY Telnet trên R1 và áp dụng lệnh `access-class 20 in`. | [TV 3 (Mục 3)](file:///d:/Lab6QTM/thanh_vien_3_gateway_nat.md) |
| **Yêu cầu 11** | Cấm VLAN 30 truy cập Internet | **Thành viên 3** | Loại trừ dải mạng VLAN 30 (`172.30.0.0/16`) khỏi Access-List 10 dùng cho NAT Overload trên R1. | [TV 3 (Mục 3)](file:///d:/Lab6QTM/thanh_vien_3_gateway_nat.md) |
| **Yêu cầu 12** | Client từ Internet truy cập Web Server bằng Static NAT | **Thành viên 3** | Cấu hình Static Port Forwarding trên R1: Ánh xạ cổng TCP 80/443 của Server `192.168.1.2` ra cổng Public của R1. | [TV 3 (Mục 3)](file:///d:/Lab6QTM/thanh_vien_3_gateway_nat.md) |

---

## 4. Quy tắc phối hợp làm việc trên Packet Tracer Online

Làm việc trên file online đồng bộ thời gian thực rất dễ gây lỗi đè cấu hình. Toàn nhóm **bắt buộc** tuân thủ quy tắc sau:

1. **Khóa thiết bị khi cấu hình (Locking Device)**: Trước khi gõ lệnh trên một Router/Switch, hãy nhắn tin lên group chat của nhóm:
   > *"Mình đang cấu hình R2, mọi người tránh thao tác trên R2 nhé."*
2. **Giải phóng thiết bị (Unlocking Device)**: Sau khi cấu hình xong, gõ lệnh `write memory` hoặc `copy running-config startup-config` để lưu cấu hình, sau đó báo cáo:
   > *"R2 đã cấu hình xong và lưu file. Thiết bị đã sẵn sàng."*
3. **Không can thiệp chéo**: Nếu cần chỉnh sửa thiết bị của thành viên khác (ví dụ: Thành viên 4 cần bật IP Helper-address trên sub-interface của R2 thuộc Thành viên 2), hãy gửi đoạn script lệnh chính xác để thành viên đó tự dán vào, **tuyệt đối không tự cấu hình trên thiết bị của người khác**.

---

## 5. Kế hoạch Kiểm thử & Nghiệm thu Toàn hệ thống (Joint Test Plan)

Sau khi cả 4 thành viên báo cáo hoàn tất công việc của mình, thực hiện kiểm thử theo các bước sau:

### Bước 1: Kiểm tra kết nối cơ bản & Nhận DHCP
- PC2, PC3 và PC4 phải nhận được IP thành công từ DHCP Server (R3) thông qua DHCP Relay (R2).
- Xác minh PC4 nhận đúng địa chỉ IP `172.30.0.40`.

### Bước 2: Kiểm tra Định tuyến & Dự phòng (OSPF)
- Thực hiện `tracert 192.168.1.2` (IP Server) từ PC1 hoặc PC2. 
- **Kết quả mong muốn**: Dữ liệu đi theo đường R2 (`192.168.10.1` / `172.20.0.1`) -> R1 (`192.168.100.1`) -> R3 (`192.168.100.6`) -> Server.
- Giả lập sự cố: Shutdown interface `e0/1` trên R1. Thực hiện `tracert` lại.
- **Kết quả mong muốn**: Dữ liệu tự động chuyển hướng đi qua đường R2-R3 trực tiếp (`192.168.100.10`).

### Bước 3: Kiểm tra Bảo mật & Truy cập (ACLs)
- **Kiểm thử ACL 1**: Từ PC4 (`172.30.0.40`), thử ping hoặc truy cập web đến Server (`192.168.1.2`).
  - *Kết quả mong muốn*: Thất bại hoàn toàn (Destination Host Unreachable).
- **Kiểm thử ACL 2**: Từ PC2/PC3 (VLAN 20), truy cập trình duyệt web và nhập `http://192.168.1.2`.
  - *Kết quả mong muốn*: Thành công (hiển thị trang chủ Web Server).
  - Từ PC2/PC3 thử Ping hoặc FTP đến Server (`192.168.1.2`).
  - *Kết quả mong muốn*: Thất bại (Ping fail/FTP blocked).
- **Kiểm thử Telnet R1**: Từ PC1 (`192.168.10.2` - VLAN 10), mở CMD gõ `telnet 192.168.100.1`.
  - *Kết quả mong muốn*: Kết nối thành công, yêu cầu Username/Password.
  - Từ PC2 (VLAN 20) hoặc PC4 (VLAN 30), thử `telnet 192.168.100.1`.
  - *Kết quả mong muốn*: Bị chặn ngay lập tức (Connection refused).

### Bước 4: Kiểm tra NAT & Internet
- PC1, PC2 truy cập Internet (thử ping `8.8.8.8` hoặc `1.1.1.1`).
  - *Kết quả mong muốn*: Ping thành công. Trên R1, gõ `show ip nat translations` phải xuất hiện bảng ánh xạ NAT.
- PC4 (VLAN 30) thử ping `8.8.8.8`.
  - *Kết quả mong muốn*: Thất bại hoàn toàn (Do yêu cầu cấm VLAN 30 ra Internet).
- Thiết bị Client từ ngoài Internet (Subnet `200.200.200.0/24`) truy cập địa chỉ Public IP của Server.
  - *Kết quả mong muốn*: Truy cập thành công dịch vụ Web nhờ cấu hình Static NAT trên R1.

---
*Chúc cả nhóm phối hợp tốt và hoàn thành bài lab xuất sắc! Hãy click vào các link tài liệu của từng thành viên ở Mục 3 để xem hướng dẫn chi tiết của mình.*
