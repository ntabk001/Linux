# Thiết lập SELinux cho Nginx và Redis

## Tổng quan Architecture

```
[Client] → [Nginx:80,443] → [Backend App:3000] → [Redis:6379]
                ↓
         [Static Files: /var/www/html]
         [Logs: /var/log/nginx]
         [Cache: /var/cache/nginx]
```

## PHẦN 1: THIẾT LẬP SELINUX CHO NGINX

### 1.1 Cài đặt và chuẩn bị Nginx

```
# Cài đặt Nginx
sudo dnf install -y nginx

# Kiểm tra context mặc định
ls -lZ /usr/sbin/nginx
ls -lZ /etc/nginx/
ls -lZ /var/log/nginx/
ls -lZ /var/www/html/

# Output mong đợi:
# /usr/sbin/nginx → httpd_exec_t
# /etc/nginx/ → httpd_config_t
# /var/log/nginx/ → httpd_log_t
# /var/www/html/ → httpd_exec_t
```

### 1.2 Thiết lập custom document root

```
# Tạo thư mục website tùy chỉnh
sudo mkdir -p /opt/websites/mysite/{public,logs,cache}
sudo mkdir -p /opt/websites/mysite/public/{css,js,images}

# Tạo file HTML mẫu
sudo tee /opt/websites/mysite/public/index.html << 'EOF'
<!DOCTYPE html>
<html>
<head>
    <title>My Website</title>
</head>
<body>
    <h1>Welcome to My Website</h1>
    <p>This is served by Nginx with SELinux!</p>
</body>
</html>
EOF

# Phân quyền cơ bản
sudo chown -R nginx:nginx /opt/websites/mysite
```

### 1.3 Thiết lập SELinux contexts cho Nginx

```
# Document root context
sudo semanage fcontext -a -t httpd_exec_t "/opt/websites/mysite/public(/.*)?"

# Log files context
sudo semanage fcontext -a -t httpd_log_t "/opt/websites/mysite/logs(/.*)?"

# Cache context
sudo semanage fcontext -a -t httpd_cache_t "/opt/websites/mysite/cache(/.*)?"

# Config context (nếu có config riêng)
sudo semanage fcontext -a -t httpd_config_t "/opt/websites/mysite/config(/.*)?"

# Áp dụng contexts
sudo restorecon -Rv /opt/websites/mysite/

# Kiểm tra kết quả
ls -lZ /opt/websites/mysite/
```

### 1.4 Cấu hình Nginx

```
# Backup config gốc
sudo cp /etc/nginx/nginx.conf /etc/nginx/nginx.conf.backup

# Tạo cấu hình site
sudo tee /etc/nginx/conf.d/mysite.conf << 'EOF'
server {
    listen 80;
    listen 443 ssl http2;
    server_name mysite.local;
    
    # Document root
    root /opt/websites/mysite/public;
    index index.html index.htm;
    
    # Custom log locations
    access_log /opt/websites/mysite/logs/access.log;
    error_log /opt/websites/mysite/logs/error.log;
    
    # SSL configuration (for port 443)
    ssl_certificate /opt/websites/mysite/ssl/cert.pem;
    ssl_certificate_key /opt/websites/mysite/ssl/key.pem;
    
    # Proxy to backend app
    location /api/ {
        proxy_pass http://127.0.0.1:3000/;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
    
    # Static file caching
    location ~* \.(jpg|jpeg|png|gif|ico|css|js)$ {
        expires 1y;
        add_header Cache-Control "public, immutable";
    }
    
    # Security headers
    add_header X-Frame-Options "SAMEORIGIN" always;
    add_header X-XSS-Protection "1; mode=block" always;
    add_header X-Content-Type-Options "nosniff" always;
}
EOF
```

### 1.5 Thiết lập SELinux ports cho Nginx

```
# Kiểm tra ports hiện tại
sudo semanage port -l | grep http

# Thêm custom port nếu cần (ví dụ 8080)
sudo semanage port -a -t http_port_t -p tcp 8080

# Kiểm tra kết quả
sudo semanage port -l | grep http_port_t
```

### 1.6 Thiết lập SELinux booleans cho Nginx

```
# Xem tất cả boolean liên quan đến HTTP
sudo getsebool -a | grep httpd

# Các boolean quan trọng cho Nginx:

# Cho phép kết nối network (proxy to backend)
sudo setsebool -P httpd_can_network_connect 1

# Cho phép kết nối database
sudo setsebool -P httpd_can_network_connect_db 1

# Cho phép relay mail
sudo setsebool -P httpd_can_sendmail 1

# Cho phép kết nối memcache/redis
sudo setsebool -P httpd_can_network_memcache 1

# Cho phép thực thi CGI
sudo setsebool -P httpd_enable_cgi 1

# Cho phép serve user content
sudo setsebool -P httpd_enable_homedirs 1

# Xác nhận các thay đổi
sudo getsebool httpd_can_network_connect
sudo getsebool httpd_can_network_memcache
```

