# LAB 8 – LOCAL DNS ATTACK: DNS SPOOFING VÀ DNS CACHE POISONING TRÊN MÔ HÌNH PFSENSE

## 1\. Thông tin sinh viên

* **Họ và tên:** Lý Chí Chương
* **MSSV:** 1150080128
* **Môn học:** Thực hành An toàn hệ thống thông tin
* **Bài thực hành:** LAB 9 – Local DNS Attack (DNS Spoofing \& Cache Poisoning trong mạng ảo cô lập)

\---

## 2\. Mục tiêu bài lab

1. **Dựng máy chủ DNS BIND9** làm Recursive Resolver và Local Cache phục vụ riêng cho vùng mạng độc lập `DNS-LAB` (10.9.0.0/24).
2. **Cấu hình zone phân giải thuận/nghịch** (`lab9.test` và `9.10.in-addr.arpa`), cấu hình forwarder cho tên miền ngoài `external.lab9.test` trỏ về máy chủ mô phỏng `172.16.6.20` và kiểm thử bằng `dig`, `nslookup`, `resolvectl`.
3. **Phân tích cấu trúc gói tin DNS** qua Wireshark/tcpdump: 16-bit Transaction ID, UDP Ephemeral Source Port, Query Name, Flags và CNAME/A Record.
4. **Thực thi 3 kỹ thuật tấn công cốt lõi**:

   * **Task 1:** Điều hướng cục bộ bằng tệp tin `/etc/hosts` (HOSTS-file Redirection).
   * **Task 2:** Tấn công giả mạo phản hồi DNS trực tiếp vào máy nạn nhân (Direct DNS Response Spoofing - On-path race condition).
   * **Task 3:** Đầu độc bộ nhớ đệm máy chủ phân giải DNS (DNS Cache Poisoning nhắm vào BIND9 Resolver).
5. **Khôi phục trạng thái sạch** và đánh giá các giải pháp phòng vệ: Phân đoạn mạng (VLAN Segmentation), lọc DNS egress trên pfSense, DNSSEC, DNS COOKIE (RFC 7873) và vô hiệu hóa Promiscuous Mode.

\---

## 3\. Môi trường thực hành

|Thành phần|Hệ điều hành / Cấu hình|Địa chỉ IP|Vai trò \& Card VirtualBox|
|-|-|-|-|
|**pfSense**|pfSense CE 2.7.x|10.9.0.1/24 (DNSLAB)<br>172.16.6.1/24 (DMZ)<br>192.168.56.254 (MGMT)|Adapter 5: `dns-lab` (Internal Network)<br>Adapter 3: `dmz-net`<br>Adapter 1: Host-only|
|**DNS-SERVER**|Ubuntu Server 24.04 LTS (BIND9)|10.9.0.10/24|Adapter 1: `dns-lab` (Internal Network)|
|**VICTIM**|Ubuntu Desktop/Server 24.04 LTS|10.9.0.100/24|Adapter 1: `dns-lab` (Internal Network)|
|**ATTACKER**|Kali Linux 2026 (Scapy + Wireshark)|10.9.0.200/24|Adapter 1: `dns-lab` (**Promiscuous: Allow VMs**)|
|**APP-TEST / AUTH-DNS**|Ubuntu Server (Python simulator)|172.16.6.20/24|Adapter: `dmz-net` (Kế thừa từ LAB 8)|
|**DMZ-WEB**|Nginx Web Server hợp lệ|172.16.6.10/24|Adapter: `dmz-net` (Kế thừa từ LAB 8)|
|**HOST MGMT**|Windows 11 (Máy thật quản trị)|192.168.56.1/24|VirtualBox Host-Only Adapter|

> \*\*Quy định an toàn tuyệt đối:\*\* Toàn bộ script Scapy/Python và kỹ thuật giả mạo gói tin chỉ được chạy trong mạng ảo nội bộ `10.9.0.0/24`. Không can thiệp vào card mạng NAT/Wi-Fi hoặc Internet thật. Domain thử nghiệm bắt buộc dùng chuẩn RFC 2606 (`.test`).

\---

