# Thiết lập SELinux cho ứng dụng Node.js

## Kịch bản thực tế

Chúng ta sẽ thiết lập SELinux cho một ứng dụng Node.js:

- **Tên ứng dụng**: myapp
- **Đường dẫn cài đặt**: `/opt/myapp/`
- **Port**: 3000
- **User**: myapp
- **Systemd service**: myapp.service

## Bước 1: Chuẩn bị môi trường

### 1.1 Kiểm tra trạng thái SELinux

```
# Kiểm tra trạng thái SELinux
sestatus
# Output mong muốn:
# SELinux status:                 enabled
# SELinuxfs mount:                /sys/fs/selinux
# SELinux root directory:         /etc/selinux
# Loaded policy name:             targeted
# Current mode:                   enforcing

# Kiểm tra mode hiện tại
getenforce
# Output: Enforcing
```

### 1.2 Cài đặt công cụ cần thiết

```
# Cài đặt các công cụ SELinux
sudo dnf install -y policycoreutils-python-utils setools-console

# Hoặc trên Ubuntu/Debian
sudo apt install -y policycoreutils python3-policycoreutils setools
```

### 1.3 Tạo cấu trúc thư mục ứng dụng

```
# Tạo user và group cho ứng dụng
sudo useradd -r -s /bin/false myapp

# Tạo cấu trúc thư mục
sudo mkdir -p /opt/myapp/{bin,config,logs,data,tmp}
sudo mkdir -p /var/log/myapp
sudo mkdir -p /var/lib/myapp

# Phân quyền cơ bản
sudo chown -R myapp:myapp /opt/myapp
sudo chown -R myapp:myapp /var/log/myapp
sudo chown -R myapp:myapp /var/lib/myapp
```

## Bước 2: Thiết lập File Contexts

### 2.1 Xem context hiện tại

```
# Kiểm tra context hiện tại
ls -lZ /opt/myapp/
# Output: drwxr-xr-x. myapp myapp unconfined_u:object_r:usr_t:s0 bin

# Kiểm tra context của các thư mục hệ thống tương tự
ls -lZ /usr/bin/node
ls -lZ /var/log/
ls -lZ /var/lib/
```

### 2.2 Thiết lập context cho executable

```
# Thiết lập context cho file thực thi chính
sudo semanage fcontext -a -t bin_t "/opt/myapp/bin/server.js"
sudo semanage fcontext -a -t bin_t "/opt/myapp/bin(/.*)?"

# Thiết lập context cho Node.js executable (nếu cài đặt riêng)
sudo semanage fcontext -a -t bin_t "/opt/myapp/node_modules/.bin(/.*)?"
```

### 2.3 Thiết lập context cho các thư mục dữ liệu

```
# Thư mục config
sudo semanage fcontext -a -t etc_t "/opt/myapp/config(/.*)?"

# Thư mục logs
sudo semanage fcontext -a -t var_log_t "/opt/myapp/logs(/.*)?"
sudo semanage fcontext -a -t var_log_t "/var/log/myapp(/.*)?"

# Thư mục dữ liệu
sudo semanage fcontext -a -t var_lib_t "/opt/myapp/data(/.*)?"
sudo semanage fcontext -a -t var_lib_t "/var/lib/myapp(/.*)?"

# Thư mục temporary
sudo semanage fcontext -a -t tmp_t "/opt/myapp/tmp(/.*)?"

# Thư mục chính của ứng dụng
sudo semanage fcontext -a -t usr_t "/opt/myapp(/.*)?"
```

### 2.4 Áp dụng các context

```
# Áp dụng tất cả context đã thiết lập
sudo restorecon -Rv /opt/myapp/
sudo restorecon -Rv /var/log/myapp/
sudo restorecon -Rv /var/lib/myapp/

# Kiểm tra kết quả
ls -lZ /opt/myapp/
ls -lZ /opt/myapp/bin/
ls -lZ /var/log/myapp/
```

## Bước 3: Thiết lập Network Context

### 3.1 Kiểm tra port contexts hiện tại

```
# Xem port contexts hiện có
sudo semanage port -l | grep http
sudo semanage port -l | grep 3000

# Xem port nào đang được sử dụng
sudo netstat -tlnp | grep :3000
```

### 3.2 Thêm port cho ứng dụng

```
# Thêm port 3000 cho HTTP traffic
sudo semanage port -a -t http_port_t -p tcp 3000

# Xác nhận port đã được thêm
sudo semanage port -l | grep 3000
# Output: http_port_t                    tcp      3000, 80, 81, 443, 488, 8008, 8009, 8443, 9000
```

## Bước 4: Tạo Systemd Service

### 4.1 Tạo service file

