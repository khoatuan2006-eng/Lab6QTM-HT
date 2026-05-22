# HƯỚNG DẪN CẤU HÌNH - THÀNH VIÊN 4
## Vai trò: Services & Security Specialist (Chuyên gia Dịch vụ & Bảo mật)
## Thiết bị phụ trách: Router `R3`, Switch `SW3`, `Web Server`

---

## 1. Sơ đồ Cổng kết nối, DHCP & Vùng DMZ (R3)

```text
                             +-------------------+
                             |     Router R1     |
                             +---------+---------+
                                       | e0/2 (192.168.100.5/30)
                                       |
                                       | e0/0 (192.168.100.6/30)
  +-------------------+      +---------+---------+
  |     Router R2     |      |     Router R3     | e0/1 (192.168.1.1/24)
  |   (Core Router)   +======+ (Services Router) +-----------+
  +-------------------+ e0/3 +---------+---------+           |
                (192.168.100.10/30)    |                     |
                  (ĐƯỜNG DỰ PHÒNG)     |                     | e0/0 (Access)
                     (Cost: 100)       |                     |
                                       |             +-------v-----------+
                                       |             |    Switch SW3     |
                                       |             +-------+-----------+
                                       |                     |
                                       |                     | e0/1 (Access)
                                       |                     |
                                       |             +-------v-----------+
                                       |             |    Web Server     |
                                       |             |   (192.168.1.2)   |
                                       |             +-------------------+
                                       |
                     [DHCP POOL FOR VLAN 20 & 30]
```

### 1.1 Bảng phân bổ Địa chỉ IP của Router R3 và vùng DMZ
Để cấu hình chính xác cho Router dịch vụ R3 và thiết lập vùng DMZ bảo mật, bạn cần nắm rõ sơ đồ cổng và địa chỉ IP cụ thể dưới đây:

| Thiết bị | Cổng (Interface) | Địa chỉ IP | Subnet Mask | Thiết bị đối diện | Ghi chú |
|---|---|---|---|---|---|
| **R3** | `e0/0` | `192.168.100.6` | `255.255.255.252` | Router R1 `e0/2` | Đường truyền chính kết nối sang R1 |
| | `e0/3` | `192.168.100.10` | `255.255.255.252` | Router R2 `e0/2` | Đường truyền dự phòng kết nối sang R2 (Cost 100) |
| | `e0/1` | `192.168.1.1` | `255.255.255.0` | Switch SW3 `e0/0` | Gateway bảo mật của vùng DMZ |
| **Server** | eth0 | `192.168.1.2` | `255.255.255.0` | Switch SW3 `e0/1` | Web Server DMZ (Đặt tĩnh tĩnh) |

---

## 2. Danh sách Công việc cần thực hiện & Đối chiếu Đề bài

Dưới đây là chi tiết công việc dịch vụ, bảo mật và cấu hình Server bạn cần thực hiện, đối chiếu chính xác theo các câu yêu cầu trong đề bài:

1. **Cấu hình thông số cơ bản** [👉 *Giải quyết **Yêu cầu 1 & 2***]: Cấu hình hostname (`R3`, `SW3`) và MOTD Banner chứa thông tin nhóm/thành viên.
2. **Cấu hình IP cho các Interface** [👉 *Giải quyết **Yêu cầu 2***]: Cấu hình địa chỉ IP cho các cổng vật lý trên R3: `e0/0` (nối sang R1) là `192.168.100.6/30`, `e0/3` (nối sang R2) là `192.168.100.10/30`, và `e0/1` (cổng Gateway DMZ) là `192.168.1.1/24`.
3. **Cấu hình định tuyến động OSPF** [👉 *Giải quyết **Yêu cầu 4***]: Bật định tuyến OSPF Process ID 1, Area 0 trên cả 3 cổng kết nối trực tiếp.
4. **Đồng bộ OSPF Cost cho đường dự phòng** [👉 *Giải quyết **Yêu cầu 4***]: Thiết lập lệnh `ip ospf cost 100` trên interface `e0/3` để đồng bộ cấu hình với Router R2 ở đầu kia, tránh định tuyến bất đối xứng.
5. **Cấu hình DHCP Server cấp IP động** [👉 *Giải quyết **Yêu cầu 5***]: Thiết lập 2 DHCP Pools (`VLAN20` và `VLAN30`) trên R3 để tự động cấp IP cho các PC thuộc VLAN 20 và VLAN 30 thông qua DHCP Relay.
6. **Cấu hình DHCP Static Binding (gán MAC cố định)** [👉 *Giải quyết **Yêu cầu 5***]: Cấu hình IP tĩnh dành riêng (`172.30.0.40/16`) trong DHCP Pool tương ứng với MAC Address của máy **PC4** để đảm bảo PC4 luôn nhận đúng địa chỉ IP này.
7. **Cấu hình Extended Access Control List (ACL)** [👉 *Giải quyết **Yêu cầu 8 & 9***]: Thiết lập bộ lọc bảo mật `DMZ_SECURITY` trên cổng `e0/1` (hướng OUT ra DMZ):
   - Cấm host **PC4 (`172.30.0.40`)** truy cập vào bất cứ tài nguyên nào thuộc DMZ (`192.168.1.0/24`) [👉 *Giải quyết **Yêu cầu 8***].
   - Chỉ cho phép VLAN 20 (`172.20.0.0/16`) truy cập Web Server (`192.168.1.2`) bằng giao thức HTTP/HTTPS (port 80/443), cấm tất cả các giao thức khác như ping, ftp, ssh [👉 *Giải quyết **Yêu cầu 9***].
