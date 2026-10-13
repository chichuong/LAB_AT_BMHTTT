# README - LAB 4: QUÉT MẠNG VỚI NMAP

## 1. Thông tin sinh viên

-   **Họ và tên:** Lý Chí Chương
-   **MSSV:** 1150080128
-   **Lớp:** CNPM2
-   **Tên lab:** LAB 4 - Quét mạng với Nmap

------------------------------------------------------------------------

## 2. Mục tiêu bài thực hành

-   Làm quen với Nmap và các kỹ thuật quét mạng cơ bản.
-   Phát hiện các host đang hoạt động trong mạng Host-Only.
-   Quét và so sánh các trạng thái cổng TCP/UDP.
-   So sánh TCP Connect Scan, SYN Scan, FIN, Xmas, NULL và ACK Scan.
-   Nhận diện dịch vụ, phiên bản dịch vụ và hệ điều hành của máy đích.
-   Sử dụng Nmap Scripting Engine (NSE) để thu thập thông tin SMB và
    kiểm tra lỗ hổng.
-   Xuất kết quả quét ra nhiều định dạng để lưu bằng chứng.
-   Thực hiện và so sánh kết quả trước/sau hardening.

------------------------------------------------------------------------

## 3. Môi trường thực hành

  Thành phần              Cấu hình / Phiên bản
  ----------------------- ----------------------
  Hypervisor              Oracle VirtualBox
  Máy quét                Kali Linux 2026.2
  Nmap                    7.99
  Máy đích chính          Metasploitable 2
  Máy phục vụ hardening   Ubuntu Server
  Kiểu mạng lab           Host-Only
  Subnet                  192.168.56.0/24
  Kali Linux              192.168.56.103
  Metasploitable 2        192.168.56.104
  Ubuntu Server           192.168.56.102

### Cấu hình mạng

Kali Linux có hai card mạng: - `eth0`: NAT - `10.0.2.15/24` - `eth1`:
Host-Only - `192.168.56.103/24`

Metasploitable 2 chỉ được kết nối vào mạng **Host-Only**, không sử dụng
Bridged Adapter để tránh đưa máy cố ý có lỗ hổng ra mạng thật.

Ubuntu Server: - `enp0s3`: NAT - `10.0.2.15/24` - `enp0s8`: Host-Only -
`192.168.56.102/24`

------------------------------------------------------------------------

## 4. Cách dựng môi trường

1.  Cài và mở Oracle VirtualBox.
2.  Import/khởi động Kali Linux 2026.2.
3.  Cấu hình Kali gồm NAT và Host-Only Adapter.
4.  Tạo Metasploitable 2 bằng file `Metasploitable.vmdk`.
5.  Cấu hình Metasploitable 2 với 1 GB RAM, 1 CPU và chỉ sử dụng
    Host-Only Adapter.
6.  Khởi động Metasploitable 2 và đăng nhập bằng tài khoản lab.
7.  Kiểm tra địa chỉ IP bằng `ip -br addr` trên Kali và `ifconfig` trên
    Metasploitable 2.
8.  Kiểm tra kết nối Kali -\> Metasploitable 2 bằng `ping`.
9.  Thực hiện toàn bộ thao tác quét từ Kali Linux.

------------------------------------------------------------------------

## 5. Các nội dung đã thực hiện

### 5.1. Kiểm tra Nmap và địa chỉ mạng

``` bash
nmap --version
ip -br addr
ping -c 4 192.168.56.104
```

**Kết quả:** PASS

-   Nmap hoạt động bình thường, phiên bản 7.99.
-   Kali Host-Only: `192.168.56.103/24`.
-   Metasploitable 2: `192.168.56.104/24`.
-   Hai máy liên lạc được với nhau, ping không mất gói.

### 5.2. Host Discovery

``` bash
sudo nmap -sn 192.168.56.0/24
```

**Kết quả:** PASS

Phát hiện 4 host đang hoạt động, trong đó: - `192.168.56.103`: Kali
Linux. - `192.168.56.104`: Metasploitable 2.

### 5.3. TCP Connect Scan

``` bash
nmap -sT 192.168.56.104
```

**Kết quả:** PASS

-   Open: 23 cổng.
-   Closed: 977 cổng.
-   Filtered: 0 cổng.
-   Thời gian thực nghiệm: 0.81 giây.

