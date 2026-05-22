# HƯỚNG DẪN CẤU HÌNH - THÀNH VIÊN 2
## Vai trò: Core Routing Specialist (Chuyên gia Định tuyến Lõi)
## Thiết bị phụ trách: Router `R2`

---

## 1. Sơ đồ Phân vùng Định tuyến L3 của R2

```text
                                  +-------------------+
                                  |     Router R1     |
                                  +---------+---------+
                                            | e0/1 (192.168.100.1/30)
                                            |
                                            | e0/0 (192.168.100.2/30)
                                  +---------+---------+
                                  |     Router R2     | e0/2 (192.168.100.9/30)
                                  |   (Core Router)   +======================+
                                  +---------+---------+                      |
                                            | e0/1 (Trunking)                |
                                            | (Sub-interfaces)               |
                                            |                                |
                                  +---------+---------+                      | (ĐƯỜNG DỰ PHÒNG)
                                  |    Switch SW1     |                      | (Cost: 100)
                                  +---------+---------+                      |
                                            |                                |
       +------------------------------------+--------------------------------+
       | e0/1.10 (192.168.10.1/24)          | e0/1.20 (172.20.0.1/16)        | e0/1.30 (172.30.0.1/16)
       |                                    |                                |
+------v-----------+                 +------v-----------+             +------v-----------+
| VLAN 10 (Static) |                 | VLAN 20 (DHCP)   |             | VLAN 30 (DHCP)   |
+------------------+                 +------------------+             +------------------+
                                                                             |
                                                                             | e0/3 (192.168.100.10/30)
                                                                      +------v-----------+
                                                                      |     Router R3     |
                                                                      +------------------+
```

### 1.1 Bảng phân bổ Địa chỉ IP của Router R2
Để cấu hình chính xác cho Router lõi R2, bạn cần nắm rõ sơ đồ cổng và địa chỉ IP cụ thể dưới đây:

| Cổng (Interface) | VLAN liên kết | Địa chỉ IP | Subnet Mask | Thiết bị đối diện | Ghi chú |
|---|---|---|---|---|---|
| `e0/0` | - | `192.168.100.2` | `255.255.255.252` | Router R1 `e0/1` | Đường truyền định tuyến chính |
| `e0/2` | - | `192.168.100.9` | `255.255.255.252` | Router R3 `e0/3` | Đường truyền dự phòng (cấu hình Cost 100) |
| `e0/1` | - | *Không cấu hình* | - | Switch SW1 `e0/1` | Cổng vật lý chính (phải `no shutdown`) |
| `e0/1.10` | VLAN 10 | `192.168.10.1` | `255.255.255.0` | Vùng mạng VLAN 10 | Gateway VLAN 10 (IP Tĩnh) |
| `e0/1.20` | VLAN 20 | `172.20.0.1` | `255.255.0.0` | Vùng mạng VLAN 20 | Gateway VLAN 20 (IP Động, có DHCP Relay) |
| `e0/1.30` | VLAN 30 | `172.30.0.1` | `255.255.0.0` | Vùng mạng VLAN 30 | Gateway VLAN 30 (IP Động, có DHCP Relay) |

---

## 2. Danh sách Công việc cần thực hiện

1. Cấu hình Hostname và Banner MOTD cho Router `R2`.
2. Chia nhỏ cổng vật lý `e0/1` thành các sub-interfaces:
   - `e0/1.10` cho VLAN 10.
   - `e0/1.20` cho VLAN 20.
   - `e0/1.30` cho VLAN 30.
3. Gán địa chỉ IP Gateway tương ứng cho từng sub-interface và bật cổng vật lý `e0/1`.
4. Cấu hình IP cho các cổng Point-to-Point nối sang các Router khác:
   - `e0/0` (Nối sang `R1`): `192.168.100.2/30`.
   - `e0/2` (Nối sang `R3`): `192.168.100.9/30` (đường dự phòng).
5. Cấu hình giao thức định tuyến động **OSPF (Process ID 1, Area 0)** trên tất cả các mạng kết nối trực tiếp.
6. Cấu hình **OSPF Cost** trên interface `e0/2` tăng lên `100` để ép dữ liệu ưu tiên đi qua đường `R2-R1-R3` làm đường chính.
7. Cấu hình **DHCP Relay Agent (IP Helper-address)** trên sub-interface VLAN 20 và VLAN 30, chuyển hướng các bản tin xin cấp IP đến Router dịch vụ `R3` (IP `192.168.100.10`).

---

## 3. Các lệnh cấu hình chi tiết (Cisco IOS)