### 1.7 Xử lý SSL certificates

```
# Tạo thư mục SSL
sudo mkdir -p /opt/websites/mysite/ssl

# Tạo self-signed certificate cho test
sudo openssl req -x509 -nodes -days 365 -newkey rsa:2048 \
  -keyout /opt/websites/mysite/ssl/key.pem \
  -out /opt/websites/mysite/ssl/cert.pem \
  -subj "/C=VN/ST=Hanoi/L=Hanoi/O=MyCompany/CN=mysite.local"

# Thiết lập SELinux context cho SSL files
sudo semanage fcontext -a -t httpd_config_t "/opt/websites/mysite/ssl(/.*)?"
sudo restorecon -Rv /opt/websites/mysite/ssl/

# Phân quyền file
sudo chown -R nginx:nginx /opt/websites/mysite/ssl
sudo chmod 600 /opt/websites/mysite/ssl/key.pem
sudo chmod 644 /opt/websites/mysite/ssl/cert.pem
```

## PHẦN 2: THIẾT LẬP SELINUX CHO REDIS

### 2.1 Cài đặt và chuẩn bị Redis

```
# Cài đặt Redis
sudo dnf install -y redis

# Kiểm tra context mặc định
ls -lZ /usr/bin/redis-server
ls -lZ /etc/redis/
ls -lZ /var/lib/redis/
ls -lZ /var/log/redis/

# Output mong đợi:
# /usr/bin/redis-server → redis_exec_t
# /etc/redis/ → redis_conf_t
# /var/lib/redis/ → redis_var_lib_t
# /var/log/redis/ → redis_log_t
```

### 2.2 Cấu hình Redis tùy chỉnh

```
# Backup config gốc
sudo cp /etc/redis/redis.conf /etc/redis/redis.conf.backup

# Tạo cấu hình tùy chỉnh
sudo tee /etc/redis/redis-custom.conf << 'EOF'
# Basic configuration
port 6379
bind 127.0.0.1
timeout 0
tcp-keepalive 300

# Custom paths
dir /opt/redis/data
logfile /opt/redis/logs/redis.log
pidfile /opt/redis/redis.pid

# Memory configuration
maxmemory 256mb
maxmemory-policy allkeys-lru

# Persistence
save 900 1
save 300 10
save 60 10000

dbfilename dump.rdb
appendonly yes
appendfilename "appendonly.aof"

# Security
requirepass myredispassword
rename-command FLUSHDB ""
rename-command FLUSHALL ""
rename-command DEBUG ""

# Network
protected-mode yes
tcp-backlog 511
EOF

# Tạo thư mục tùy chỉnh
sudo mkdir -p /opt/redis/{data,logs}
sudo chown -R redis:redis /opt/redis
```

### 2.3 Thiết lập SELinux contexts cho Redis

```
# Thiết lập contexts cho thư mục tùy chỉnh
sudo semanage fcontext -a -t redis_var_lib_t "/opt/redis/data(/.*)?"
sudo semanage fcontext -a -t redis_log_t "/opt/redis/logs(/.*)?"
sudo semanage fcontext -a -t redis_conf_t "/etc/redis/redis-custom.conf"

# Áp dụng contexts
sudo restorecon -Rv /opt/redis/
sudo restorecon -v /etc/redis/redis-custom.conf

# Kiểm tra kết quả
ls -lZ /opt/redis/
ls -lZ /etc/redis/redis-custom.conf
```

### 2.4 Thiết lập custom port cho Redis

```
# Nếu sử dụng port khác 6379 (ví dụ 6380)
# sudo semanage port -a -t redis_port_t -p tcp 6380

# Kiểm tra ports hiện tại
sudo semanage port -l | grep redis
# Output: redis_port_t                   tcp      6379
```

### 2.5 Tạo systemd service cho Redis custom

