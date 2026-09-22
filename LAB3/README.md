# LAB 3 - NHẬN DIỆN VÀ ỨNG PHÓ CÁC MỐI ĐE DỌA ĐẾN AN TOÀN THÔNG TIN

## 1. Thông tin sinh viên

- Họ và tên: Lý Chí Chương
- MSSV: 1150080128
- Tên lab: Lab 3 - Nhận diện và ứng phó các mối đe dọa đến an toàn thông tin
- Môn học: Thực hành An toàn hệ thống thông tin
- Năm học: 2026-2027

## 2. Phiên bản môi trường

Môi trường thực hành được triển khai trên máy ảo nhằm cô lập các hoạt động của bài lab với máy thật.

- VMware Workstation Pro: 26H1
- Hệ điều hành máy ảo: Windows 11 Pro x64
- Windows Build: [điền build thực tế của VM]
- CPU máy ảo: 2 vCPU
- RAM máy ảo: 6 GB
- Disk: 64 GB
- Network: Host-only
- Microsoft Defender: Enabled
- Windows Firewall: Enabled
- PowerShell: 5.1
- Python: 3.14.7
- Wireshark: 4.6.8
- Npcap: Đã cài đặt
- Sysmon: 15.22
- Autoruns: 14.3
- Process Explorer: 17.14

## 3. Cách dựng môi trường

Đầu tiên, em tạo một máy ảo Windows 11 Pro x64 bằng VMware Workstation Pro. Máy ảo được cấp 2 vCPU, 6 GB RAM và 64 GB ổ đĩa.

Sau khi cài Windows và VMware Tools, card mạng của máy ảo được chuyển sang chế độ Host-only để cô lập môi trường thực hành. Trong những bước cần Internet như tải công cụ hoặc kiểm tra HTTPS, máy ảo được chuyển tạm thời sang NAT và sau đó chuyển lại Host-only.

Snapshot có tên `LAB3_CLEAN_20260914` được tạo trước khi tiến hành các nội dung thực hành.

Thư mục làm việc chính được tạo tại:

C:\LAB3

Bao gồm các thư mục:

- `C:\LAB3\Evidence`
- `C:\LAB3\Tools`
- `C:\LAB3\Downloads`
- `C:\LAB3\Assets`

Các công cụ cần thiết gồm Python, Wireshark/Npcap, Sysmon, Autoruns và Process Explorer được cài đặt và kiểm tra trước khi thực hành.

Microsoft Defender, Windows Firewall và trạng thái hệ thống cũng được kiểm tra để tạo baseline trước khi thực hiện các tình huống.

## 4. Các tình huống đã thực hiện

### TH1 - Nhận diện tài sản, mối đe dọa và rủi ro

Xác định các tài sản cần bảo vệ trên hệ thống, xây dựng bảng Risk Register và phân loại các tình huống theo năm nhóm nguồn đe dọa: hành động vô ý, hành động cố ý, thảm họa tự nhiên, lỗi kỹ thuật và lỗi quản lý.

Kết quả: PASS

### TH2 - Kiểm tra khả năng phát hiện malware bằng EICAR

Sử dụng chuỗi kiểm thử EICAR để kiểm tra hoạt động của Microsoft Defender. Defender đã phát hiện mẫu thử và ghi nhận sự kiện trong Protection History.

Không tắt Defender và không tạo exclusion để cho mẫu chạy.

Kết quả: PASS

### TH3 - Password Attack và Keylogging

Tạo tài khoản local `lab3user`, thực hiện các lần xác thực đúng và sai để tạo Security Event.

Theo dõi các Event ID:

- 4624: đăng nhập thành công
- 4625: đăng nhập thất bại
- 4648: sử dụng credential được cung cấp rõ ràng

Sau đó thực hiện đổi mật khẩu của `lab3user`, kiểm tra mật khẩu cũ không còn sử dụng được và credential mới hoạt động.

Kết quả: PASS

### TH4 - Persistence và phát hiện Backdoor

Cài đặt Sysmon và sử dụng Autoruns để quan sát các cơ chế persistence.

Tạo hai persistence artifact phục vụ thực hành:

- Registry Run: `LAB3_Run_Demo`
- Scheduled Task: `LAB3_Persistence_Demo`

Sử dụng Sysmon và Autoruns để phát hiện các thay đổi.

Tiếp theo, tạo Python HTTP server chỉ lắng nghe trên:

127.0.0.1:8080

Sử dụng `Get-NetTCPConnection` và Process Explorer để xác định PID và ánh xạ cổng đang Listen với tiến trình `python.exe`.

Kết quả: PASS

### TH5 - Sniffing HTTP và HTTPS

Sử dụng Wireshark để bắt traffic HTTP trên loopback interface.

Với HTTP, các tham số thử nghiệm như `lab_user` và `lab_code` có thể quan sát trực tiếp trong HTTP request.

Sau đó thực hiện HTTPS capture trên TCP port 443. Nội dung ứng dụng không xuất hiện dưới dạng plaintext như trường hợp HTTP mà được bảo vệ bởi TLS.

