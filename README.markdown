# Cấu hình tối ưu cho môi trường Lab trên Dell R630 với ESXi 7 (Prometheus, Grafana, Chi tiết mạng)

## 1. Mục tiêu
- Cấu hình **Prometheus** và **Grafana** trên Windows Server 2022 để giám sát tài nguyên (CPU, RAM, disk, mạng) của ESXi, pfSense, n8n, GitLab.
- Làm rõ cấu hình mạng trên ESXi, sử dụng **vSwitch0** để tạo Port Group cho WAN và Internal (LAN, VLAN 10).
- Đảm bảo n8n và GitLab chạy bằng Docker, tự khởi động khi VM khởi động.
- Đảm bảo tất cả dải mạng (WAN: 192.168.1.x/24, LAN: 172.168.1.x/24, VLAN 10: 10.0.0.x/24) truy cập Internet.
- Chuẩn bị cho cụm Kubernetes (3 master, 3 worker).
- Sử dụng 1 cổng NIC 1Gb (NIC0), không có switch hỗ trợ VLAN.

## 2. Tối ưu hóa phần cứng
- **CPU**: Dual Intel Xeon E5-2600 series (12-16 cores) để hỗ trợ ~18 vCPU.
- **RAM**: Tối thiểu 64GB (khuyến nghị 128GB) cho pfSense (4GB), Windows Server (12GB), n8n (6GB), GitLab (12GB), K8s (~36GB).
- **Lưu trữ**: RAID 5/6 với SSD (tối thiểu 1TB) cho tốc độ I/O cao.
- **NIC**: 1 cổng NIC 1Gb, tối ưu băng thông bằng QoS trên pfSense.

## 3. Cấu hình mạng trên ESXi
### 3.1. Tổng quan
- Sử dụng **vSwitch0** (vSwitch mặc định trên ESXi) để quản lý tất cả lưu lượng mạng, vì chỉ có 1 NIC (NIC0).
- Tạo 2 Port Group trên **vSwitch0**:
  - **PG-WAN**: Dùng cho dải WAN (192.168.1.x/24), không gắn VLAN ID.
  - **PG-Internal**: Dùng cho LAN (172.168.1.x/24) và VLAN 10 (10.0.0.x/24), không gắn VLAN ID (pfSense xử lý VLAN tagging ảo).

### 3.2. Các bước cấu hình trên ESXi Web Client
1. **Truy cập ESXi Web Client**:
   - Mở trình duyệt trên Windows Server, truy cập `https://192.168.1.10`.
   - Đăng nhập bằng tài khoản root của ESXi.
2. **Tạo vSwitch0 (nếu chưa có)**:
   - Vào **Networking > Virtual Switches**.
   - Kiểm tra **vSwitch0** (mặc định được tạo khi cài ESXi).
   - Gắn NIC0 (vmnic0) vào vSwitch0:
     - Chọn **vSwitch0** > **Edit Settings** > Tab **Physical NICs** > Thêm vmnic0.
3. **Tạo Port Group**:
   - Vào **Networking > Port Groups** > **Add Port Group**.
   - Tạo **PG-WAN**:
     - Name: PG-WAN
     - VLAN ID: 0 (không gắn VLAN ID, dành cho WAN).
     - Virtual Switch: vSwitch0
   - Tạo **PG-Internal**:
     - Name: PG-Internal
     - VLAN ID: 0 (pfSense sẽ xử lý VLAN 10 ảo).
     - Virtual Switch: vSwitch0
4. **Tối ưu hóa**:
   - Vào **vSwitch0** > **Edit Settings** > Tab **Security**:
     - Bật **Promiscuous Mode** để pfSense xử lý VLAN tagging ảo.
   - Đặt **MTU** thành 9000 (nếu router hỗ trợ jumbo frames) trong **vSwitch0 Properties** > **Advanced**.
5. **Gán IP cho ESXi**:
   - Vào **Networking > VMkernel NICs** > **vmk0** > **Edit Settings**.
   - Gán IP tĩnh: 192.168.1.10/24, gateway 192.168.1.1, DNS 8.8.8.8.