Một số cổng mở: `21/ftp`, `22/ssh`, `23/telnet`, `80/http`,
`445/microsoft-ds`, `3306/mysql`, `5432/postgresql`, `5900/vnc`.

### 5.4. SYN Scan

``` bash
sudo nmap -sS 192.168.56.104
```

**Kết quả:** PASS

-   Open: 23 cổng.
-   Closed: 977 cổng.
-   Filtered: 0 cổng.
-   Thời gian thực nghiệm: 0.89 giây.

Kết quả cổng tương đồng với `-sT`.

### 5.5. FIN / Xmas / NULL Scan

``` bash
sudo nmap -sF 192.168.56.104
sudo nmap -sX 192.168.56.104
sudo nmap -sN 192.168.56.104
```

**Kết quả:** PASS

  Kỹ thuật     Closed   Open/Filtered   Thời gian
  ---------- -------- --------------- -----------
  FIN             977              23      2.15 s
  Xmas            977              23      2.20 s
  NULL            977              23      2.12 s

`open|filtered` không được hiểu chắc chắn là cổng đang mở.

### 5.6. ACK Scan

``` bash
sudo nmap -sA 192.168.56.104
```

**Kết quả:** PASS

Nmap báo:

``` text
Not shown: 1000 unfiltered tcp ports (reset)
```

ACK Scan cho thấy 1000 cổng ở trạng thái `unfiltered`; kết quả này không
có nghĩa 1000 cổng đều đang mở.

### 5.7. UDP Scan

``` bash
sudo nmap -sU --top-ports 20 192.168.56.104
```

**Kết quả:** PASS

Một số kết quả đáng chú ý: - `53/udp`: open - domain - `68/udp`:
open\|filtered - dhcpc - `69/udp`: open\|filtered - tftp - `137/udp`:
open - netbios-ns - `138/udp`: open\|filtered - netbios-dgm

Thời gian quét: 14.81 giây.

### 5.8. Version Detection

``` bash
sudo nmap -sV 192.168.56.104
```

**Kết quả:** PASS

Một số dịch vụ được phát hiện:

  Port       Dịch vụ      Phiên bản
  ---------- ------------ -----------------------------------
  21/tcp     FTP          vsftpd 2.3.4
  22/tcp     SSH          OpenSSH 4.7p1 Debian 8ubuntu1
  80/tcp     HTTP         Apache httpd 2.2.8 (Ubuntu) DAV/2
  445/tcp    SMB          Samba smbd 3.X - 4.X
  3306/tcp   MySQL        MySQL 5.0.51a-3ubuntu5
  5432/tcp   PostgreSQL   PostgreSQL DB 8.3.0 - 8.3.7

### 5.9. OS Detection

``` bash
sudo nmap -O 192.168.56.104
```

**Kết quả:** PASS

Nmap nhận diện: - Device type: general purpose - Running: Linux 2.6.X -
OS details: Linux 2.6.9 - 2.6.33 - Network Distance: 1 hop

Đây là kết quả fingerprint/suy đoán của Nmap, không được xem là tuyệt
đối.

### 5.10. Aggressive Scan

``` bash
sudo nmap -A 192.168.56.104
```

**Kết quả:** PASS

Thu được đồng thời thông tin dịch vụ, phiên bản, OS fingerprint, NSE mặc
định và traceroute. Thời gian thực nghiệm: 23.73 giây.

### 5.11. NSE - SMB OS Discovery

``` bash
sudo nmap -p 445 --script smb-os-discovery 192.168.56.104
```

**Kết quả:** PASS

Thông tin thu được: - OS: Unix (Samba 3.0.20-Debian) - Computer name:
metasploitable - Domain name: localdomain - FQDN:
metasploitable.localdomain

### 5.12. NSE - Kiểm tra MS17-010

``` bash
sudo nmap -p 445 --script smb-vuln-ms17-010 192.168.56.104
```

**Kết quả:** PASS đối với việc thực thi phép kiểm tra, nhưng **kết quả
lỗ hổng không xác định**.

Script không trả về trạng thái `VULNERABLE`. Vì vậy không đủ cơ sở để
kết luận máy có hoặc không có lỗ hổng MS17-010.

### 5.13. Xuất kết quả

Normal output:

``` bash
sudo nmap -sV -O 192.168.56.104 -oN ket_qua.txt
```

XML:

``` bash
sudo nmap -sV -O 192.168.56.104 -oX ket_qua.xml
```

Grepable:

``` bash
sudo nmap -p 445 192.168.56.104 -oG smb.txt
grep "445/open" smb.txt
```

