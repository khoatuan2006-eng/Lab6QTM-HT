# HƯỚNG DẪN CẤU HÌNH - THÀNH VIÊN 3
## Vai trò: Gateway & NAT Specialist (Chuyên gia Cổng kết nối & NAT)
## Thiết bị phụ trách: Router `R1`

---

## 1. Sơ đồ Cổng kết nối và Biên dịch NAT (R1)

```text
                                  +-------------------+
                                  |    ISP Internet   | (NAT OUTSIDE)
                                  +---------+---------+
                                            | e0/0 (DHCP hoặc IP Tĩnh: 100.100.100.1/24)
                                            |
                                            | e0/0
                                  +---------+---------+
                                  |     Router R1     |
                                  | (NAT & Telnet Edge|
                                  +----+-----------+--+
                                       |           |
               e0/1 (192.168.100.1/30) |           | e0/2 (192.168.100.5/30)
                        (NAT INSIDE)   |           | (NAT INSIDE)
                                       |           |
                             +---------v-+       +-v---------+
                             | Router R2 |       | Router R3 |
                             +-----------+       +-----------+
```

### 1.1 Bảng phân bổ Địa chỉ IP của Router R1
Để cấu hình chính xác cho Router biên kết nối Internet R1, bạn cần nắm rõ sơ đồ cổng và địa chỉ IP cụ thể dưới đây:

| Cổng (Interface) | Địa chỉ IP | Subnet Mask | Thiết bị đối diện | Ghi chú |
|---|---|---|---|---|
| `e0/0` | DHCP từ ISP *(Dải Public `100.100.100.0/24`)* | - | ISP Gateway | Cổng kết nối ngoài (NAT Outside) |
| `e0/1` | `192.168.100.1` | `255.255.255.252` | Router R2 `e0/0` | Cổng kết nối trong (NAT Inside - Nối sang R2) |
| `e0/2` | `192.168.100.5` | `255.255.255.252` | Router R3 `e0/0` | Cổng kết nối trong (NAT Inside - Nối sang R3) |

---

## 2. Danh sách Công việc cần thực hiện

1. Cấu hình Hostname và Banner MOTD cho Router `R1`.
2. Cấu hình IP và phân định biên dịch NAT trên các interface:
   - Cổng `e0/0` (Nối ra ISP): Nhận IP động qua DHCP hoặc đặt IP tĩnh `100.100.100.1/24` theo đề bài. Cấu hình cổng này làm **`ip nat outside`**.
   - Cổng `e0/1` (Nối sang `R2`): `192.168.100.1/30`. Cấu hình làm **`ip nat inside`**.
   - Cổng `e0/2` (Nối sang `R3`): `192.168.100.5/30`. Cấu hình làm **`ip nat inside`**.
3. Cấu hình Default Route (`0.0.0.0/0`) hướng ra ngoài Internet thông qua cổng `e0/0`.
4. Cấu hình **OSPF (Process ID 1, Area 0)** trên 2 cổng kết nối nội bộ (`e0/1` và `e0/2`).
5. **Đặc biệt**: Sử dụng lệnh **`default-information originate`** trong OSPF để quảng bá Default Route cho `R2` và `R3` tự động học được.
6. Cấu hình **NAT Overload (PAT)**: Cho phép các thiết bị từ VLAN 10 và VLAN 20 truy cập Internet, **cấm hoàn toàn VLAN 30 ra Internet**.
7. Cấu hình **Static NAT (Port Forwarding)**: Cho phép thiết bị Client từ Internet truy cập dịch vụ Web của Server trong DMZ (`192.168.1.2`) qua cổng Public của R1.
8. Cấu hình bảo mật quản trị: Chỉ cho phép các thiết bị thuộc **VLAN 10 (`192.168.10.0/24`)** được phép **Telnet** vào quản trị Router `R1`.

---

## 3. Các lệnh cấu hình chi tiết (Cisco IOS)