8. **Cấu hình IP và Dịch vụ Web Server** [👉 *Giải quyết **Yêu cầu 2 & 9***]: Cấu hình IP tĩnh `192.168.1.2/24` cho Web Server và bật các dịch vụ HTTP/HTTPS trên Server để chạy trang Web.

---

## 3. Các lệnh cấu hình chi tiết (Cisco IOS)

### 3.1 Cấu hình trên Router R3

```ios
enable
configure terminal

! 1. Cấu hình Hostname & Banner MOTD
hostname R3
banner motd #
===================================================
*  BÀI LAB 6 - NHÓM X                              *
*  R3 - THÀNH VIÊN 4 (SERVICES & SECURITY TEAM)    *
===================================================
#

! 2. Cấu hình địa chỉ IP các interface
interface e0/0
 ip address 192.168.100.6 255.255.255.252
 no shutdown

interface e0/1
 ip address 192.168.1.1 255.255.255.0
 no shutdown

interface e0/3
 ip address 192.168.100.10 255.255.255.252
 ! Tăng cost OSPF để đồng bộ đường dự phòng với R2
 ip ospf cost 100
 no shutdown

! 3. Cấu hình OSPF
router ospf 1
 network 192.168.100.4 0.0.0.3 area 0
 network 192.168.100.8 0.0.0.3 area 0
 network 192.168.1.0 0.0.0.255 area 0
 exit

! 4. Cấu hình DHCP Server cấp IP động cho VLAN 20 & VLAN 30
! Loại trừ các IP Gateway và IP đặc biệt để tránh cấp trùng
ip dhcp excluded-address 172.20.0.1
ip dhcp excluded-address 172.30.0.1
ip dhcp excluded-address 172.30.0.40  ! Loại trừ IP của PC4 để dành riêng

! Cấu hình Pool cho VLAN 20
ip dhcp pool VLAN20
 network 172.20.0.0 255.255.0.0
 default-router 172.20.0.1
 dns-server 8.8.8.8

! Cấu hình Pool cho VLAN 30
ip dhcp pool VLAN30
 network 172.30.0.0 255.255.0.0
 default-router 172.30.0.1
 dns-server 8.8.8.8

! 5. Cấu hình DHCP Static Binding riêng cho PC4 để luôn nhận IP .40
! Lưu ý: Thay "01aa.bbcc.ddee.ff" bằng địa chỉ MAC thực tế của PC4 trong Packet Tracer.
! Số "01" ở đầu đại diện cho giao thức Ethernet, theo sau là MAC viết liền chấm giữa.
ip dhcp pool PC4_RESERVATION
 host 172.30.0.40 255.255.0.0
 client-identifier 01aa.bbcc.ddee.ff
 default-router 172.30.0.1
 dns-server 8.8.8.8
 exit

! 6. Cấu hình Extended ACL Bảo mật vùng DMZ (Đặt tên là DMZ_SECURITY)
ip access-list extended DMZ_SECURITY
 ! Rule 1: Cấm PC4 (172.30.0.40) truy cập toàn bộ vùng DMZ (192.168.1.0/24)
 deny ip host 172.30.0.40 192.168.1.0 0.0.0.255
 
 ! Rule 2: Chỉ cho phép VLAN 20 truy cập Web Server (192.168.1.2) bằng HTTP (80)
 permit tcp 172.20.0.0 0.0.255.255 host 192.168.1.2 eq 80
 ! Rule 3: Chỉ cho phép VLAN 20 truy cập Web Server bằng HTTPS (443)
 permit tcp 172.20.0.0 0.0.255.255 host 192.168.1.2 eq 443
 ! Rule 4: Cấm mọi giao thức khác (Ping, FTP, SSH...) từ VLAN 20 đến Web Server
 deny ip 172.20.0.0 0.0.255.255 host 192.168.1.2
 
 ! Rule 5: Cho phép tất cả các lưu lượng khác lưu thông bình thường (VLAN 10 truy cập Server, v.v.)
 permit ip any any
 exit

! 7. Áp dụng ACL vào interface e0/1 hướng OUT ra ngoài vùng DMZ
interface e0/1
 ip access-group DMZ_SECURITY out
 exit

write memory
```