## 4\. Các bước đã thực hiện

### 4.1. Tạo Snapshot \& Khởi tạo Mạng DNSLAB trên pfSense

1. Tạo snapshot an toàn `LAB9\_START\_FROM\_LAB8` cho pfSense và Kali Linux trước khi can thiệp cấu hình.
2. Bổ sung **Adapter 5** cho pfSense trên VirtualBox: Attached to `Internal Network`, Name = `dns-lab`.
3. Gán Interface trên WebGUI pfSense (**Interfaces → Assignments**):

   * Thêm OPT mới, đổi tên thành `DNSLAB`.
   * Cấu hình IPv4 Static: `10.9.0.1/24`.
   * **Lưu ý cấu hình:** Không chọn `IPv4 Upstream Gateway` (ngăn pfSense nhận nhầm thành WAN làm lỗi NAT).

### 4.2. Cấu hình Luật Tường lửa (Firewall Rules) cho Interface DNSLAB

Tại mục **Firewall → Rules → DNSLAB**, đã thiết lập 5 rule theo đúng thứ tự ưu tiên:

|STT|Action|Source|Destination|Port|Mục đích kiểm soát|
|-|-|-|-|-|-|
|1|**Pass**|`10.9.0.10`|`172.16.6.20`|UDP/TCP 53|Cho phép DNS-SERVER forward truy vấn sang AUTH-DNS ở DMZ|
|2|**Pass**|`10.9.0.0/24`|`172.16.6.10`|TCP 80|Cho phép Client/Victim truy cập Webserver chuẩn trong DMZ|
|3|**Block + Log**|`10.9.0.0/24`|`10.6.0.0/24`|Any|Cách ly, bảo vệ mạng LAN-DOMAIN / Active Directory LAB 8|
|4|**Block + Log**|`10.9.0.0/24`|`192.168.56.0/24`|Any|Chặn truy cập trái phép vào mạng quản trị MGMT|
|5|**Block + Log**|`10.9.0.0/24`|Any|Any|Cách ly hoàn toàn dải DNSLAB ra khỏi Internet trong pha tấn công|

### 4.3. Thiết lập Máy chủ DNS BIND9 (10.9.0.10)

1. Đặt IP tĩnh qua Netplan: `10.9.0.10/24`, Gateway: `10.9.0.1`.
2. Sao lưu và cấu hình `/etc/bind/named.conf.options`:

   * Lắng nghe trên cổng 53 của `10.9.0.10`.
   * Cấu hình `allow-query { 10.9.0.0/24; localhost; };`.
   * Thêm ngoại lệ tắt cookie cho máy chủ giả lập: `server 172.16.6.20 { send-cookie no; };` nhằm tái lập kịch bản on-path spoofing trong môi trường lab.
3. Cấu hình `/etc/bind/named.conf.local`:

   * Khai báo Master zone cho `lab9.test` và Reverse zone `9.10.in-addr.arpa`.
   * Khai báo Forward zone cho `external.lab9.test` trỏ forwarders về `172.16.6.20`.
4. Kiểm tra tính hợp lệ cú pháp:

```bash
   sudo named-checkconf
   sudo named-checkzone lab9.test /etc/bind/db.lab9.test
   sudo named-checkzone 9.10.in-addr.arpa /etc/bind/db.10.9.0
   sudo systemctl restart bind9
   ```

### 4.4. Cấu hình Máy Victim (10.9.0.100) \& Attacker Kali (10.9.0.200)

* **Victim:** Cấu hình Netplan nhận IP `10.9.0.100/24`, DNS duy nhất là `10.9.0.10`. Kiểm tra qua `resolvectl status`.
* **Kali:** Cấu hình IP `10.9.0.200/24`. Bật **Promiscuous Mode = Allow VMs** trên card VirtualBox để có thể bắt gói unicast Layer 2 giữa Victim và DNS-SERVER.
* **Fake Web:** Khởi chạy HTTP Server giả mạo trên Kali:

```bash
  sudo python3 -m http.server 80 --directory \~/LAB9/fake\_site
  ```

