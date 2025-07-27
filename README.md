Homelab trên Dell R630 với ESXi 7
Tổng quan
Dự án triển khai một môi trường homelab trên server Dell R630 sử dụng VMware ESXi 7, với các thành phần chính:

pfSense: Tường lửa, quản lý các dải mạng WAN (192.168.1.x/24), LAN (172.168.1.x/24), và VLAN 10 (10.0.0.x/24).
Windows Server 2022: Máy quản trị, giám sát tài nguyên với Prometheus và Grafana.
n8n: Dịch vụ tự động hóa workflow, chạy bằng Docker.
GitLab: Nền tảng DevOps, chạy bằng Docker.
Prometheus & Grafana: Giám sát tài nguyên của ESXi, pfSense, n8n, và GitLab.
Chuẩn bị cho cụm Kubernetes (3 master, 3 worker) trong tương lai.

Tất cả dải mạng truy cập Internet, sử dụng một cổng NIC 1Gb (NIC0) và không có switch hỗ trợ VLAN (pfSense xử lý VLAN ảo).
Yêu cầu phần cứng

Server: Dell R630
CPU: Dual Intel Xeon E5-2600 series (12-16 cores).
RAM: Tối thiểu 64GB (khuyến nghị 128GB).
Lưu trữ: RAID 5/6 với SSD (tối thiểu 1TB).
NIC: 1 cổng 1Gb (NIC0).


Phần mềm:
VMware ESXi 7.0
pfSense (mới nhất)
Windows Server 2022
Ubuntu 22.04 LTS (cho n8n, GitLab)
Docker, Prometheus, Grafana



Cấu trúc mạng

Dải mạng:
WAN: 192.168.1.x/24
LAN: 172.168.1.x/24
VLAN 10: 10.0.0.x/24 (VLAN ảo, quản lý bởi pfSense)


Cấu hình ESXi:
vSwitch0: Gắn NIC0, quản lý tất cả lưu lượng mạng.
Port Groups:
PG-WAN: Không gắn VLAN ID, dùng cho WAN (192.168.1.x/24).
PG-Internal: Không gắn VLAN ID, dùng cho LAN (172.168.1.x/24) và VLAN 10 (10.0.0.x/24).


Bật Promiscuous Mode trên PG-Internal để pfSense xử lý VLAN tagging ảo.
Đặt MTU 9000 nếu router hỗ trợ jumbo frames.



Cài đặt và cấu hình
1. ESXi 7

Tải ISO ESXi 7 từ VMware hoặc Dell.
Cài đặt qua USB hoặc iDRAC.
Gán IP tĩnh:IP: 192.168.1.10/24
Gateway: 192.168.1.1
DNS: 8.8.8.8


Cấu hình mạng:
Truy cập ESXi Web Client: https://192.168.1.10.
Vào Networking > Virtual Switches > Kiểm tra vSwitch0, gắn NIC0 (vmnic0).
Tạo Port Groups (Networking > Port Groups):
PG-WAN: Name: PG-WAN, VLAN ID: 0, Virtual Switch: vSwitch0.
PG-Internal: Name: PG-Internal, VLAN ID: 0, Virtual Switch: vSwitch0.


Bật Promiscuous Mode:vSwitch0 > Edit Settings > Security > Promiscuous Mode: Accept


Đặt MTU:vSwitch0 > Edit Settings > Advanced > MTU: 9000





2. pfSense (Tường lửa)

VM: 2 vCPU, 4GB RAM, 20GB SSD disk.
NIC:
NIC1: PG-WAN (192.168.1.2/24, gateway 192.168.1.1).
NIC2: PG-Internal (LAN: 172.168.1.1/24, VLAN 10: 10.0.0.1/24).


Cấu hình mạng:
WAN: IP 192.168.1.2/24, gateway 192.168.1.1, DNS: 8.8.8.8, 1.1.1.1.
LAN: IP 172.168.1.1/24, DHCP 172.168.1.100-200.
VLAN 10: VLAN ID 10 trên NIC2, IP 10.0.0.1/24, DHCP 10.0.0.100-200.


NAT (Firewall > NAT > Outbound):
Chọn Hybrid Outbound NAT.
Rule cho LAN:Interface: WAN
Source: 172.168.1.0/24
Destination: Any
NAT Address: WAN Address
Description: NAT for LAN to Internet