### 3.3. Lưu ý
- **vSwitch0** được sử dụng vì đây là cấu hình mặc định và phù hợp khi chỉ có 1 NIC.
- **PG-Internal** không cần VLAN ID trên ESXi vì pfSense sẽ quản lý VLAN 10 ảo thông qua NIC2.

## 4. Cấu hình pfSense (Tường lửa)
1. **Tài nguyên VM**:
   - 2 vCPU, 4GB RAM, 20GB SSD disk.
   - 2 NIC ảo:
     - NIC1: PG-WAN (192.168.1.2/24, gateway 192.168.1.1).
     - NIC2: PG-Internal (LAN: 172.168.1.1/24, VLAN 10: 10.0.0.1/24).
2. **Cấu hình mạng**:
   - **WAN**: IP tĩnh 192.168.1.2/24, gateway 192.168.1.1, DNS: 8.8.8.8, 1.1.1.1.
   - **LAN**: IP 172.168.1.1/24, DHCP range 172.168.1.100-200.
   - **VLAN 10**: Tạo VLAN ID 10 trên NIC2, IP 10.0.0.1/24, DHCP range 10.0.0.100-200.
3. **Cấu hình NAT**:
   - **Firewall > NAT > Outbound**:
     - Chọn **Hybrid Outbound NAT**.
     - Rule cho LAN:
       - Interface: WAN
       - Source: 172.168.1.0/24
       - Destination: Any
       - NAT Address: WAN Address
       - Description: "NAT for LAN to Internet"
     - Rule cho VLAN 10:
       - Interface: WAN
       - Source: 10.0.0.0/24
       - Destination: Any
       - NAT Address: WAN Address
       - Description: "NAT for VLAN 10 to Internet"
4. **Cấu hình Firewall**:
   - **Firewall > Rules > LAN**:
     - Action: Pass
     - Source: 172.168.1.0/24
     - Destination: Any
     - Protocol: Any
     - Description: "Allow LAN to Internet"
   - **Firewall > Rules > VLAN 10**:
     - Action: Pass
     - Source: 10.0.0.0/24
     - Destination: Any
     - Protocol: Any
     - Description: "Allow VLAN 10 to Internet"
   - Rule cho Windows Server: Cho phép 192.168.1.3, 172.168.1.2, 10.0.0.2 truy cập tất cả dịch vụ.
5. **Tối ưu hóa**:
   - Bật **Hardware Checksum Offloading**, **TCP Segmentation Offloading**, **Large Receive Offloading** (System > Advanced > Networking).
   - Cấu hình **QoS** (Traffic Shaper) ưu tiên port 80/443 (GitLab), 5678 (n8n).
6. **Bảo mật**:
   - Cài **pfBlockerNG** để chặn quảng cáo/IP độc hại.
   - Bật **Snort** hoặc **Suricata** cho IDS/IPS.
   - Cấu hình **OpenVPN** hoặc **WireGuard** cho quản trị từ xa.
7. **Backup**:
   - Tự động sao lưu cấu hình pfSense vào Windows Server.

## 5. Cấu hình Windows Server 2022 (Máy quản trị, Prometheus, Grafana)
1. **Tài nguyên VM**:
   - 4 vCPU, 12GB RAM, 150GB SSD disk.
   - 3 NIC ảo:
     - NIC1: PG-WAN (192.168.1.3/24, gateway 192.168.1.2).
     - NIC2: PG-Internal (10.0.0.2/24, gateway 10.0.0.1, VLAN 10).
     - NIC3: PG-Internal (172.168.1.2/24, gateway 172.168.1.1).
2. **Cấu hình mạng**:
   - NIC1: IP 192.168.1.3/24, DNS: 192.168.1.2, 8.8.8.8.
   - NIC2: IP 10.0.0.2/24, DNS: 10.0.0.1, 8.8.8.8.
   - NIC3: IP 172.168.1.2/24, DNS: 172.168.1.1, 8.8.8.8.
   - Kiểm tra Internet: `ping 8.8.8.8` trên cả 3 NIC.
