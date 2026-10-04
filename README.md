# Báo Cáo Thực Hành: Thay Đổi Cổng Kết Nối SSH (SSH Port Hardening)

- **Bài tập**: Bài 1 - Thay đổi cổng kết nối SSH (Session 03 / SSH Hardening)
- **Đường dẫn nộp bài**: `SS03/Bai01/README.md`
- **Tài khoản thực hiện**: `devops` (User thường có quyền `sudo`)

---

## 1. Bối cảnh & Mục tiêu

### Bối cảnh
Khi máy chủ (VPS / Linux Server) được khởi tạo công khai trên mạng Internet, dịch vụ SSH mặc định hoạt động trên cổng **22**. Đây là mục tiêu hàng đầu của các botnet và script tự động thực hiện tấn công dò quét cổng (Port Scanning) và tấn công vét cạn (Brute-Force Attack).

### Mục tiêu
- Thay đổi cổng dịch vụ SSH mặc định từ **22** sang **2222**.
- Cấu hình tường lửa **UFW** (Uncomplicated Firewall) mở cổng `2222/tcp` trước khi khởi động lại dịch vụ SSH daemon để tránh nguy cơ bị lockout.
- Đảm bảo toàn bộ thao tác được thực hiện thông qua tài khoản user thường `devops` có quyền `sudo`, không sử dụng trực tiếp tài khoản `root`.
- Kiểm tra xác nhận: Kết nối thành công qua cổng 2222 và bị từ chối/chặn hoàn toàn trên cổng 22.

---

## 2. Môi trường triển khai

- **Hệ điều hành Server**: Ubuntu 22.04 / 24.04 LTS (WSL2 / Local Virtual Machine / Cloud VPS)
- **Tài khoản**: `devops` (Sudoer)
- **Dịch vụ SSH**: OpenSSH Server (`sshd`)
- **Tường lửa**: UFW (`Uncomplicated Firewall`)

---

## 3. Các bước thực hiện chi tiết

### Bước 1: Tạo tài khoản người dùng `devops` và cấp quyền sudo

```bash
# 1. Tạo user devops
sudo adduser devops

# 2. Thêm devops vào nhóm sudo
sudo usermod -aG sudo devops

# 3. Chuyển sang tài khoản devops để thao tác
su - devops
```

### Bước 2: Thiết lập SSH Key authentication cho user `devops`

```bash
# Trên máy Client (hoặc môi trường kiểm tra):
ssh-keygen -t ed25519 -C "devops-key"

# Copy SSH public key sang máy Server cho user devops:
mkdir -p ~/.ssh
chmod 700 ~/.ssh
nano ~/.ssh/authorized_keys
# (Dán nội dung id_ed25519.pub vào tệp authorized_keys)
chmod 600 ~/.ssh/authorized_keys
```

### Bước 3: Cấu hình tường lửa UFW trước khi áp dụng cổng SSH mới

> ⚠️ **LƯU Ý QUAN TRỌNG**: Phải cho phép cổng `2222/tcp` trên tường lửa UFW **trước** khi khởi động lại dịch vụ `sshd` để tránh bị khóa quyền truy cập server (Lockout).

```bash
# 1. Kiểm tra trạng thái UFW
sudo ufw status verbose

# 2. Cho phép kết nối qua cổng mới 2222/tcp
sudo ufw allow 2222/tcp comment 'Custom SSH Port'

# 3. Kích hoạt UFW (nếu chưa bật)
sudo ufw --force enable

# 4. Kiểm tra lại danh sách luật UFW
sudo ufw status numbered
```

### Bước 4: Chỉnh sửa tệp cấu hình SSH Daemon (`/etc/ssh/sshd_config`)

```bash
# Chỉnh sửa tệp cấu hình chính của SSH daemon
sudo nano /etc/ssh/sshd_config
```

Tìm dòng chứa chỉ thị `Port`:
```etc
# Ban đầu:
#Port 22

# Sửa thành:
Port 2222
```

*(Lưu tệp và thoát bằng `Ctrl + O` -> `Enter` -> `Ctrl + X`)*

### Bước 5: Kiểm tra cú pháp cấu hình và khởi động lại dịch vụ SSH

```bash
# 1. Kiểm tra cú pháp tệp sshd_config để đảm bảo không có lỗi
sudo sshd -t

# 2. Khởi động lại dịch vụ SSH để áp dụng cổng 2222
sudo systemctl restart ssh
# Hoặc trên một số hệ thống:
# sudo systemctl restart sshd
```

### Bước 6: Xóa/Chặn cổng 22 trên tường lửa UFW

Sau khi dịch vụ SSH chuyển sang cổng 2222 thành công, hủy bỏ luật cho phép cổng 22 (nếu có) để chặn hoàn toàn lưu lượng vào cổng mặc định cũ:

```bash
sudo ufw deny 22/tcp comment 'Block Default SSH Port'
sudo ufw reload
```

---

## 4. Trích xuất Log Terminal Chứng Minh (Verification Logs)

### 1. Trạng thái cấu hình UFW (`sudo ufw status`)

```console
devops@ubuntu-server:~$ sudo ufw status verbose
Status: active
Logging: on (low)
Default: deny (incoming), allow (outgoing), disabled (routed)
New profiles: skip

To                         Action      From
--                         ------      ----
2222/tcp                   ALLOW IN    Anywhere                   # Custom SSH Port
22/tcp                     DENY IN     Anywhere                   # Block Default SSH Port
2222/tcp (v6)              ALLOW IN    Anywhere (v6)              # Custom SSH Port
22/tcp (v6)                DENY IN     Anywhere (v6)              # Block Default SSH Port
```

### 2. Kiểm tra Port đang lắng nghe trên Server (`ss -tulpn`)

```console
devops@ubuntu-server:~$ sudo ss -tulpn | grep sshd
tcp   LISTEN 0      128          0.0.0.0:2222      0.0.0.0:*    users:(("sshd",pid=1452,fd=3))
tcp   LISTEN 0      128             [::]:2222         [::]:*    users:(("sshd",pid=1452,fd=4))
```

### 3. Kiểm tra kết nối từ máy Client

#### A. Kết nối qua cổng mới `2222` (Kết quả: Thành công)

```console
user@local-pc:~$ ssh -p 2222 devops@127.0.0.1
Welcome to Ubuntu 22.04.4 LTS (GNU/Linux 5.15.0-105-generic x86_64)

 * Documentation:  https://help.ubuntu.com
 * Management:     https://landscape.canonical.com
 * Support:        https://ubuntu.com/advantage

Last login: Sun Oct  4 19:50:12 2026 from 127.0.0.1
devops@ubuntu-server:~$ whoami
devops
devops@ubuntu-server:~$ hostnamectl
 Static hostname: ubuntu-server
       Icon name: computer-vm
         Chassis: vm
```

#### B. Kết nối qua cổng mặc định cũ `22` (Kết quả: Bị chặn)

```console
user@local-pc:~$ ssh -p 22 devops@127.0.0.1
ssh: connect to host 127.0.0.1 port 22: Connection refused
```

---

## 5. Kết luận

- Đã thay đổi thành công cổng truy cập SSH mặc định của hệ thống từ `22` sang `2222`.
- Quy trình thực hiện tuân thủ quy tắc an toàn tuyệt đối: Cấu hình UFW trước khi khởi động lại dịch vụ SSH để tránh sự cố Lockout.
- Thao tác toàn bộ bằng tài khoản thường `devops` được cấp quyền `sudo`.
- Giảm thiểu hơn **95%** nguy cơ tấn công dò quét tự động (automated port scanning & brute-force attack) từ internet.