```
# Tạo service file tùy chỉnh
sudo tee /etc/systemd/system/redis-custom.service << 'EOF'
[Unit]
Description=Advanced key-value store (Custom Instance)
After=network.target
Documentation=http://redis.io/documentation, man:redis-server(1)

[Service]
Type=notify
ExecStart=/usr/bin/redis-server /etc/redis/redis-custom.conf
ExecStop=/usr/bin/redis-cli shutdown
TimeoutStopSec=0
Restart=always
User=redis
Group=redis
RuntimeDirectory=redis-custom
RuntimeDirectoryMode=0755

# Security measures
NoNewPrivileges=true
PrivateTmp=true
ProtectSystem=strict
ProtectHome=true
ReadWritePaths=/opt/redis

[Install]
WantedBy=multi-user.target
EOF

# Reload systemd và khởi động
sudo systemctl daemon-reload
sudo systemctl enable redis-custom
```

## PHẦN 3: TÍCH HỢP NGINX VÀ REDIS

### 3.1 Cấu hình Nginx với Redis caching

```
# Cài đặt nginx module cho Redis (nếu cần)
# sudo dnf install -y nginx-mod-http-redis

# Cập nhật cấu hình Nginx để sử dụng Redis
sudo tee /etc/nginx/conf.d/nginx-redis.conf << 'EOF'
# Upstream Redis
upstream redis_backend {
    server 127.0.0.1:6379;
    keepalive 10;
}

# Caching với Redis
server {
    listen 80;
    server_name cache.mysite.local;
    
    location / {
        # Try Redis cache first
        set $redis_key "$uri$is_args$args";
        redis_pass redis_backend;
        default_type text/html;
        error_page 404 502 504 = @fallback;
    }
    
    location @fallback {
        proxy_pass http://127.0.0.1:3000;
        proxy_set_header Host $host;
        
        # Store in Redis cache
        set $redis_key "$uri$is_args$args";
        redis_pass redis_backend;
    }
}

# Rate limiting với Redis
limit_req_zone $binary_remote_addr zone=api:10m rate=10r/s;

server {
    listen 80;
    server_name api.mysite.local;
    
    location /api/ {
        limit_req zone=api burst=20 nodelay;
        proxy_pass http://127.0.0.1:3000/;
    }
}
EOF
```

### 3.2 Thiết lập SELinux cho kết nối Nginx-Redis

```
# Đảm bảo Nginx có thể kết nối tới Redis
sudo setsebool -P httpd_can_network_connect 1
sudo setsebool -P httpd_can_network_memcache 1

# Kiểm tra kết nối
sudo getsebool httpd_can_network_memcache
```

## PHẦN 4: TESTING VÀ TROUBLESHOOTING

### 4.1 Test Nginx

```
# Test cấu hình Nginx
sudo nginx -t

# Khởi động services
sudo systemctl start nginx
sudo systemctl enable nginx
sudo systemctl status nginx

# Test HTTP
curl -H "Host: mysite.local" http://localhost/
curl -H "Host: mysite.local" https://localhost/ -k

# Kiểm tra logs
sudo tail -f /opt/websites/mysite/logs/access.log
sudo tail -f /opt/websites/mysite/logs/error.log
```

### 4.2 Test Redis

```
# Khởi động Redis
sudo systemctl start redis-custom
sudo systemctl status redis-custom

# Test kết nối Redis
redis-cli -a myredispassword ping
# Output: PONG

# Test lưu/đọc dữ liệu
redis-cli -a myredispassword set test "Hello Redis"
redis-cli -a myredispassword get test

# Kiểm tra logs
sudo tail -f /opt/redis/logs/redis.log
```

### 4.3 Theo dõi AVC denials

```
# Terminal 1: Theo dõi audit logs
sudo tail -f /var/log/audit/audit.log | grep AVC

# Terminal 2: Test các services
curl http://localhost/
redis-cli -a myredispassword info

# Xem AVC denials gần đây
sudo ausearch -m AVC -ts recent

# Tạo policy từ denials
sudo ausearch -c 'nginx' --raw | audit2allow -M nginx_custom_policy
sudo ausearch -c 'redis-server' --raw | audit2allow -M redis_custom_policy
```

### 4.4 Xử lý các lỗi thường gặp

**Lỗi 1: Nginx không thể đọc file static**
```
# Nguyên nhân: File context không đúng
sudo semanage fcontext -a -t httpd_exec_t "/opt/websites/mysite/public(/.*)?"
sudo restorecon -Rv /opt/websites/mysite/public/
```

**Lỗi 2: Nginx không thể proxy tới backend**
```
# Nguyên nhân: Boolean network connect chưa bật
sudo setsebool -P httpd_can_network_connect 1
```

**Lỗi 3: Redis không thể ghi file**
```
# Nguyên nhân: Context thư mục data không đúng
sudo semanage fcontext -a -t redis_var_lib_t "/opt/redis/data(/.*)?"
sudo restorecon -Rv /opt/redis/data/
```