### 4.5. Thiết lập Baseline \& Khởi động Authoritative DNS (172.16.6.20)

* Trên APP-TEST (172.16.6.20), chạy `sudo python3 auth\_dns\_delayed.py --bind 172.16.6.20 --delay 1.5` (tạo độ trễ 1.5s để mô phỏng trễ mạng đường dài, tạo điều kiện cho cuộc đua DNS).
* **Kiểm tra Baseline từ Victim:**

  * `dig @10.9.0.10 www.external.lab9.test A +short` trả về đúng IP chuẩn `172.16.6.10`.
  * `curl http://www.external.lab9.test` mở được website chuẩn ở DMZ.

\---

## 5\. Kết quả thực nghiệm từng Task

### 5.1. Task 1 – Điều hướng bằng tệp tin `/etc/hosts`

* Thêm bản ghi `10.9.0.200 www.lab9.test` vào `/etc/hosts` của máy Victim.
* **Kết quả kiểm thử:**

  * `curl http://www.lab9.test` → Trả về trang Web giả mạo của Attacker (`10.9.0.200`).
  * `dig @10.9.0.10 www.lab9.test A +short` → Vẫn trả về IP gốc `10.9.0.10`.
* **Giải thích:** `curl` sử dụng trình phân giải hệ thống NSS (`/etc/nsswitch.conf` ưu tiên files trước dns), trong khi `dig` mở socket UDP gửi truy vấn trực tiếp đến DNS-SERVER.
* Đã khôi phục lại tệp hosts gốc và xóa cache bằng `resolvectl flush-caches`.

### 5.2. Task 2 – Giả mạo phản hồi DNS trực tiếp (Direct DNS Spoofing)

* Trên Kali, chạy script Scapy bắt gói truy vấn từ Victim:

```bash
  sudo python3 dns\_spoof\_lab9.py --mode victim --iface eth1
  ```

* Từ Victim, gửi truy vấn: `resolvectl query www.external.lab9.test`.
* **Kết quả kiểm thử:**

  * Script trên Kali bắt được TXID và Port nguồn của Victim, gửi ngay gói phản hồi giả mạo với IP nguồn giả là `10.9.0.10` trả về IP `10.9.0.200`.
  * Gói phản hồi giả thắng cuộc đua do server thật bị trễ 1.5s.
  * Victim truy cập `curl http://www.external.lab9.test` hiển thị trang phishing của Kali.
  * Wireshark ghi nhận rõ ràng: Gói Query, Gói Giả mạo (Forged Response đến trước) và Gói Hợp lệ (Legitimate Response đến sau bị bỏ qua).

### 5.3. Task 3 – Đầu độc bộ nhớ đệm (DNS Cache Poisoning trên BIND9)

* Xóa sạch cache BIND9: `sudo rndc flush`.
* Trên Kali, chạy script ở chế độ đầu độc resolver:

```bash
  sudo python3 dns\_spoof\_lab9.py --mode resolver --iface eth1
  ```

* Từ Victim kích hoạt phân giải tên miền ngoài `www.external.lab9.test`.
* **Kết quả kiểm thử:**

  * BIND9 forward truy vấn sang DMZ. Kali bắt được gói forward này và gửi gói phản hồi giả danh `172.16.6.20` về cho BIND9.
  * BIND9 chấp nhận dữ liệu độc hại và nạp `www.external.lab9.test -> 10.9.0.200` vào Cache RAM với TTL = 120s.
  * **Chứng minh:** Tắt hẳn script Scapy trên Kali. Máy Victim xóa cache cục bộ rồi truy vấn lại vẫn ra IP độc hại `10.9.0.200`.
  * Thực thi `sudo rndc dumpdb -cache` trên BIND9, kiểm tra tệp `/var/cache/bind/named\_dump.db` xác nhận sự tồn tại của bản ghi `10.9.0.200`.
  * Sau khi kiểm thử, thực hiện `sudo rndc flush` để đưa hệ sinh thái DNS về trạng thái sạch.

\---

## 6\. Trạng thái kiểm thử tổng hợp

