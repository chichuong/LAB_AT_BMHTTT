\# LAB 1: TELNET VÀ SSH



\## 1. Thông tin sinh viên



\- Họ và tên: Lý Chí Chương

\- Mã số sinh viên: 1150080128



\## 2. Tên bài Lab



\*\*Tìm hiểu và so sánh Telnet và SSH\*\*



\## 3. Nội dung đã thực hiện



\- Cài đặt và cấu hình Ubuntu Server trên VirtualBox.

\- Cấu hình mạng giữa Kali Linux và Ubuntu Server.

\- Kiểm tra kết nối giữa Kali Linux và Ubuntu Server.

\- Cài đặt và cấu hình dịch vụ Telnet trên Ubuntu Server.

\- Kết nối từ Kali Linux đến Ubuntu Server bằng Telnet.

\- Sử dụng Wireshark để bắt và phân tích lưu lượng Telnet trên TCP port 23.

\- Sử dụng chức năng Follow TCP Stream để quan sát dữ liệu của phiên Telnet.

\- Cài đặt và kiểm tra dịch vụ SSH trên Ubuntu Server.

\- Kết nối từ Kali Linux đến Ubuntu Server bằng SSH.

\- Sử dụng Wireshark để bắt và phân tích lưu lượng SSH trên TCP port 22.

\- Sử dụng Follow TCP Stream để so sánh dữ liệu giữa Telnet và SSH.

\- So sánh mức độ bảo mật của Telnet và SSH.



\## 4. Kết quả thực hiện



\- Kali Linux và Ubuntu Server kết nối thành công trong mạng Lab.

\- Telnet hoạt động thành công trên TCP port 23.

\- SSH hoạt động thành công trên TCP port 22.

\- Wireshark bắt được lưu lượng của cả Telnet và SSH.

\- Với Telnet, nội dung phiên truyền qua mạng có thể được quan sát và khôi phục bằng Wireshark.

\- Với SSH, Wireshark bắt được các gói tin nhưng nội dung phiên sau khi thiết lập kết nối được mã hóa.

\- Kết quả cho thấy SSH có khả năng bảo vệ thông tin tốt hơn Telnet và phù hợp hơn cho việc quản trị máy chủ từ xa.



\## 5. Lưu ý khi kiểm tra hoặc chạy lại bài Lab



\- Khởi động cả hai máy ảo Kali Linux và Ubuntu Server trước khi thực hiện.

\- Hai máy phải được kết nối vào cùng mạng Lab.

\- Địa chỉ IP Ubuntu Server sử dụng trong bài thực hành: `192.168.56.102`.

\- Kiểm tra kết nối từ Kali Linux đến Ubuntu Server:



&#x20; ```bash

&#x20; ping -c 4 192.168.56.102