Rule cho VLAN 10:Interface: WAN
Source: 10.0.0.0/24
Destination: Any
NAT Address: WAN Address
Description: NAT for VLAN 10 to Internet




Firewall Rules:
LAN:Action: Pass
Source: 172.168.1.0/24
Destination: Any
Protocol: Any
Description: Allow LAN to Internet


VLAN 10:Action: Pass
Source: 10.0.0.0/24
Destination: Any
Protocol: Any
Description: Allow VLAN 10 to Internet


Windows Server: Allow 192.168.1.3, 172.168.1.2, 10.0.0.2 to all services.


Tối ưu hóa:
Bật offloading:System > Advanced > Networking
Enable: Hardware Checksum Offloading, TCP Segmentation Offloading, Large Receive Offloading


Cấu hình QoS (Traffic Shaper) ưu tiên port 80/443 (GitLab), 5678 (n8n).


Bảo mật:
Cài pfBlockerNG, Snort/Suricata.
Cấu hình OpenVPN hoặc WireGuard.


Backup: Tự động sao lưu vào Windows Server.

3. Windows Server 2022 (Máy quản trị)

VM: 4 vCPU, 12GB RAM, 150GB SSD disk.
NIC:
NIC1: PG-WAN (192.168.1.3/24, DNS: 192.168.1.2, 8.8.8.8).
NIC2: PG-Internal (10.0.0.2/24, DNS: 10.0.0.1, 8.8.8.8).
NIC3: PG-Internal (172.168.1.2/24, DNS: 172.168.1.1, 8.8.8.8).