### 3.2 Cấu hình trên Switch SW3
Switch SW3 đóng vai trò là switch L2 thông thường kết nối R3 với Server. Chỉ cần cấu hình Hostname và MOTD cơ bản:
```ios
enable
configure terminal
hostname SW3
banner motd # SW3 - DMZ SWITCH #
exit
write memory
```

### 3.3 Cấu hình trên Web Server
Click đúp vào máy Server trong Packet Tracer, vào phần **Desktop -> IP Configuration** để đặt cấu hình IP tĩnh:
- **IP Address**: `192.168.1.2`
- **Subnet Mask**: `255.255.255.0`
- **Default Gateway**: `192.168.1.1`
- **DNS Server**: `8.8.8.8`

*Vào tab **Services -> HTTP**, đảm bảo cả hai mục **HTTP** và **HTTPS** đều được chọn **On**. Có thể chỉnh sửa nội dung file `index.html` tùy ý để dễ nhận biết.*

---

## 4. ⚠️ CẢNH BÁO LỖI CẤU HÌNH THƯỜNG GẶP (TROUBLESHOOTING)

> [!WARNING]
> ### 1. Lỗi Định dạng MAC khi cấu hình DHCP Static Binding
> - **Triệu chứng**: PC4 xin IP động nhưng không nhận được địa chỉ `172.30.0.40` mà nhận một IP ngẫu nhiên khác hoặc APIPA (`169.254.x.x`).
> - **Nguyên nhân**: Nhập sai định dạng MAC trong lệnh `client-identifier`. Cisco IOS đòi hỏi phải thêm tiền tố `01` (Ethernet) trước địa chỉ MAC và định dạng theo dạng `01xx.xxxx.xxxx.xx` (viết thường).
> - **Cách khắc phục**: Vào PC4 -> Tab Config hoặc Command Prompt gõ `ipconfig /all` để lấy Physical Address (MAC). Ví dụ MAC là `000A.41F2.B3D4`. Tiền tố thêm vào sẽ thành: `0100.0a41.f2b3.d4`. Xóa pool cũ và nhập lại câu lệnh chính xác.

> [!IMPORTANT]
> ### 2. Bẫy "Cấm tất cả" (Implicit Deny All) trong ACL khiến Server mất kết nối
> - **Triệu chứng**: Sau khi áp dụng ACL, không chỉ VLAN 20 bị chặn Ping, mà cả VLAN 10 cũng không thể ping hoặc truy cập được Server. Thậm chí Server cũng không thể ping ra ngoài.
> - **Nguyên nhân**: Cuối mỗi Access-List của Cisco luôn có một lệnh cấm ẩn là `deny ip any any`. Nếu bạn không khai báo dòng `permit ip any any` ở cuối ACL, mọi gói tin đi qua cổng `e0/1` không thuộc trường hợp Web của VLAN 20 sẽ bị hủy toàn bộ.
> - **Cách khắc phục**: Luôn kiểm tra xem đã có dòng `permit ip any any` ở cuối ACL `DMZ_SECURITY` hay chưa.

> [!CAUTION]
> ### 3. Sai thứ tự ưu tiên của các Rule trong ACL
> - **Triệu chứng**: PC4 vẫn có thể ping được Server mặc dù đã có lệnh cấm PC4.
> - **Nguyên nhân**: Bạn đặt dòng cấm PC4 ở dưới dòng cho phép chung, hoặc viết sai IP nguồn/đích. Router duyệt ACL từ trên xuống dưới, gặp dòng khớp đầu tiên sẽ thực thi ngay và bỏ qua các dòng dưới.
> - **Cách khắc phục**: Luôn đặt các rule cấm chi tiết (như cấm host PC4 `172.30.0.40` hoặc cấm VLAN 20) lên phía trên cùng của Access-List, các lệnh cho phép chung `permit ip any any` phải đặt dưới cùng.

---

## 5. Lệnh kiểm tra và nghiệm thu công việc (Verification)

Gõ các lệnh sau trên Router R3:

1. **`show ip dhcp binding`**:
   - Kiểm tra xem máy PC2, PC3 và đặc biệt là PC4 đã nhận đúng IP được cấp chưa. PC4 phải hiển thị dòng gắn kết địa chỉ `172.30.0.40` với kiểu cấp phát là `Manual` (Tĩnh).
2. **`show ip access-lists`**:
   - Hiển thị danh sách các luật bảo mật. 
   - Thử ping từ PC4 đến Server -> Kiểm tra xem dòng `deny ip host 172.30.0.40...` có tăng chỉ số `matches` lên không.
   - Thử truy cập Web từ PC2 -> Dòng `permit tcp... eq 80` phải tăng chỉ số `matches`.
3. **`show ip route`**:
   - Xác minh bảng định tuyến có đầy đủ các mạng nội bộ thông qua OSPF (các dòng chữ **O** trỏ sang `192.168.100.5` hoặc `192.168.100.9`).
