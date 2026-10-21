# LAB 7 – XÂY DỰNG CHÍNH SÁCH WEB PROTECTION TRÊN pfSense + WEB PROXY

## 1. Thông tin sinh viên

- **Họ và tên:** Lý Chí Chương
- **MSSV:** 1150080128
- **Môn học:** Thực hành An toàn hệ thống thông tin
- **Bài thực hành:** LAB 7 – Xây dựng chính sách Web Protection trên mô hình pfSense + Web Proxy

## 2. Mục tiêu

Xây dựng mô hình pfSense làm firewall/gateway và Web Proxy làm lớp kiểm soát truy cập website. Thực hành cấu hình alias, rule cho phép truy cập proxy, rule chặn truy cập HTTP/HTTPS trực tiếp và chuẩn bị chính sách lọc nội dung theo nhóm website. Phân biệt rõ cấu hình đã thực hiện với kiểm thử đã xác nhận.

## 3. Môi trường thực hành

| Thành phần | Phiên bản / cấu hình | Vai trò |
|---|---|---|
| VirtualBox | Phiên bản chưa ghi nhận | Nền tảng máy ảo |
| pfSense-LAB6 | pfSense CE 2.7.2 | Firewall/gateway |
| Ubuntu-Server-Lab1 / WEB-PROXY-LAB7 | Ubuntu 26.04.1 LTS (máy đã kiểm tra) | Máy dự kiến chạy Squid, E2Guardian, ClamAV |
| Windows host | Windows 11, build 26200.9711 | Truy cập WebGUI và quản trị lab |
| DC-Lab3 | Chưa xác nhận cấu hình dịch vụ | Dự kiến làm DC/DNS |

**Lưu ý:** Tài liệu LAB 7 yêu cầu Ubuntu Server 24.04 LTS. Máy Ubuntu thực tế được kiểm tra là 26.04.1 LTS, nên chưa thể xác nhận môi trường hoàn toàn khớp tài liệu.

### Địa chỉ mạng theo mô hình

| Thành phần | Địa chỉ |
|---|---|
| pfSense WAN | 192.168.250.3 |
| pfSense LAN | 10.6.0.1/24 |
| pfSense MGMT | 192.168.56.254/24 |
| pfSense DMZ | 172.16.6.1/24 |
| Windows Host-only | 192.168.56.1/24 |
| WEB-PROXY dự kiến | 10.6.0.30/24 |
| DNS/DC01 dự kiến | 10.6.0.10 |
| Client-A/B dự kiến | 10.6.0.21 / 10.6.0.22 |
| DMZ-WEB dự kiến | 172.16.6.10 |

## 4. Các bước đã thực hiện

### 4.1. Khôi phục kết nối quản trị pfSense

Sau khi khởi động lại máy, trình duyệt không truy cập được `https://192.168.56.254`. Kiểm tra `ifconfig` xác nhận interface MGMT vẫn có địa chỉ `192.168.56.254`. Windows Host-only Adapter có IP `192.168.56.1`. Ban đầu ping thất bại; sau khi kiểm tra kết nối Host-only, ping từ pfSense đến `192.168.56.1` thành công (3/3 gói, mất gói 0%). Sau đó truy cập được WebGUI pfSense.

### 4.2. Tạo Alias trên pfSense

Truy cập **Firewall → Aliases** và tạo các alias phục vụ LAB 7:

| Alias | Kiểu | Giá trị |
|---|---|---|
| `LAB7_CLIENTS` | Host(s) | 10.6.0.21, 10.6.0.22 |
| `LAB7_PROXY` | Host(s) | 10.6.0.30 |
| `WEB_PORTS` | Port(s) | 80, 443 |
| `MAIL_PORTS` | Port(s) | 25, 465, 587, 110, 995, 143, 993 |

### 4.3. Cấu hình Firewall Rules

Tại **Firewall → Rules → LAN**, đã tạo và sắp xếp các rule sau (theo ảnh cấu hình gần nhất):

| Thứ tự | Rule | Action | Source | Destination | Port |
|---|---|---|---|---|---|
| 1 | `LAB7_ALLOW_DNS` | Pass | LAN subnets | Cần xác nhận đã sửa thành 10.6.0.10 | TCP/UDP 53 |
| 2 | `LAB7_ALLOW_PROXY` | Pass | LAB7_CLIENTS | LAB7_PROXY | TCP 8080 |
| 3 | `LAB7_ALLOW_PROXY_WEB` | Pass | LAB7_PROXY | Any | TCP WEB_PORTS |
| 4 | `LAB7_BLOCK_DIRECT_WEB` | Block | LAB7_CLIENTS | Any | TCP WEB_PORTS |

**Điểm cần kiểm tra:** Ảnh rule gần nhất cho thấy Destination của `LAB7_ALLOW_DNS` là `*` (Any). Cần chỉnh thành `10.6.0.10` và **Apply Changes**. Đồng thời kiểm tra không có rule Pass rộng hơn đứng trên rule chặn trực tiếp.

### 4.4. Chuẩn bị Ubuntu WEB-PROXY