Kết quả: PASS

### TH6 - DoS/DDoS và Mail Bombing

Chạy `local_load_test.py` để tạo tải thử nghiệm an toàn tới dịch vụ local tại `127.0.0.1:8080`.

Phân tích `ddos_sample.csv` để thống kê số lượng request theo SourceIP và quan sát đặc điểm nhiều nguồn của DDoS.

Phân tích `mailbomb_sample.csv` để thống kê số lượng email theo Sender, tổng dung lượng và dung lượng trung bình của email.

Kết quả: PASS

### TH7 - Phishing và Social Engineering

Phân tích offline file `phishing_email.txt` và xác định các dấu hiệu đáng ngờ như:

- Tạo cảm giác khẩn cấp
- Giả mạo uy tín người gửi
- Domain đáng ngờ
- Reply-To khác From
- Yêu cầu cung cấp credential hoặc truy cập liên kết

Tiếp tục phân tích các tình huống trong `social_engineering_cases.csv` và phân loại các hình thức Social Engineering.

Không truy cập các domain hoặc liên kết xuất hiện trong mẫu phishing.

Kết quả: PASS

## 5. Cleanup và Recovery Verification

Sau khi hoàn thành các tình huống, em tiến hành dọn dẹp môi trường thực hành.

Các thành phần được loại bỏ gồm:

- Registry Run `LAB3_Run_Demo`
- Scheduled Task `LAB3_Persistence_Demo`
- Python listener tại port 8080
- Local account `lab3user`

Sau khi cleanup, hệ thống được kiểm tra lại để xác nhận các artifact trên không còn tồn tại.

Microsoft Defender và Real-time Protection vẫn ở trạng thái hoạt động.

Cuối cùng, SHA-256 được tạo cho các file trong thư mục Evidence nhằm hỗ trợ kiểm tra tính toàn vẹn của bằng chứng.

Kết quả: PASS

## 6. Lỗi gặp phải và cách khắc phục

### Lỗi 1 - Sử dụng sai Windows ISO

Ban đầu em sử dụng Windows 11 ARM64 trong khi máy host sử dụng kiến trúc x86-64, khiến VMware báo lỗi không tương thích kiến trúc.

Cách khắc phục: tải lại Windows 11 x64 ISO và tạo lại máy ảo đúng kiến trúc.

### Lỗi 2 - Máy ảo không boot được từ ISO

VMware xuất hiện EFI Network Timeout do máy ảo chưa boot đúng từ file ISO.

Cách khắc phục: kiểm tra lại CD/DVD trong VMware Settings, gắn Windows 11 ISO và khởi động lại máy ảo.

### Lỗi 3 - Không có Internet trong quá trình cài Windows

Do Network Adapter đang đặt ở chế độ Host-only nên Windows OOBE không có kết nối Internet.

Cách khắc phục: sử dụng chế độ thiết lập offline để hoàn thành Windows. Khi cần tải công cụ, chuyển tạm thời sang NAT và sau đó trả về Host-only.

### Lỗi 4 - Lệnh runas không nhận mật khẩu

Khi thực hiện TH3, lệnh `runas` báo lỗi `Unable to acquire user password`.

Cách khắc phục: kiểm tra tài khoản `lab3user`, đặt yêu cầu mật khẩu và sử dụng PowerShell `Get-Credential` kết hợp `Start-Process -Credential` để thực hiện kiểm tra credential trong môi trường lab.

### Lỗi 5 - Sysmon không tìm thấy file cấu hình

Sysmon báo:

`Failed to open xml configuration`

Nguyên nhân là file `sysmon-lab.xml` không nằm tại đường dẫn dự kiến ban đầu.

Cách khắc phục: tìm lại file bằng `Get-ChildItem`, xác định đường dẫn thực tế:

C:\LAB3\Downloads\lab3_assets\sysmon-lab.xml

Sau đó cài Sysmon bằng đúng đường dẫn của file cấu hình và kiểm tra `Microsoft-Windows-Sysmon/Operational` đã được bật.

### Lỗi 6 - Wireshark không hiển thị interface

Wireshark hiển thị thông báo không có packet capture driver nên không xuất hiện danh sách interface.

Cách khắc phục: cài đặt Npcap, đóng và mở lại Wireshark với quyền Administrator. Sau khi cài Npcap, Wireshark nhận được Ethernet và loopback interface bình thường.

## 7. Kết quả tổng kết

| Nội dung | Kết quả |
|---|---|
| Chuẩn bị môi trường | PASS |
| Baseline hệ thống | PASS |
| TH1 - Threat/Risk Identification | PASS |
| TH2 - EICAR/Defender | PASS |
| TH3 - Authentication/Keylogging | PASS |
| TH4 - Persistence/Backdoor Detection | PASS |
| TH5 - HTTP/HTTPS Sniffing | PASS |
| TH6 - DoS/DDoS/Mail Bombing | PASS |
| TH7 - Phishing/Social Engineering | PASS |
| Cleanup và Verification | PASS |

**Kết quả chung: PASS**