```ios
enable
configure terminal

! 1. Cấu hình Hostname & Banner MOTD
hostname R1
banner motd #
===================================================
*  BÀI LAB 6 - NHÓM X                              *
*  R1 - THÀNH VIÊN 3 (GATEWAY & NAT SPECIALIST)    *
===================================================
#

! 2. Cấu hình IP các interface và chỉ định vùng INSIDE/OUTSIDE cho NAT
! Cổng nối ra ISP Internet
interface e0/0
 ip address dhcp
 ! Hoặc đặt IP tĩnh nếu ISP không cấp DHCP: ip address 100.100.100.1 255.255.255.0
 ip nat outside
 no shutdown

! Cổng nối sang Router Core R2
interface e0/1
 ip address 192.168.100.1 255.255.255.252
 ip nat inside
 no shutdown

! Cổng nối sang Router Dịch vụ R3 (Đường chính DMZ)
interface e0/2
 ip address 192.168.100.5 255.255.255.252
 ip nat inside
 no shutdown

! 3. Cấu hình Default Route hướng ra ISP
! Nếu sử dụng DHCP ở e0/0, mặc định Router sẽ tự tạo Default Route.
! Để chắc chắn hoặc khi dùng IP tĩnh, chạy lệnh sau:
ip route 0.0.0.0 0.0.0.0 e0/0

! 4. Cấu hình OSPF và quảng bá Default Route xuống mạng nội bộ
router ospf 1
 ! Chỉ quảng bá các subnet nội bộ trực tiếp kết nối (Không quảng bá cổng ISP e0/0)
 network 192.168.100.0 0.0.0.3 area 0
 network 192.168.100.4 0.0.0.3 area 0
 ! Quảng bá Default Route xuống cho R2 và R3 tự động học
 default-information originate
 exit

! 5. Cấu hình NAT Overload (PAT)
! Định nghĩa Access-list cho NAT: Chỉ cho phép VLAN 10 và VLAN 20.
! KHÔNG thêm VLAN 30 (172.30.0.0/16) -> VLAN 30 sẽ bị cấm ra Internet theo cơ chế "Implicit Deny All".
access-list 10 permit 192.168.10.0 0.0.0.255
access-list 10 permit 172.20.0.0 0.0.255.255

! Áp dụng NAT Overload lên cổng ngoài e0/0
ip nat inside source list 10 interface e0/0 overload

! 6. Cấu hình Static NAT (Truy cập Web Server từ Internet)
! Ánh xạ cổng dịch vụ Web (HTTP-80 và HTTPS-443) của Server (192.168.1.2) ra địa chỉ IP Public của R1
ip nat inside source static tcp 192.168.1.2 80 interface e0/0 80
ip nat inside source static tcp 192.168.1.2 443 interface e0/0 443

! LƯU Ý: Nếu thầy cô yêu cầu Static NAT ánh xạ 1-to-1 toàn bộ IP (không chỉ port Web), dùng lệnh:
! ip nat inside source static 192.168.1.2 100.100.100.2

! 7. Cấu hình bảo mật quản trị (Chỉ cho phép VLAN 10 Telnet)
! Định nghĩa danh sách IP được phép Telnet (VLAN 10)
access-list 20 permit 192.168.10.0 0.0.0.255

username admin secret 123
line vty 0 4
 login local
 transport input telnet
 ! Áp dụng bộ lọc ACL 20 cho các kết nối Telnet đi vào
 access-class 20 in
 exit

write memory
```

---

## 4. ⚠️ CẢNH BÁO LỖI CẤU HÌNH THƯỜNG GẶP (TROUBLESHOOTING)

> [!WARNING]
> ### 1. Quên lệnh "default-information originate" trong OSPF
> - **Triệu chứng**: R1 ping Internet (`8.8.8.8`) thành công, nhưng các Router nội bộ (R2, R3) và toàn bộ các PC đều không thể ping ra Internet, báo lỗi `Destination Host Unreachable`.
> - **Nguyên nhân**: Bạn đã cấu hình NAT và Default Route trên R1 rất tốt, nhưng R2 và R3 không hề biết đường đi ra Internet nằm ở đâu vì chúng không có Default Route.
> - **Cách khắc phục**: Nhất định phải thêm lệnh `default-information originate` trong tiến trình OSPF trên R1 để tự động phân phối tuyến đường `0.0.0.0/0` cho R2 và R3.

> [!IMPORTANT]
> ### 2. Quên định nghĩa "ip nat inside/outside" trên Interface
> - **Triệu chứng**: Các PC trong VLAN 10 và 20 không thể kết nối Internet mặc dù ACL NAT đã cấu hình đúng. Gõ `show ip nat translations` không hiển thị thông tin gì.
> - **Nguyên nhân**: Router R1 không biết cổng nào là cổng mạng nội bộ (Inside) và cổng nào là cổng hướng ra ngoài Internet (Outside) để kích hoạt cơ chế dịch địa chỉ.
> - **Cách khắc phục**: Kiểm tra lại interface `e0/0` đã có `ip nat outside` chưa, và các cổng `e0/1`, `e0/2` đã có `ip nat inside` chưa.

> [!CAUTION]
> ### 3. Lỗi Vô ý khóa toàn bộ kết nối quản trị (Telnet Lockout Trap)
> - **Triệu chứng**: Cấu hình xong `access-class 20 in` thì lập tức bị ngắt kết nối Telnet từ máy tính của bạn và không thể Telnet lại được nữa.
> - **Nguyên nhân**: Bạn đang thực hiện Telnet từ một thiết bị có IP không thuộc dải `192.168.10.0/24` (ví dụ máy PC2 thuộc VLAN 20 có IP `172.20.x.x`). ACL 20 có cơ chế ẩn là cấm tất cả các IP không được liệt kê (Implicit Deny).
> - **Cách khắc phục**: Trong Packet Tracer, bạn luôn có thể click trực tiếp vào tab CLI của thiết bị để sửa đổi. Hãy kiểm tra lại IP của máy trạm thực hiện cấu hình, đảm bảo gán đúng IP tĩnh thuộc VLAN 10 (ví dụ `192.168.10.2`) để có quyền truy cập.

---

## 5. Lệnh kiểm tra và nghiệm thu công việc (Verification)

Chạy các lệnh sau trên R1 để nghiệm thu cấu hình:

1. **`show ip route`**:
   - Phải thấy dòng Default Route trỏ ra cổng `e0/0` (ví dụ: `S* 0.0.0.0/0 [1/0] via e0/0`).
2. **`show ip nat translations`**:
   - Khi có PC bên trong (VLAN 10 hoặc 20) đang ping ra Internet, bảng này phải hiển thị các dòng ánh xạ (Inside Global, Inside Local, Outside Local, Outside Global).
   - Kiểm tra xem dòng Static NAT cho Web Server (`192.168.1.2`) có xuất hiện thường trực trong bảng không.
3. **`show ip ospf neighbor`**:
   - Đảm bảo thiết lập Neighbors thành công với `R2` (`192.168.100.2`) và `R3` (`192.168.100.6`).
4. **`show access-lists`**:
   - Kiểm tra số lượng gói tin khớp (matches) trên ACL 10 (NAT) và ACL 20 (Telnet) để xác minh các bộ lọc đang hoạt động hiệu quả.
