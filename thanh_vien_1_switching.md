# HƯỚNG DẪN CẤU HÌNH - THÀNH VIÊN 1
## Vai trò: Switching Specialist (Chuyên gia Chuyển mạch L2)
## Thiết bị phụ trách: `SW1` & `SW2`

---

## 1. Sơ đồ Phân vùng Switching L2

```text
                 +-------------------+
                 |     Router R2     |
                 +---------+---------+
                           | e0/1 (Trunking: dot1q)
                           |
                           | e0/1 (Trunking)
                 +---------+---------+
                 |    Switch SW1     |  e0/0 (Trunking)
                 |   (VTP Server)    +========================+
                 +----+----+----+----+                        |
                      |    |    |                             |
            e1/0-3    |    |    | e3/0-3                      |
        +-------------+    |    +-------------+               |
        |                  | e2/0-3           |               |
  +-----+------+     +-----+------+     +-----+------+        |
  | PC1 (VL10) |     | PC2 (VL20) |     | PC3 (VL20) |        |
  +------------+     +------------+     +------------+        |
                                                              |
                                                              | e0/0 (Trunking)
                                                    +---------+---------+
                                                    |    Switch SW2     |
                                                    |   (VTP Client)    |
                                                    +----+--------------+
                                                         | e3/0-3
                                                         |
                                                    +----+--------------+
                                                    | PC4 (VL30: .40)   |
                                                    +-------------------+
```

### 1.1 Bảng Phân bổ Cổng và IP trong Phân vùng L2
Vì SW1 và SW2 là các thiết bị chuyển mạch Layer 2 (L2 Switch), chúng hoạt động ở Lớp 2 và không cấu hình địa chỉ IP Layer 3 trực tiếp trên cổng. Tuy nhiên, các cổng cần được gán chính xác vào các VLAN để dẫn đến Gateway (IP Lớp 3) trên Router R2:

| Thiết bị | Cổng (Interface) | VLAN gán | Chế độ | Thiết bị đối diện | Địa chỉ IP Gateway (Lớp 3) |
|---|---|---|---|---|---|
| **SW1** | `e0/0` | - | Trunking | SW2 `e0/0` | *Không cấu hình IP* |
| | `e0/1` | - | Trunking | Router R2 `e0/1` | *Không cấu hình IP* |
| | `e1/0` - `e1/3` | VLAN 10 | Access | PC1 | Gateway trên R2: `192.168.10.1` |
| | `e2/0` - `e2/3` | VLAN 20 | Access | PC2, PC3 | Gateway trên R2: `172.20.0.1` |
| | `e3/0` - `e3/3` | VLAN 30 | Access | - | Gateway trên R2: `172.30.0.1` |
| **SW2** | `e0/0` | - | Trunking | SW1 `e0/0` | *Không cấu hình IP* |
| | `e3/0` - `e3/3` | VLAN 30 | Access | PC4 | Gateway trên R2: `172.30.0.1` |

---

## 2. Danh sách Công việc cần thực hiện

1. Cấu hình các thông số cơ bản (Hostname, MOTD Banner) cho cả 2 Switch.
2. Cấu hình kết nối Trunking giữa `SW1 ↔ SW2` và `SW1 ↔ R2`.
3. Cấu hình **VTP (VLAN Trunking Protocol)**: `SW1` làm **VTP Server**, `SW2` làm **VTP Client**.
4. Khởi tạo các VLAN: `10`, `20`, `30`, `40`, `50` trên `SW1` (sẽ tự động đồng bộ sang `SW2`).
5. Gán các cổng truy cập (Access Ports) vào các VLAN tương ứng trên cả hai switch theo dải cổng được yêu cầu:
   - **VLAN 10**: dải cổng từ `e1/0` đến `e1/3`.
   - **VLAN 20**: dải cổng từ `e2/0` đến `e2/3`.
   - **VLAN 30**: dải cổng từ `e3/0` đến `e3/3`.
6. Cấu hình **Spanning-Tree PortFast** trên các cổng Access để tối ưu hóa thời gian nhận DHCP của các máy tính PC.

---

## 3. Các lệnh cấu hình chi tiết (Cisco IOS)

### 3.1 Cấu hình trên Switch SW1 (VTP Server)

```ios
enable
configure terminal

! 1. Cấu hình Hostname & Banner MOTD
hostname SW1
banner motd #
===================================================
*  BÀI LAB 6 - NHÓM X                              *
*  SW1 - THÀNH VIÊN 1 (SWITCHING SPECIALIST)      *
===================================================
#

! 2. Cấu hình VTP Server
vtp mode server
vtp domain Lab6QTM
vtp password 123

! 3. Tạo các VLAN
vlan 10
 name VLAN_10
vlan 20
 name VLAN_20
vlan 30
 name VLAN_30
vlan 40
 name VLAN_40
vlan 50
 name VLAN_50
exit

! 4. Cấu hình Trunking nối lên Router R2 và nối sang SW2
! Lưu ý: Trên một số dòng Switch (như 3560/3750 hoặc IOL), phải chỉ định encapsulation trước.
interface e0/0
 switchport trunk encapsulation dot1q
 switchport mode trunk
 no shutdown

interface e0/1
 switchport trunk encapsulation dot1q
 switchport mode trunk
 no shutdown

! 5. Gán Access ports và cấu hình PortFast cho các thiết bị đầu cuối
interface range e1/0-3
 switchport mode access
 switchport access vlan 10
 spanning-tree portfast
 no shutdown

interface range e2/0-3
 switchport mode access
 switchport access vlan 20
 spanning-tree portfast
 no shutdown

interface range e3/0-3
 switchport mode access
 switchport access vlan 30
 spanning-tree portfast
 no shutdown

exit
write memory
```