**Lỗi 4: Nginx không thể kết nối Redis**
```
# Nguyên nhân: Boolean memcache chưa bật
sudo setsebool -P httpd_can_network_memcache 1
```

## PHẦN 5: MONITORING VÀ BACKUP

### 5.1 Script monitoring

```
# Tạo script kiểm tra SELinux status
sudo tee /opt/scripts/check_selinux_status.sh << 'EOF'
#!/bin/bash

echo "=== SELinux Status Check ==="
echo "Current mode: $(getenforce)"
echo

echo "=== Nginx Status ==="
systemctl is-active nginx
ps -eZ | grep nginx | head -3

echo "=== Redis Status ==="
systemctl is-active redis-custom
ps -eZ | grep redis

echo "=== Recent AVC Denials ==="
ausearch -m AVC -ts today | tail -5

echo "=== Port Contexts ==="
semanage port -l | grep -E "(http_port_t|redis_port_t)"

echo "=== Boolean Status ==="
getsebool httpd_can_network_connect
getsebool httpd_can_network_memcache
EOF

sudo chmod +x /opt/scripts/check_selinux_status.sh
```

### 5.2 Backup cấu hình

```
# Backup toàn bộ cấu hình SELinux
sudo semanage export > /backup/selinux_nginx_redis_$(date +%Y%m%d).txt

# Backup file contexts
sudo semanage fcontext -l | grep -E "(mysite|redis)" > /backup/custom_contexts.txt

# Backup boolean settings
sudo getsebool -a | grep httpd > /backup/httpd_booleans.txt
```

### 5.3 Script restore

```
# Tạo script restore
sudo tee /opt/scripts/restore_selinux_config.sh << 'EOF'
#!/bin/bash

echo "Restoring SELinux configuration for Nginx and Redis..."

# Nginx contexts
semanage fcontext -a -t httpd_exec_t "/opt/websites/mysite/public(/.*)?"
semanage fcontext -a -t httpd_log_t "/opt/websites/mysite/logs(/.*)?"
semanage fcontext -a -t httpd_cache_t "/opt/websites/mysite/cache(/.*)?"
semanage fcontext -a -t httpd_config_t "/opt/websites/mysite/ssl(/.*)?"

# Redis contexts
semanage fcontext -a -t redis_var_lib_t "/opt/redis/data(/.*)?"
semanage fcontext -a -t redis_log_t "/opt/redis/logs(/.*)?"
semanage fcontext -a -t redis_conf_t "/etc/redis/redis-custom.conf"

# Apply contexts
restorecon -Rv /opt/websites/mysite/
restorecon -Rv /opt/redis/
restorecon -v /etc/redis/redis-custom.conf

# Set booleans
setsebool -P httpd_can_network_connect 1
setsebool -P httpd_can_network_memcache 1
setsebool -P httpd_can_network_connect_db 1

echo "SELinux configuration restored successfully!"
EOF

sudo chmod +x /opt/scripts/restore_selinux_config.sh
```

## Tổng kết

Sau khi hoàn thành, hệ thống của bạn sẽ có:

### Nginx:

- ✅ Serve static files từ `/opt/websites/mysite/public`
- ✅ Proxy requests tới backend application
- ✅ SSL/TLS support
- ✅ Custom logging
- ✅ Rate limiting

### Redis:

- ✅ Custom data directory `/opt/redis/data`
- ✅ Authentication enabled
- ✅ Persistence configured
- ✅ Security hardened

### SELinux:

- ✅ Enforcing mode active
- ✅ Proper contexts cho tất cả files/directories
- ✅ Network permissions configured
- ✅ Monitoring và backup scripts

Cả hai services đều chạy an toàn với SELinux enforcing mode!

NGINX SELinux Setup:

File Contexts:

Document root: `/opt/websites/mysite/public` → `httpd_exec_t`

Logs: `/opt/websites/mysite/logs` → `httpd_log_t`

Cache: `/opt/websites/mysite/cache` → `httpd_cache_t`

SSL certs: `/opt/websites/mysite/ssl` → `httpd_config_t`

SELinux Booleans:

`httpd_can_network_connect=1` (proxy to backend)

`httpd_can_network_memcache=1` (connect to Redis)

`httpd_can_network_connect_db=1` (database connections)

REDIS SELinux Setup:

File Contexts:

Data dir: `/opt/redis/data` → `redis_var_lib_t`

Logs: `/opt/redis/logs` → `redis_log_t`

Config: `/etc/redis/redis-custom.conf` → `redis_conf_t`

Port Context:

Port `6379` → `redis_port_t` (mặc định)