Đã đăng nhập Ubuntu và kiểm tra các interface `enp0s3`, `enp0s8`. Netplan ban đầu có ràng buộc MAC từ máy gốc, dẫn đến lỗi `Cannot find unique matching interface for enp0s3`. Sau khi sửa file YAML, xuất hiện lỗi cú pháp `Invalid YAML: did not find expected '-' indicator`. Đã thử gán IP `10.6.0.30/24` tạm thời bằng lệnh `ip addr`, nhưng ping `10.6.0.1` vẫn báo `Destination Host Unreachable`. Vì vậy chưa triển khai thành công các dịch vụ Squid, E2Guardian và ClamAV.

## 5. Kiểm thử và trạng thái

| Nội dung | Trạng thái | Minh chứng / ghi chú |
|---|---|---|
| Truy cập WebGUI pfSense | **PASS** | Đã mở được Dashboard tại 192.168.56.254 |
| Kết nối pfSense → Windows Host-only | **PASS** | Ping 3/3 gói, 0% loss |
| Tạo Alias LAB 7 | **Đã cấu hình** | Cần lưu ảnh màn hình Alias để nộp |
| Tạo 4 rule LAN LAB 7 | **Đã cấu hình** | Có ảnh danh sách rule; cần xác nhận DNS Destination |
| Ubuntu WEB-PROXY → pfSense LAN | **FAIL** | Destination Host Unreachable |
| Chống bypass khi Client truy cập trực tiếp | **Chưa kiểm thử** | Chưa có Client-A/B được xác nhận IP và kết quả truy cập |
| Website bình thường qua proxy | **Chưa kiểm thử** | Proxy chưa hoạt động |
| Geo-Sim và Job Search hard block | **Chưa kiểm thử** | Chưa có E2Guardian và website mô phỏng |
| Warning / temporary bypass | **Chưa kiểm thử** | Chưa có E2Guardian |
| Education whitelist | **Chưa kiểm thử** | Chưa có E2Guardian |
| Entertainment/Game schedule 60 phút | **Chưa kiểm thử** | Chưa cấu hình schedule và kiểm thử thời gian |
| Streaming warning | **Chưa kiểm thử** | Chưa có E2Guardian |
| Specific game hard block | **Chưa kiểm thử** | Chưa có E2Guardian |
| ClamAV/EICAR | **Chưa kiểm thử** | Chưa triển khai AV scanner |
| Giới hạn file 100 MB | **Chưa kiểm thử** | Chưa cấu hình và đối chiếu file 100/110 MB |

**Quy ước:** “Đã cấu hình” không đồng nghĩa với “PASS”; chỉ ghi PASS khi có phép thử và bằng chứng tương ứng.

## 6. Lỗi gặp phải và hướng khắc phục

1. **Không truy cập được WebGUI sau khi mở lại máy:** kiểm tra Host-only Adapter, IP MGMT, trạng thái link và kết nối ping hai chiều. Sau khi mạng Host-only hoạt động lại, WebGUI truy cập thành công.
2. **Lệnh `Test-NetConnection` không chạy trong CMD:** gọi thông qua `powershell -Command "Test-NetConnection 192.168.56.254 -Port 443"`.
3. **Netplan không khớp interface sau khi clone:** kiểm tra ràng buộc MAC trong `/etc/netplan/00-installer-config.yaml`, đối chiếu MAC thực tế và tên card.
4. **Sai cú pháp YAML:** chỉnh đúng thụt lề và cấu trúc `routes`, kiểm tra bằng `sudo netplan generate` trước khi áp dụng.
5. **WEB-PROXY không ping được gateway:** kiểm tra tên Internal Network của VirtualBox trên cả pfSense và Ubuntu; hai card phải nối cùng một mạng ảo, sau đó kiểm tra ARP/ping.
6. **Alias port không hiện trong dropdown pfSense:** chọn `(other)` và nhập tên alias vào ô Custom, sau đó lưu rule.

## 7. Hướng hoàn thiện LAB 7

1. Xác nhận WEB-PROXY cùng Internal Network với LAN pfSense và ping được `10.6.0.1`.
2. Cấu hình IP tĩnh, DNS và cài Squid, E2Guardian, ClamAV trên Ubuntu phù hợp với tài liệu.
3. Dựng website mô phỏng tại DMZ và bản ghi DNS `lab7.test`.
4. Kiểm thử luồng proxy, chặn truy cập trực tiếp và từng chính sách website.
5. Đối chiếu log tại **Status → System Logs → Firewall**, `/var/log/e2guardian/access.log`, `/var/log/squid/access.log` và nhật ký ClamAV.
6. Chụp ảnh cấu hình và kết quả thật cho từng kịch bản, bổ sung vào báo cáo Word.

## 8. Kết luận

Trong quá trình thực hành, đã khôi phục được kết nối quản trị pfSense và thiết lập bộ alias/rule LAN cơ bản để định hướng lưu lượng Web qua proxy. Phần proxy trên Ubuntu chưa kết nối được đến gateway nên chưa đủ điều kiện xác nhận các chính sách lọc nội dung, quét virus và giới hạn tải file. Các phần này được ghi là chưa kiểm thử, không đánh dấu PASS khi chưa có bằng chứng thực nghiệm.

## 9. Tài liệu tham khảo

- Tài liệu nội bộ: **LAB 7 – Xây dựng chính sách Web Protection trên mô hình pfSense + Web Proxy** (18 trang).
- Netgate pfSense Documentation: Firewall Rules, Aliases, Schedules.
- Ubuntu Server / Netplan Documentation.
- E2Guardian, Squid và ClamAV documentation.