Chuyển XML sang HTML:

``` bash
xsltproc ket_qua.xml -o bao_cao.html
```

**Kết quả:** PASS

Các file kết quả đã được tạo và có thể sử dụng làm bằng chứng thực hành.

------------------------------------------------------------------------

## 6. Before / After Hardening

Máy phục vụ phần hardening: Ubuntu Server `192.168.56.102`.

Lệnh quét Before:

``` bash
sudo nmap -sV 192.168.56.102 -oN before.txt
```

**Trạng thái:** Đang thực hiện / chưa hoàn tất kết quả After.

Sau khi áp dụng biện pháp hardening, sử dụng lại cùng kiểu quét để tạo
`after.txt` và so sánh sự thay đổi của trạng thái cổng/dịch vụ.

------------------------------------------------------------------------

## 7. Tổng hợp PASS / FAIL

  Nội dung                     Kết quả
  ---------------------------- ----------------
  Nmap hoạt động               PASS
  Cấu hình Host-Only           PASS
  Kali ping Metasploitable 2   PASS
  Host Discovery `-sn`         PASS
  TCP Connect `-sT`            PASS
  SYN Scan `-sS`               PASS
  FIN Scan `-sF`               PASS
  Xmas Scan `-sX`              PASS
  NULL Scan `-sN`              PASS
  ACK Scan `-sA`               PASS
  UDP Scan `-sU`               PASS
  Version Detection `-sV`      PASS
  OS Detection `-O`            PASS
  Aggressive Scan `-A`         PASS
  NSE SMB OS Discovery         PASS
  NSE MS17-010                 INCONCLUSIVE
  Xuất TXT/XML/Grepable/HTML   PASS
  Before/After Hardening       ĐANG THỰC HIỆN

------------------------------------------------------------------------

## 8. Lỗi gặp phải và cách khắc phục

### Lỗi 1 - Chạy Nmap nhầm trên Ubuntu Server

**Hiện tượng:**

``` text
sudo: nmap: command not found
```

**Nguyên nhân:** Lệnh quét được nhập nhầm trên Ubuntu Server thay vì
Kali Linux.

**Khắc phục:** Không cần cài Nmap lên Ubuntu Server. Chuyển về Kali
Linux và thực hiện Nmap từ máy quét Kali tới địa chỉ
Ubuntu/Metasploitable.

### Lỗi 2 - Gõ sai lệnh `ifconfig`

**Hiện tượng:**

``` text
-bash: imconfig: command not found
```

**Nguyên nhân:** Gõ nhầm `ifconfig`.

**Khắc phục:** Nhập lại đúng:

``` bash
ifconfig
```

### Lưu ý khi đọc kết quả NSE

Script `smb-vuln-ms17-010` không xuất `VULNERABLE`. Không được suy diễn
kết quả này thành "máy an toàn" hoặc "đã vá". Chỉ ghi nhận rằng phép
kiểm tra hiện tại không đưa ra kết luận về lỗ hổng.

------------------------------------------------------------------------

## 9. Kết luận

Qua LAB 4, em đã dựng được môi trường quét mạng cô lập bằng Host-Only và
sử dụng Kali Linux/Nmap để thực hiện host discovery, TCP/UDP port
scanning, service/version detection, OS fingerprinting và NSE. Kết quả
cho thấy mỗi kỹ thuật quét trả lời một câu hỏi khác nhau; đặc biệt các
trạng thái `open`, `closed`, `filtered`, `open|filtered` và `unfiltered`
cần được diễn giải đúng theo kỹ thuật đang sử dụng.

Bài thực hành cũng cho thấy việc chỉ biết một cổng đang mở là chưa đủ để
đánh giá hệ thống. Thông tin về dịch vụ, phiên bản, hệ điều hành và kết
quả script cần được kết hợp, đồng thời các kết quả không xác định không
nên được suy diễn thành hệ thống an toàn.

------------------------------------------------------------------------

## 10. Các tệp bằng chứng

Các tệp chính được tạo trong quá trình thực hành:

``` text
README.md
ket_qua.txt
ket_qua.xml
smb.txt
bao_cao.html
before.txt
```

Sau khi hoàn tất phần hardening bổ sung:

``` text
after.txt
```

Các ảnh chụp màn hình được lưu kèm thư mục LAB4 để chứng minh các bước
thực hành và kết quả quan sát trực tiếp.