```
sudo tee /etc/systemd/system/myapp.service << 'EOF'
[Unit]
Description=My Node.js Application
After=network.target

[Service]
Type=simple
User=myapp
Group=myapp
WorkingDirectory=/opt/myapp
ExecStart=/usr/bin/node /opt/myapp/bin/server.js
Restart=always
RestartSec=10
StandardOutput=journal
StandardError=journal
SyslogIdentifier=myapp

# Security settings
NoNewPrivileges=true
PrivateTmp=true
ProtectSystem=strict
ProtectHome=true
ReadWritePaths=/opt/myapp/logs /opt/myapp/data /opt/myapp/tmp /var/log/myapp /var/lib/myapp

[Install]
WantedBy=multi-user.target
EOF
```

### 4.2 Thiết lập SELinux context cho systemd

```
# Thiết lập context cho service file
sudo restorecon -v /etc/systemd/system/myapp.service

# Reload systemd
sudo systemctl daemon-reload
```

## Bước 5: Xử lý AVC Denials

### 5.1 Tạo file ứng dụng mẫu để test

```
# Tạo file server.js đơn giản
sudo tee /opt/myapp/bin/server.js << 'EOF'
const http = require('http');
const fs = require('fs');
const path = require('path');

const server = http.createServer((req, res) => {
  // Ghi log
  const logMessage = `${new Date().toISOString()} - ${req.method} ${req.url}\n`;
  fs.appendFileSync('/opt/myapp/logs/access.log', logMessage);
  
  res.writeHead(200, {'Content-Type': 'text/plain'});
  res.end('Hello from My App!\n');
});

server.listen(3000, '0.0.0.0', () => {
  console.log('Server running on port 3000');
});
EOF

sudo chown myapp:myapp /opt/myapp/bin/server.js
sudo chmod +x /opt/myapp/bin/server.js
```

### 5.2 Chạy ứng dụng và theo dõi AVC denials

```
# Terminal 1: Theo dõi audit logs
sudo tail -f /var/log/audit/audit.log | grep AVC

# Terminal 2: Khởi động service
sudo systemctl start myapp
sudo systemctl status myapp

# Test ứng dụng
curl http://localhost:3000
```

### 5.3 Xử lý AVC denials thường gặp

```
# Xem các AVC denials gần đây
sudo ausearch -m AVC -ts recent

# Tạo policy từ AVC denials
sudo ausearch -c 'node' --raw | audit2allow -M myapp_policy

# Xem nội dung policy được tạo
cat myapp_policy.te

# Cài đặt policy
sudo semodule -i myapp_policy.pp

# Kiểm tra module đã được cài đặt
sudo semodule -l | grep myapp
```

## Bước 6: Thiết lập Booleans (nếu cần)

### 6.1 Kiểm tra các boolean có sẵn

```
# Xem các boolean liên quan đến HTTP
sudo getsebool -a | grep http

# Một số boolean thường cần thiết cho web applications
sudo getsebool httpd_can_network_connect
sudo getsebool httpd_can_network_connect_db
sudo getsebool httpd_can_sendmail
```

### 6.2 Bật các boolean cần thiết

```
# Cho phép kết nối network (nếu ứng dụng cần kết nối external APIs)
sudo setsebool -P httpd_can_network_connect 1

# Cho phép kết nối database
sudo setsebool -P httpd_can_network_connect_db 1

# Xác nhận thay đổi
sudo getsebool httpd_can_network_connect
```

## Bước 7: Tạo Custom SELinux Policy

### 7.1 Tạo policy file tùy chỉnh

```
# Tạo file policy
sudo tee /tmp/myapp_custom.te << 'EOF'
module myapp_custom 1.0;

require {
    type unconfined_t;
    type http_port_t;
    type var_log_t;
    type usr_t;
    class tcp_socket name_bind;
    class file { create write append };
}

# Cho phép bind vào HTTP ports
allow unconfined_t http_port_t:tcp_socket name_bind;

# Cho phép ghi vào log files
allow unconfined_t var_log_t:file { create write append };

# Cho phép đọc files trong /opt/myapp
allow unconfined_t usr_t:file read;
EOF

# Compile và cài đặt policy
sudo checkmodule -M -m -o /tmp/myapp_custom.mod /tmp/myapp_custom.te
sudo semodule_package -o /tmp/myapp_custom.pp -m /tmp/myapp_custom.mod
sudo semodule -i /tmp/myapp_custom.pp
```

## Bước 8: Kiểm tra và Xác minh

### 8.1 Kiểm tra toàn bộ thiết lập

```
# Kiểm tra service
sudo systemctl status myapp
sudo systemctl enable myapp

# Kiểm tra port binding
sudo netstat -tlnp | grep :3000

# Kiểm tra process context
ps -eZ | grep myapp

# Test ứng dụng
curl http://localhost:3000
```

