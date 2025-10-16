# kết nối Hybrid Cloud (on-premise to Cloud)
Trong xu thế phát triển hiện nay, hầu hết các tổ chức và doanh nghiệp đều hướng tới việc kết hợp hạ tầng On-Premises (tại chỗ) với các dịch vụ điện toán đám mây công cộng như AWS, AZURE ... để tạo ra một môi trường Hybrid Cloud linh hoạt. Trong demo này, thay vì sử dụng dịch vụ Cloud VPN Gateway, hệ thống sẽ triển khai kết nối trực tiếp đến một VM trên AZURE (có cả IP Private và IP Public) để đóng vai trò đầu mối giao tiếp. Kết nối bảo mật được thiết lập thông qua pfSense (chạy tại On-Premises) và Tailscale VPN (hỗ trợ remote access và định tuyến giữa hai bên).
Khi đường hầm được thiết lập, hệ thống On-Premises có thể coi AZURE VM như một thành phần trong mạng LAN nội bộ. Điều này cho phép:
- Các máy trong LAN On-Premises truy cập trực tiếp tài nguyên trên AZURE (ví dụ: Web Server cài trên VM).
- VM trên AZURE có thể kết nối ngược về hạ tầng On-Premises (Windows Server, Client trong LAN, Router trong EVE-NG).
- Người quản trị có thể truy cập từ xa vào pfSense hoặc VM GCP mà không cần Public IP cố định nhờ Tailscale.

 1. Tạo Vnet trên AZURE
 ![draw]( https://raw.githubusercontent.com/buivandat2k4-jpg/cloud-networking/refs/heads/main/z7124487135755_da61498531cd91b0d7f55ff5a9852556.jpg)

 ```bash
 Trước tiên ta cần tạo resource group
 Xong sẽ tạo vnet nằm bên trong cái group đó
 tạo dải ip cho vnet: 10.0.0.0/16

 ```
 2. Tạo VM trên AZURE 
![draw](https://raw.githubusercontent.com/buivandat2k4-jpg/cloud-networking/refs/heads/main/z7124599739175_bcdeaedf8c2216fc11bf5eafe4be51a5.jpg)

 ```bash
- Các bước tạo máy áo VM 
+ chọn vùng cùng với vùng group
+ đặt tên
+ Availability options: No infrastructure redundancy required
+ Security type: Standard
+ hđh: chọn windown
+ Các mục khác để im
+ Đặt Tk, MK cho máy ảo
+ Phần Public inbound port: để allow
+ Select inbound port : chọn SSH, RDP, HTTP
+ Next
+ Ở phần Networking
+ chọn vnet đã tạo nó sẽ tự chọn mấy mục dưới như subnet con và ip public
+ NIC network security group: basic
+ Public inbound ports: allow 
+ Select inbound ports: HTTP (80), SSH (22), RDP (3389)
+ create
-Tạo xong ta sẽ có 1 vm như ảnh và có ip 10.0.0.4 
 ```
 3. Cài Pfsense trên eve và tải Tailscale trong Pfsense
 ![draw](https://raw.githubusercontent.com/buivandat2k4-jpg/cloud-networking/refs/heads/main/z7124600089226_0ced2c97720ac8dfb67910458a8292a7.jpg)
 ```bash
- Cài Pfsense trên eve thì mọi người có thể lên Youtube để xem
- Cài đặt pfSense
- Cài pfSense trên VMware hoặc máy thật.
- Thiết lập:
	+ WAN: nối ra Internet (có thể qua NAT của VMware).
	+ LAN: nối vào mạng nội bộ (ví dụ 192.168.1.0/24).
→ Sau bước này pfSense hoạt động như router/firewall bình thường, client trong LAN có thể truy cập Internet.
Cài đặt Tailscale trên pfSense
- Truy cập pfSense WebGUI.
	+ Vào System → Package Manager → Available Packages.
	+ Tìm và cài Tailscale (nếu không có, cài qua shell: pkg install tailscale).
	+ sau khi cài xong thêm 1 số lệnh để bắt đầu chạy tailscale
 ```
 4. Cấu hình Firewall Rules
 ![draw](https://raw.githubusercontent.com/buivandat2k4-jpg/cloud-networking/refs/heads/main/z7124600352506_e325b2eb88cad6b41db66a8210aff634.jpg)
 ```bash
 Mục tiêu của bước này là đảm bảo rằng máy chủ VM trên AZURE (10.0.0.0/16) và máy trong LAN On-Premises (192.168.75.0/24) có thể giao tiếp trực tiếp qua IP private, thông qua đường hầm Tailscale.
Cấu hình Static Route
+  Truy cập: System → Routing → Static Routes
+ Thêm network của Tailscale (subnet 100.x.x.x hoặc dải mạng GCP 10.0.0.0/16).
Mục đích: Cho phép pfSense biết rằng muốn đi đến dải 10.0.0.0/16 thì phải chuyển qua interface Tailscale1. Nếu không thêm route, pfSense sẽ không biết gửi gói tin qua đâu.
Firewall Rules cho LAN
- Truy cập: Firewall → Rules → LAN
- Thêm 2 rule:
- Rule 1 (cho phép kết nối từ AZURE về LAN On-Premises)
	+ Protocol: any
	+ Source: 10.0.0.0/16
	+ Destination: LAN net (192.168.75.0/24)
	+ Port: any
	+ Gateway: any
- Rule 2 (cho phép toàn bộ lưu lượng khác đi ra ngoài)
	+ Protocol: any
	+ Source: any
	+ Destination: any
	+ Port: any
	+ Gateway: any
- Mục đích:
Rule 1 đảm bảo các máy trong AZURE có thể truy cập trực tiếp vào hệ thống LAN nội bộ.
Rule 2 giữ nguyên khả năng truy cập internet/bên ngoài cho các máy LAN.
Firewall Rules cho Interface TAILSCALE
- Truy cập: Firewall → Rules → TAILSCALE
- Thêm 2 rule:
- Rule 1 (cho phép từ AZURE truy cập LAN On-Premises qua Tailscale)
	+ Protocol: any
	+ Source: 10.0.0.0/16
	+ Destination: 192.168.75.0/24
	+ Port: any
- Rule 2 (cho phép tất cả các lưu lượng còn lại)
	+ Protocol: any
	+ Source: any
	+ Destination: any
	+ Port: any
- Mục đích:
	+ Rule 1 đảm bảo lưu lượng từ VM (10.0.0.0/16) qua Tailscale có thể đi 	đến mạng LAN nội bộ.
	+ Rule 2 là rule tổng, đảm bảo không bị chặn các kết nối khác trên interface 	Tailscale.

 ```
 5. Kích hoạt subnet routing
 ![draw](https://raw.githubusercontent.com/buivandat2k4-jpg/cloud-networking/refs/heads/main/z7124600571257_859a50e43b9afb444b8c5a8187f1f774.jpg)
 ![draw](https://raw.githubusercontent.com/buivandat2k4-jpg/cloud-networking/refs/heads/main/z7124606945969_e649f93c6963cfc33f28f3e08ebae2f8.jpg)
 
 ```bash
 Vào VPN -> chọn Tailscale -> chọn enable -> Routes nhâp dải ip muốn quảng bá
 Vào Web của Tailscale đăng nhập tài khoản -> chọn vào thiết bị Pfsense -> chọn edit route -> tích chọn các dải ip -> save
 ```
 6. Ping 
 ![draw](https://raw.githubusercontent.com/buivandat2k4-jpg/cloud-networking/refs/heads/main/z7124601421589_fb2cfde74868a50241704be9d4cbd892.jpg)
 ```bash
Ping thử xem kết nối đã thành công chưa
 ```
 