|Nội dung kiểm thử|Trạng thái|Bằng chứng / Ghi chú kỹ thuật|
|-|-|-|
|**Cấu hình pfSense DNSLAB \& Rules**|**PASS**|Gán IP 10.9.0.1/24, cấu hình đúng 5 firewall rules cô lập|
|**Dịch vụ BIND9 khởi động sạch**|**PASS**|`named-checkconf` và `named-checkzone` không lỗi, listen UDP/53|
|**Kiểm tra Baseline trước tấn công**|**PASS**|`www.external.lab9.test` phân giải chính xác về `172.16.6.10`|
|**Task 1: Hosts File Redirection**|**PASS**|Curl bị trỏ về 10.9.0.200 trong khi Dig trực tiếp vẫn ra IP gốc|
|**Task 2: Direct DNS Spoofing**|**PASS**|Kali inject gói giả thắng cuộc đua, Victim curl ra Web Kali|
|**Task 2: Wireshark Packet Inspection**|**PASS**|Bắt đủ 3 gói: Query, Forged Response (trước), Legitimate (sau)|
|**Task 3: BIND9 Cache Poisoning**|**PASS**|BIND9 bị đầu độc; tắt script Kali mà Victim query vẫn ra Web giả|
|**Task 3: BIND9 Cache Dump Verification**|**PASS**|Tệp `named\_dump.db` ghi nhận bản ghi `10.9.0.200` (TTL 120s)|
|**Khôi phục trạng thái sạch**|**PASS**|`rndc flush` thành công, hệ thống phân giải chuẩn về `172.16.6.10`|
|**Hardening: Tắt Promiscuous Mode**|**PASS**|Đã chuyển card mạng Kali từ `Allow VMs` về `Deny` sau bài lab|

\---

## 7\. Giải đáp 10 câu hỏi phân tích cốt lõi (Mục L)

1. **Tại sao `/etc/hosts` ảnh hưởng trình duyệt nhưng `dig` vẫn trả IP gốc?**

   * Ứng dụng phổ thông dùng hàm C `getaddrinfo()` tuân theo thứ tự trong `/etc/nsswitch.conf` (`hosts: files dns`), nên đọc tệp `hosts` trước. Tiện ích `dig` là công cụ chẩn đoán chuyên dụng, bỏ qua thư viện NSS và mở socket UDP gửi truy vấn trực tiếp đến cổng 53 của DNS-SERVER.
2. **Những trường nào bắt buộc phải khớp để phản hồi DNS giả mạo được chấp nhận?**

   * IP nguồn/đích, Cổng UDP nguồn/đích (khớp đúng cổng ngẫu nhiên của bên gửi), 16-bit Transaction ID, phần Question Section (QNAME và QTYPE), cờ phản hồi (QR=1, Opcode=0, RCODE=0) và phải đến trước gói tin hợp lệ.
3. **Tại sao Task 2 chỉ ảnh hưởng một Victim, trong khi Task 3 ảnh hưởng toàn mạng?**

   * Task 2 bắn phản hồi giả trực tiếp vào IP của Victim, chỉ lưu trong cache cục bộ của máy đó. Task 3 đầu độc trực tiếp bộ nhớ đệm dùng chung (Shared Cache) của máy chủ BIND9, khiến mọi client trong mạng hỏi BIND9 đều nhận kết quả độc hại.
4. **Vai trò của TTL trong tấn công DNS Cache Poisoning?**

   * Quyết định độ dài thời gian bản ghi độc hại được lưu giữ trong RAM của máy chủ BIND9 mà không cần truy vấn lại. TTL càng cao, thời gian chiếm quyền điều hướng nạn nhân càng lâu.
5. **Tại sao VirtualBox Promiscuous Mode = Allow VMs là bắt buộc trong bài?**

   * Switch ảo Internal Network thông thường chỉ gửi khung tin Unicast đến đúng card mạng có MAC đích tương ứng. Chế độ `Allow VMs` cho phép card mạng Kali nghe lén toàn bộ lưu lượng Layer 2 đi qua switch ảo, từ đó mới bắt được TXID và Port ngẫu nhiên để giả mạo.