### 8.2 Kiểm tra logs

```
# Kiểm tra application logs
sudo tail -f /opt/myapp/logs/access.log

# Kiểm tra system logs
sudo journalctl -u myapp -f

# Kiểm tra SELinux logs
sudo ausearch -m AVC -ts today
```

## Bước 9: Backup và Documentation

### 9.1 Backup cấu hình SELinux

```
# Export toàn bộ cấu hình SELinux
sudo semanage export > /backup/selinux_myapp_backup_$(date +%Y%m%d).txt

# Backup các file contexts
sudo semanage fcontext -l | grep myapp > /backup/myapp_fcontext.txt

# Backup port contexts
sudo semanage port -l | grep 3000 > /backup/myapp_ports.txt
```

### 9.2 Tạo script tự động hóa

```
# Tạo script để restore cấu hình
sudo tee /opt/myapp/scripts/restore_selinux.sh << 'EOF'
#!/bin/bash

echo "Restoring SELinux contexts for MyApp..."

# File contexts
semanage fcontext -a -t bin_t "/opt/myapp/bin(/.*)?"
semanage fcontext -a -t etc_t "/opt/myapp/config(/.*)?"
semanage fcontext -a -t var_log_t "/opt/myapp/logs(/.*)?"
semanage fcontext -a -t var_log_t "/var/log/myapp(/.*)?"
semanage fcontext -a -t var_lib_t "/opt/myapp/data(/.*)?"
semanage fcontext -a -t var_lib_t "/var/lib/myapp(/.*)?"
semanage fcontext -a -t tmp_t "/opt/myapp/tmp(/.*)?"
semanage fcontext -a -t usr_t "/opt/myapp(/.*)?"

# Port contexts
semanage port -a -t http_port_t -p tcp 3000

# Apply contexts
restorecon -Rv /opt/myapp/
restorecon -Rv /var/log/myapp/
restorecon -Rv /var/lib/myapp/

echo "SELinux contexts restored successfully!"
EOF

sudo chmod +x /opt/myapp/scripts/restore_selinux.sh
```

## Bước 10: Troubleshooting

### 10.1 Các lệnh debug hữu ích

```
# Xem tất cả AVC denials trong ngày
sudo ausearch -m AVC -ts today | audit2allow

# Xem chi tiết một AVC denial cụ thể
sudo ausearch -m AVC -c 'node'

# Kiểm tra context của process đang chạy
sudo ps -eZ | grep myapp

# Kiểm tra boolean states
sudo sesearch --allow -s unconfined_t -t http_port_t
```

### 10.2 Các vấn đề thường gặp và cách khắc phục

**Lỗi 1: Permission denied khi bind port**

```
# Nguyên nhân: Port chưa được add vào SELinux context
# Khắc phục:
sudo semanage port -a -t http_port_t -p tcp 3000
```

**Lỗi 2: Không thể ghi file log**

```
# Nguyên nhân: File context không đúng
# Khắc phục:
sudo semanage fcontext -a -t var_log_t "/opt/myapp/logs(/.*)?"
sudo restorecon -Rv /opt/myapp/logs/
```

**Lỗi 3: Service không khởi động được**

```
# Nguyên nhân: Binary context không đúng
# Khắc phục:
sudo semanage fcontext -a -t bin_t "/opt/myapp/bin/server.js"
sudo restorecon -v /opt/myapp/bin/server.js
```

## Tổng kết

Sau khi hoàn thành tất cả các bước trên, ứng dụng `Node.js` của bạn sẽ:

- Chạy an toàn với SELinux enforcing mode

- Có đầy đủ quyền truy cập cần thiết

- Tuân thủ nguyên tắc least privilege

- Có khả năng logging và monitoring đầy đủ

Nhớ luôn test trên môi trường development trước khi áp dụng lên production!


Tóm tắt các bước chính:

Chuẩn bị: Kiểm tra SELinux status, cài tools, tạo `user/directories`

File Contexts: Thiết lập context cho executable, config, logs, data

Network: Thêm port `3000` vào `http_port_t` `context`

Systemd: Tạo service file với security hardening

AVC Handling: Theo dõi và xử lý các denial

Booleans: Bật các quyền network cần thiết

Custom Policy: Tạo policy riêng nếu cần

Testing: Kiểm tra toàn diện

Backup: Lưu trữ cấu hình

Troubleshooting: Xử lý các lỗi thường gặp

Điểm quan trọng cần nhớ:

File contexts: `/opt/myapp/bin` → `bin_t`, `/opt/myapp/logs` → `var_log_t`

Port context: Port `3000` → `http_port_t`

Process `context`: Chạy với user `myapp`

Security: Áp dụng principle of least privilege