### 3.2 Cấu hình trên Switch SW2 (VTP Client)

```ios
enable
configure terminal

! 1. Cấu hình Hostname & Banner MOTD
hostname SW2
banner motd #
===================================================
*  BÀI LAB 6 - NHÓM X                              *
*  SW2 - THÀNH VIÊN 1 (SWITCHING SPECIALIST)      *
===================================================
#

! 2. Cấu hình Trunking về phía SW1
interface e0/0
 switchport trunk encapsulation dot1q
 switchport mode trunk
 no shutdown

! 3. Cấu hình VTP Client
vtp mode client
vtp domain Lab6QTM
vtp password 123

! 4. Gán Access ports và cấu hình PortFast cho PC4 (VLAN 30)
interface range e3/0-3
 switchport mode access
 switchport access vlan 30
 spanning-tree portfast
 no shutdown

! Nếu có gắn thêm thiết bị thuộc VLAN khác vào SW2, hãy cấu hình tương tự:
interface range e1/0-3
 switchport mode access
 switchport access vlan 10
 spanning-tree portfast
 no shutdown

interface range e2/0-3
 switchport mode access
 switchport access vlan 20
 spanning-tree portfast
 no shutdown

exit
write memory
```

---

## 4. ⚠️ CẢNH BÁO LỖI CẤU HÌNH THƯỜNG GẶP (TROUBLESHOOTING)

> [!WARNING]
> ### 1. Lỗi Không đồng bộ được VLAN (VTP Mismatch)
> - **Triệu chứng**: Gõ `show vlan brief` trên SW2 không thấy các VLAN 10, 20, 30, 40, 50.
> - **Nguyên nhân**: Sai tên miền (Domain Name) hoặc mật khẩu VTP. Lưu ý rằng tên miền VTP phân biệt chữ hoa/chữ thường (ví dụ: `Lab6QTM` khác `lab6qtm`).
> - **Cách khắc phục**: Dùng lệnh `show vtp status` trên cả hai switch để đối chiếu domain name và số lượng VLAN. Đảm bảo gõ lại lệnh cấu hình domain và password thật chính xác trên cả hai bên.

> [!IMPORTANT]
> ### 2. Lỗi DHCP Timeout trên PC do độ trễ Spanning-Tree (STP)
> - **Triệu chứng**: Các PC (PC2, PC3, PC4) cấu hình nhận IP động (DHCP) bị báo lỗi `DHCP failed. APIPA is being used` mặc dù Router R3 đã cấu hình đúng DHCP Server.
> - **Nguyên nhân**: Khi cắm cáp hoặc khởi động lại thiết bị, giao thức STP tiêu chuẩn cần mất tới 30 - 50 giây để chuyển trạng thái từ Blocking -> Listening -> Learning -> Forwarding. Trong thời gian này, gói tin xin IP (DHCP Discover) gửi đi từ PC bị Switch chặn lại, dẫn tới hết hạn (Timeout).
> - **Cách khắc phục**: Nhất định phải thêm lệnh `spanning-tree portfast` trên các cổng Access kết nối trực tiếp đến PC để bỏ qua các bước chờ này và chuyển ngay sang trạng thái Forwarding.

> [!CAUTION]
> ### 3. Lỗi Lệnh Trunk bị từ chối (Trunk Encapsulation Reject)
> - **Triệu chứng**: Gõ lệnh `switchport mode trunk` bị báo lỗi: `Command rejected: An interface whose trunk encapsulation is "Auto" can not be configured to "Trunk" mode.`
> - **Nguyên nhân**: Một số switch Cisco cao cấp mặc định sử dụng cơ chế Dynamic Trunking Protocol (DTP) nên không cho phép ép cổng về chế độ Trunk trực tiếp.
> - **Cách khắc phục**: Luôn chạy lệnh `switchport trunk encapsulation dot1q` trước khi chạy lệnh `switchport mode trunk` để giải quyết triệt để lỗi này.

---

## 5. Lệnh kiểm tra và nghiệm thu công việc (Verification)

Sau khi cấu hình xong, gõ các lệnh sau để đảm bảo hệ thống chuyển mạch hoạt động hoàn hảo:

1. **`show vtp status`**: 
   - SW1 phải hiển thị: `VTP Operating Mode: Server`.
   - SW2 phải hiển thị: `VTP Operating Mode: Client`.
   - VTP Domain Name của cả hai phải giống hệt nhau (ví dụ: `Lab6QTM`).
2. **`show interfaces trunk`**:
   - Kiểm tra các port `e0/0`, `e0/1` trên SW1 và `e0/0` trên SW2 đã ở trạng thái `trunking`, encapsulation là `802.1q`.
3. **`show vlan brief`**:
   - Trên SW2 phải hiển thị đầy đủ danh sách VLAN từ 10 đến 50 với các cổng đã gán chính xác giống SW1.