3. **Cài đặt Prometheus**:
   - Tải Prometheus cho Windows từ [trang chính thức](https://prometheus.io/download/) (phiên bản mới nhất, ví dụ: prometheus-2.54.1.windows-amd64.zip).
   - Giải nén vào `C:\Prometheus`.
   - Tạo file cấu hình `C:\Prometheus\prometheus.yml`:
     ```yaml
     global:
       scrape_interval: 15s
     scrape_configs:
       - job_name: 'prometheus'
         static_configs:
           - targets: ['localhost:9090']
       - job_name: 'windows'
         static_configs:
           - targets: ['localhost:9182']
       - job_name: 'n8n'
         static_configs:
           - targets: ['10.0.0.10:9100']
       - job_name: 'gitlab'
         static_configs:
           - targets: ['10.0.0.11:9100']
       - job_name: 'pfsense'
         static_configs:
           - targets: ['192.168.1.2:9119']
     ```
   - Cài đặt **Windows Exporter** để giám sát Windows Server:
     - Tải từ [GitHub](https://github.com/prometheus-community/windows_exporter/releases) (ví dụ: windows_exporter-0.26.0-amd64.msi).
     - Cài đặt và chạy service trên port 9182.
   - Chạy Prometheus như Windows Service:
     - Tải **NSSM** (Non-Sucking Service Manager) từ [trang chính thức](https://nssm.cc/download).
     - Cài đặt Prometheus service:
       ```cmd
       nssm install Prometheus C:\Prometheus\prometheus.exe
       nssm set Prometheus AppParameters --config.file=C:\Prometheus\prometheus.yml
       nssm start Prometheus
       ```
   - Kiểm tra: Truy cập `http://192.168.1.3:9090` từ trình duyệt.
4. **Cài đặt Grafana**:
   - Tải Grafana Enterprise hoặc OSS cho Windows từ [trang chính thức](https://grafana.com/grafana/download) (ví dụ: grafana-11.2.0.windows-amd64.msi).
   - Cài đặt vào `C:\Program Files\Grafana`.
   - Chạy Grafana như Windows Service:
     - Service mặc định chạy sau khi cài đặt.
     - Kiểm tra: Truy cập `http://192.168.1.3:3000` (user: admin, pass: admin, đổi mật khẩu sau khi đăng nhập).
   - Thêm Prometheus làm Data Source:
     - Vào Grafana > **Configuration > Data Sources** > **Add data source** > Chọn **Prometheus**.
     - URL: `http://192.168.1.3:9090`.
     - Lưu và kiểm tra kết nối.
   - Thêm Dashboard:
     - Nhập dashboard từ [Grafana Dashboards](https://grafana.com/grafana/dashboards/):
       - Windows: Dashboard ID 10467 (Windows Exporter).
       - n8n/GitLab: Dashboard ID 1860 (Node Exporter).
       - pfSense: Dashboard ID 12315 (pfSense Exporter).
5. **Cấu hình Firewall trên Windows**:
   - Mở port 9090 (Prometheus) và 3000 (Grafana):
     ```cmd
     netsh advfirewall firewall add rule name="Prometheus" dir=in action=allow protocol=TCP localport=9090
     netsh advfirewall firewall add rule name="Grafana" dir=in action=allow protocol=TCP localport=3000
     ```
6. **Cấu hình pfSense**:
   - Thêm firewall rule trên VLAN 10:
     - Action: Pass
     - Source: 10.0.0.0/24
     - Destination: 192.168.1.3, port 9090, 3000
     - Description: "Allow VLAN 10 to Prometheus/Grafana"
   - Thêm firewall rule trên LAN:
     - Action: Pass
     - Source: 172.168.1.0/24
     - Destination: 192.168.1.3, port 9090, 3000
     - Description: "Allow LAN to Prometheus/Grafana"
7. **Vai trò và công cụ khác**:
   - Truy cập **ESXi Web Client** (https://192.168.1.10), pfSense (https://172.168.1.1), n8n (https://10.0.0.10:5678), GitLab (https://10.0.0.11).
   - Cài **kubectl**, **Lens** cho K8s.
8. **Bảo mật**:
   - Bật **Windows Defender**, cài **Sysmon**, sử dụng mật khẩu mạnh.
   - Đổi mật khẩu Grafana mặc định.
9. **Tối ưu hóa**:
   - Tắt dịch vụ không cần thiết (Windows Search, Print Spooler).

## 6. Cấu hình dịch vụ n8n (Docker, Auto-Start)
1. **Tài nguyên VM**:
   - 2 vCPU, 6GB RAM, 30GB SSD disk.
   - NIC: PG-Internal (IP 10.0.0.10/24, VLAN 10).
2. **Cài đặt**:
   - Sử dụng Ubuntu 22.04 LTS.
   - Cài đặt Docker:
     ```bash
     sudo apt update && sudo apt install -y docker.io
     sudo systemctl enable docker
     sudo systemctl start docker
     ```
   - Cài đặt **Node Exporter** để giám sát:
     ```bash
     docker run -d --name node-exporter \
       -p 9100:9100 \
       --restart=always \
       prom/node-exporter:latest
     ```
   - Chạy n8n container:
     ```bash
     docker run -d --name n8n \
       -p 5678:5678 \
       -v n8n_data:/home/node/.n8n \
       --restart=always \
       n8nio/n8n
     ```
3. **Tự động khởi động**:
   - Cờ `--restart=always` cho cả n8n và Node Exporter.
   - Docker service: `sudo systemctl enable docker`.
4. **Tối ưu hóa**:
   - Volume `-v n8n_data:/home/node/.n8n` lưu trữ dữ liệu.
   - Bật HTTPS qua pfSense reverse proxy (Let's Encrypt).
   - Giới hạn workflow đồng thời (Settings > Limits).
5. **Bảo mật**:
   - Bật xác thực người dùng (Settings > Security).
   - Firewall rule trên pfSense: Chỉ cho phép LAN (172.168.1.x) và Windows Server (192.168.1.3) truy cập port 5678, 9100.
6. **Kiểm tra Internet**:
   - Test workflow gọi `https://api.github.com`.

## 7. Cấu hình dịch vụ GitLab (Docker, Auto-Start)
1. **Tài nguyên VM**:
   - 4 vCPU, 12GB RAM, 200GB SSD disk.
   - NIC: PG-Internal (IP 10.0.0.11/24, VLAN 10).
2. **Cài đặt**:
   - Sử dụng Ubuntu 22.04 LTS.
   - Cài đặt Docker:
     ```bash
     sudo apt update && sudo apt install -y docker.io
     sudo systemctl enable docker
     sudo systemctl start docker
     ```
   - Cài đặt **Node Exporter**:
     ```bash
     docker run -d --name node-exporter \
       -p 9100:9100 \
       --restart=always \
       prom/node-exporter:latest
     ```
   - Chạy GitLab container:
     ```bash
     docker run -d --name gitlab \
       -p 80:80 -p 443:443 \
       -v gitlab_config:/etc/gitlab \
       -v gitlab_logs:/var/log/gitlab \
       -v gitlab_data:/var/opt/gitlab \
       --restart=always \
       gitlab/gitlab-ce:latest
     ```
3. **Tự động khởi động**:
   - Cờ `--restart=always` cho GitLab và Node Exporter.
   - Docker service: `sudo systemctl enable docker`.
4. **Cấu hình GitLab**:
   - Chỉnh sửa `/etc/gitlab/gitlab.rb`:
     ```bash
     docker exec -it gitlab vi /etc/gitlab/gitlab.rb
     ```
     ```ruby
     external_url 'https://10.0.0.11'
     nginx['listen_port'] = 443
     nginx['listen_https'] = true
     ```
   - Chạy: `docker exec -it gitlab gitlab-ctl reconfigure`.
5. **Tối ưu hóa**:
   - Cài GitLab Runner (VM riêng, 2 vCPU, 4GB RAM, IP 10.0.0.12):
     ```bash
     docker run -d --name gitlab-runner \
       -v gitlab_runner_config:/etc/gitlab-runner \
       --restart=always \
       gitlab/gitlab-runner:latest
     ```
   - Tăng unicorn worker:
     ```ruby
     unicorn['worker_processes'] = 4
     ```
6. **Bảo mật**:
   - Bật **2FA** cho người dùng.
   - Sử dụng **Let's Encrypt** qua pfSense reverse proxy.
   - Firewall rule trên pfSense: Chỉ cho phép LAN (172.168.1.x) và Windows Server truy cập port 80/443, 9100.
7. **Kiểm tra Internet**:
   - Test: `docker exec -it gitlab curl https://google.com`.

## 8. Chuẩn bị cho cụm Kubernetes
1. **Tài nguyên**:
   - **3 Master Nodes**: 2 vCPU, 4GB RAM, 20GB SSD disk.
   - **3 Worker Nodes**: 4 vCPU, 8GB RAM, 50GB SSD disk.
   - **Tổng**: ~18 vCPU, 36GB RAM, 220GB disk.
2. **Cấu hình mạng**:
   - Sử dụng VLAN 10 (10.0.0.x/24), IP tĩnh 10.0.0.20-25.
   - Tất cả node gắn vào PG-Internal, nhận VLAN 10 từ pfSense.
3. **Tối ưu hóa**:
   - Sử dụng **containerd** làm container runtime.
   - Cài **Calico** (CNI), **MetalLB** (LoadBalancer).
   - Di chuyển n8n và GitLab vào K8s bằng Helm chart.
4. **Kiểm tra Internet**:
   - Test: `curl https://google.com` từ mỗi node.

## 9. Quản lý và giám sát
- **Windows Server**:
  - Truy cập **ESXi Web Client** (https://192.168.1.10), pfSense (https://172.168.1.1), n8n (https://10.0.0.10:5678), GitLab (https://10.0.0.11).
  - Truy cập **Prometheus** (http://192.168.1.3:9090), **Grafana** (http://192.168.1.3:3000).
- **Backup**:
  - Sao lưu VM qua ESXi snapshot (hàng tuần).
  - Sao lưu Docker volumes (n8n_data, gitlab_config/data/logs) vào Windows Server.
- **Bảo mật**:
  - Cập nhật ESXi, pfSense, Ubuntu, Windows Server định kỳ.
  - Bật 2FA cho n8n, GitLab, Grafana.

## 10. Kiểm tra
- **Internet**:
  - Windows Server: `ping 8.8.8.8` trên cả 3 NIC.
  - n8n: Test workflow gọi `https://api.github.com`.
  - GitLab: `docker exec -it gitlab curl https://google.com`.
  - pfSense: Ping 8.8.8.8 từ WAN, LAN, VLAN 10 (Diagnostics > Ping).
- **Prometheus/Grafana**:
  - Kiểm tra metrics tại `http://192.168.1.3:9090`.
  - Xem dashboard tại `http://192.168.1.3:3000`.

## 11. Lưu ý
- **vSwitch0**: Sử dụng vSwitch mặc định để đơn giản hóa, gắn NIC0 và tạo 2 Port Group (PG-WAN, PG-Internal).
- **Prometheus/Grafana**: Chạy trên Windows Server, giám sát ESXi, pfSense, n8n, GitLab qua Node Exporter và Windows Exporter.
- **Docker Auto-Start**: Cờ `--restart=always` và `systemctl enable docker` đảm bảo n8n, GitLab, Node Exporter tự khởi động.
- **Tài liệu tham khảo**:
  - [Hướng dẫn ESXi Networking](https://docs.vmware.com/en/VMware-vSphere/7.0/com.vmware.vsphere.networking.doc/GUID-2B11DBB8-C030-4B42-A611-92D99B2D2780.html)
  - [Hướng dẫn Prometheus](https://prometheus.io/docs/introduction/overview/)
  - [Hướng dẫn Grafana](https://grafana.com/docs/grafana/latest/setup-grafana/installation/)
  - [Hướng dẫn Docker Auto-Start](https://docs.docker.com/config/containers/start-containers-automatically/)
  - [Hướng dẫn n8n Docker](https://docs.n8n.io/hosting/installation/docker/)
  - [Hướng dẫn GitLab Docker](https://docs.gitlab.com/ee/install/docker.html)