```ios
enable
configure terminal

! 1. Cấu hình Hostname & Banner MOTD
hostname R2
banner motd #
===================================================
*  BÀI LAB 6 - NHÓM X                              *
*  R2 - THÀNH VIÊN 2 (CORE ROUTING SPECIALIST)    *
===================================================
#

! 2. Cấu hình sub-interfaces (Router-on-Stick)
! Cấu hình sub-interface cho VLAN 10 (IP Tĩnh)
interface e0/1.10
 encapsulation dot1Q 10
 ip address 192.168.10.1 255.255.255.0
 no shutdown

! Cấu hình sub-interface cho VLAN 20 (IP Động)
interface e0/1.20
 encapsulation dot1Q 20
 ip address 172.20.0.1 255.255.0.0
 ! DHCP Relay: Chuyển hướng yêu cầu DHCP đến IP cổng e0/3 của R3
 ip helper-address 192.168.100.10
 no shutdown

! Cấu hình sub-interface cho VLAN 30 (IP Động)
interface e0/1.30
 encapsulation dot1Q 30
 ip address 172.30.0.1 255.255.0.0
 ! DHCP Relay: Chuyển hướng yêu cầu DHCP đến IP cổng e0/3 của R3
 ip helper-address 192.168.100.10
 no shutdown

! 3. Bật cổng vật lý e0/1 nối xuống SW1 (Rất quan trọng!)
interface e0/1
 no shutdown

! 4. Cấu hình cổng nối sang Router R1 (Đường chính)
interface e0/0
 ip address 192.168.100.2 255.255.255.252
 no shutdown

! 5. Cấu hình cổng nối sang Router R3 (Đường dự phòng)
interface e0/2
 ip address 192.168.100.9 255.255.255.252
 ! Tăng Cost OSPF để đường này có độ ưu tiên thấp hơn (Đường dự phòng)
 ip ospf cost 100
 no shutdown

! 6. Cấu hình giao thức OSPF
router ospf 1
 ! Quảng bá các subnet của VLAN (Lưu ý dùng Wildcard Mask ngược với Subnet Mask)
 network 192.168.10.0 0.0.0.255 area 0
 network 172.20.0.0 0.0.255.255 area 0
 network 172.30.0.0 0.0.255.255 area 0
 ! Quảng bá các đường Point-to-Point giữa các Router
 network 192.168.100.0 0.0.0.3 area 0
 network 192.168.100.8 0.0.0.3 area 0
 exit

write memory
```

---

## 4. ⚠️ CẢNH BÁO LỖI CẤU HÌNH THƯỜNG GẶP (TROUBLESHOOTING)

> [!WARNING]
> ### 1. Quên không bật cổng vật lý e0/1 (Interface Down)
> - **Triệu chứng**: Gõ lệnh `show ip interface brief` thấy các sub-interfaces `e0/1.10`, `e0/1.20`, `e0/1.30` đều ở trạng thái `down/down` hoặc `protocol down`. Các PC không thể ping được Gateway của mình.
> - **Nguyên nhân**: Bạn đã bật `no shutdown` trên từng sub-interface nhưng quên không bật cổng vật lý `e0/1`. Sub-interface chỉ là cổng ảo và nó phụ thuộc hoàn toàn vào trạng thái vật lý của cổng chính.
> - **Cách khắc phục**: Vào cổng vật lý `interface e0/1` và gõ `no shutdown`.

> [!IMPORTANT]
> ### 2. Sai lệch OSPF Cost một chiều (Asymmetric Routing)
> - **Triệu chứng**: Đường truyền chính đi qua R1 hoạt động tốt từ R2 -> R3, nhưng luồng phản hồi từ R3 -> R2 lại đi trực tiếp qua link R3-R2, không đi qua R1.
> - **Nguyên nhân**: Bạn mới chỉ cấu hình `ip ospf cost 100` trên cổng `e0/2` của R2. Router R3 ở đầu bên kia của link (cổng `e0/3`) vẫn giữ OSPF Cost mặc định (ví dụ: `1` hoặc `10`), khiến OSPF trên R3 đánh giá đường R3-R2 tốt hơn R3-R1-R2.
> - **Cách khắc phục**: Nhắc nhở **Thành viên 4** cấu hình `ip ospf cost 100` trên cổng `e0/3` của R3 để đảm bảo tính đồng bộ cả hai chiều.

> [!CAUTION]
> ### 3. Sai Wildcard Mask trong cấu hình OSPF
> - **Triệu chứng**: Các Router lân cận (R1, R3) không học được bảng định tuyến của các VLAN 20, 30 (`172.20.0.0/16` và `172.30.0.0/16`).
> - **Nguyên nhân**: Sử dụng sai Wildcard mask trong câu lệnh `network`. VLAN 20 và 30 có Subnet Mask là `/16` (`255.255.0.0`), do đó Wildcard Mask tương ứng phải là `0.0.255.255`. Nếu viết nhầm thành `0.0.0.255` (mask của `/24`), OSPF sẽ chỉ tìm kiếm cổng có IP thuộc dải `172.20.0.x` và bỏ sót gateway thực tế.
> - **Cách khắc phục**: Xóa lệnh cũ và gõ lại đúng Wildcard Mask `/16` là `0.0.255.255`.

---

## 5. Lệnh kiểm tra và nghiệm thu công việc (Verification)

Gõ các lệnh sau trên Router R2 để xác minh cấu hình:

1. **`show ip interface brief`**: 
   - Kiểm tra tất cả các interface `e0/0`, `e0/1`, `e0/2` và các sub-interfaces ảo đều phải ở trạng thái `up/up`.
2. **`show ip ospf neighbor`**:
   - Phải nhìn thấy 2 người láng giềng OSPF (Neighbors) ở trạng thái `FULL`:
     - R1 (`192.168.100.1` qua cổng `e0/0`).
     - R3 (`192.168.100.10` qua cổng `e0/2`).
3. **`show ip route`**:
   - Phải nhìn thấy các tuyến đường học được bằng giao thức OSPF (ký hiệu chữ **O** ở đầu dòng) trỏ tới vùng DMZ (`192.168.1.0/24`) và cổng Gateway mặc định.
4. **`show ip ospf interface e0/2`**:
   - Kiểm tra dòng hiển thị Cost. Phải hiển thị chính xác `Cost: 100` để đảm bảo đường dự phòng đã được thiết lập đúng thông số.