Công cụ:
Truy cập ESXi Web Client (https://192.168.1.10), pfSense (https://172.168.1.1), n8n (https://10.0.0.10:5678), GitLab (https://10.0.0.11).
Cài kubectl, Lens cho Kubernetes.


Bảo mật:
Bật Windows Defender, cài Sysmon, mật khẩu mạnh.


Kiểm tra Internet:ping 8.8.8.8



4. Prometheus & Grafana

Cài trên Windows Server:
Prometheus:
Tải từ Prometheus (ví dụ: prometheus-2.54.1.windows-amd64.zip).
Giải nén vào C:\Prometheus.
Tạo file C:\Prometheus\prometheus.yml:global:
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


Cài Windows Exporter (port 9182) từ GitHub.
Cài Prometheus như service với NSSM (tải từ NSSM):nssm install Prometheus C:\Prometheus\prometheus.exe
nssm set Prometheus AppParameters --config.file=C:\Prometheus\prometheus.yml
nssm start Prometheus


Truy cập: http://192.168.1.3:9090.


Grafana:
Tải từ Grafana (ví dụ: grafana-11.2.0.windows-amd64.msi).
Cài vào C:\Program Files\Grafana.
Truy cập: http://192.168.1.3:3000 (user: admin, pass: admin, đổi mật khẩu).
Thêm Data Source Prometheus:URL: http://192.168.1.3:9090


Nhập dashboard từ Grafana Dashboards:Windows: ID 10467
n8n/GitLab: ID 1860
pfSense: ID 12315




Firewall Windows:netsh advfirewall firewall add rule name="Prometheus" dir=in action=allow protocol=TCP localport=9090
netsh advfirewall firewall add rule name="Grafana" dir=in action=allow protocol=TCP localport=3000


Firewall pfSense:
Rule cho LAN:Action: Pass
Source: 172.168.1.0/24
Destination: 192.168.1.3, port 9090,3000
Description: Allow LAN to Prometheus/Grafana


Rule cho VLAN 10:Action: Pass
Source: 10.0.0.0/24
Destination: 192.168.1.3, port 9090,3000
Description: Allow VLAN 10 to Prometheus/Grafana







5. n8n (Docker, Auto-Start)

VM: 2 vCPU, 6GB RAM, 30GB SSD disk.
NIC: PG-Internal (10.0.0.10/24, VLAN 10).
Cài đặt:
Sử dụng Ubuntu 22.04 LTS.
Cài Docker:sudo apt update
sudo apt install -y docker.io
sudo systemctl enable docker
sudo systemctl start docker


Cài Node Exporter:docker run -d --name node-exporter \
  -p 9100:9100 \
  --restart=always \
  prom/node-exporter:latest


Chạy n8n:docker run -d --name n8n \
  -p 5678:5678 \
  -v n8n_data:/home/node/.n8n \
  --restart=always \
  n8nio/n8n




Auto-Start: --restart=always và systemctl enable docker.
Bảo mật:
Bật xác thực người dùng (Settings > Security).
Firewall rule trên pfSense:Action: Pass
Source: 172.168.1.0/24, 192.168.1.3
Destination: 10.0.0.10, port 5678,9100
Description: Allow LAN and Windows Server to n8n




Internet: Test workflow:https://api.github.com



6. GitLab (Docker, Auto-Start)

VM: 4 vCPU, 12GB RAM, 200GB SSD disk.
NIC: PG-Internal (10.0.0.11/24, VLAN 10).
Cài đặt:
Sử dụng Ubuntu 22.04 LTS.
Cài Docker:sudo apt update
sudo apt install -y docker.io
sudo systemctl enable docker
sudo systemctl start docker


Cài Node Exporter:docker run -d --name node-exporter \
  -p 9100:9100 \
  --restart=always \
  prom/node-exporter:latest


Chạy GitLab:docker run -d --name gitlab \
  -p 80:80 -p 443:443 \
  -v gitlab_config:/etc/gitlab \
  -v gitlab_logs:/var/log/gitlab \
  -v gitlab_data:/var/opt/gitlab \
  --restart=always \
  gitlab/gitlab-ce:latest


Cấu hình /etc/gitlab/gitlab.rb:docker exec -it gitlab vi /etc/gitlab/gitlab.rb

external_url 'https://10.0.0.11'
nginx['listen_port'] = 443
nginx['listen_https'] = true
unicorn['worker_processes'] = 4

docker exec -it gitlab gitlab-ctl reconfigure




GitLab Runner (VM riêng, 2 vCPU, 4GB RAM, 10.0.0.12):docker run -d --name gitlab-runner \
  -v gitlab_runner_config:/etc/gitlab-runner \
  --restart=always \
  gitlab/gitlab-runner:latest


Auto-Start: --restart=always và systemctl enable docker.
Bảo mật:
Bật 2FA, HTTPS (Let's Encrypt qua pfSense).
Firewall rule trên pfSense:Action: Pass
Source: 172.168.1.0/24, 192.168.1.3
Destination: 10.0.0.11, port 80,443,9100
Description: Allow LAN and Windows Server to GitLab




Internet: Test:docker exec -it gitlab curl https://google.com



7. Chuẩn bị cho Kubernetes

Tài nguyên:
3 Master Nodes: 2 vCPU, 4GB RAM, 20GB SSD disk.
3 Worker Nodes: 4 vCPU, 8GB RAM, 50GB SSD disk.
Tổng: ~18 vCPU, 36GB RAM, 220GB disk.


Mạng: VLAN 10 (10.0.0.20-25), gắn vào PG-Internal.
Tối ưu hóa: Sử dụng containerd, Calico (CNI), MetalLB (LoadBalancer).
Dịch vụ: Di chuyển n8n, GitLab vào K8s bằng Helm chart.

Giám sát và quản lý

Windows Server:
Truy cập:ESXi Web Client: https://192.168.1.10
pfSense: https://172.168.1.1
n8n: https://10.0.0.10:5678
GitLab: https://10.0.0.11
Prometheus: http://192.168.1.3:9090
Grafana: http://192.168.1.3:3000




Backup:
ESXi snapshot (hàng tuần).
Sao lưu Docker volumes vào Windows Server.


Bảo mật:
Cập nhật ESXi, pfSense, Ubuntu, Windows Server.
Bật 2FA cho n8n, GitLab, Grafana.



Kiểm tra

Internet:ping 8.8.8.8

n8n workflow: https://api.github.com

docker exec -it gitlab curl https://google.com

pfSense: Diagnostics > Ping > 8.8.8.8


Prometheus/Grafana:Prometheus: http://192.168.1.3:9090
Grafana: http://192.168.1.3:3000



Lưu ý

vSwitch0: Gắn NIC0, tạo PG-WAN và PG-Internal.
Prometheus/Grafana: Giám sát Windows (9182), n8n/GitLab (9100), pfSense (9119 nếu có).
Docker Auto-Start: --restart=always và systemctl enable docker.

Tài liệu tham khảo

VMware ESXi Networking
pfSense NAT
Prometheus
Grafana
n8n Docker
GitLab Docker