6. **Vì sao dùng mạng DNSLAB riêng thay vì tấn công trực tiếp DC01 LAN-DOMAIN?**

   * Bảo vệ kiến trúc Active Directory / Kerberos của bài LAB 8 khỏi nguy cơ tê liệt xác thực; đồng thời tạo môi trường sandbox sạch, không bị nhiễu bởi các gói tin broadcast của Windows để phân tích gói tin chính xác nhất.
7. **Khác biệt giữa Authoritative DNS, Recursive Resolver và Caching Resolver?**

   * *Authoritative DNS:* Nắm giữ cơ sở dữ liệu gốc (zone files) của tên miền và trả lời kết quả chính thức cho zone đó.
   * *Recursive Resolver:* Đóng vai trò máy chủ trung gian đi tìm kiếm câu trả lời thay cho client bằng cách duyệt qua hệ thống cây phân giải DNS toàn cầu.
   * *Caching Resolver:* Lưu trữ tạm thời các câu trả lời nhận được vào RAM theo giá trị TTL để phản hồi tức thì cho các truy vấn lặp lại tiếp theo.
8. **DNSSEC giải quyết vấn đề gì và tại sao zone `.test` unsigned trong lab không được bảo vệ?**

   * DNSSEC dùng chữ ký số (RRSIG/DNSKEY) và chuỗi tin cậy (Chain of Trust) để đảm bảo tính toàn vẹn và xác thực nguồn gốc bản ghi. Zone `.test` trong bài là zone nội bộ chưa được ký số (unsigned) và không có liên kết chuỗi tin cậy với Root Zone, do đó resolver coi là không an toàn (Insecure) và chấp nhận gói tin không cần chữ ký.
9. **Tại sao chỉ chặn UDP/53 là không đủ để kiểm soát DNS hiện đại?**

   * DNS còn sử dụng TCP/53 (cho gói tin lớn, DNSSEC, Zone Transfer), DNS over HTTPS (DoH qua TCP/443), DNS over TLS (DoT qua TCP/853) và DNS over QUIC (DoQ qua UDP/853) được mã hóa tinh vi.
10. **Bốn biện pháp củng cố bảo mật (Hardening) sau khi hoàn thành bài lab?**

    * (1) Bật lại DNS COOKIE (RFC 7873); (2) Đặt cấu hình Promiscuous Mode trên hypervisor/vSwitch về `Deny`; (3) Thiết lập luật DNS Egress Filtering trên pfSense chỉ cho phép IP DNS-SERVER ra ngoài; (4) Phân đoạn mạng cô lập (VLAN / Micro-segmentation) tách biệt client và server ở Layer 2.

\---

## 8\. Hướng dẫn nộp bài \& Bổ sung ảnh minh chứng

Trước khi nộp bài báo cáo Word (`\[Mã lớp]-Lab9\_MSSV-HoTen.docx`), sinh viên cần chụp ảnh thực tế từ các máy ảo của mình và chèn vào các khung hình tương ứng:

* \[x] Hình 1: Topology mạng và trạng thái gán card VirtualBox.
* \[x] Hình 2: Giao diện pfSense Firewall Rules trên tab DNSLAB.
* \[x] Hình 3: Terminal kiểm tra IP (`ip -br addr`) và kiểm tra cú pháp BIND9 (`named-checkconf`).
* \[x] Hình 4: Giao diện Fake Webserver chạy trên Kali Linux (`10.9.0.200`).
* \[x] Hình 5: Truy vấn Baseline thành công ra IP DMZ `172.16.6.10`.
* \[x] Hình 6: Task 1 - So sánh curl trỏ về Kali nhưng dig vẫn ra kết quả DNS gốc.
* \[x] Hình 7: Task 2 - Wireshark bắt gói Query và Forged Response.
* \[x] Hình 8: Task 3 - Kết quả dump cache BIND9 (`named\_dump.db`) chứa IP độc hại.
* \[x] Hình 9: Khôi phục sạch cache BIND9 (`rndc flush`) và test lại.
* \[x] Hình 10: Thiết lập VirtualBox trả Promiscuous Mode về `Deny`.

