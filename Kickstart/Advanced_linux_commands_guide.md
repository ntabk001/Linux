# Hướng dẫn sử dụng các lệnh Linux nâng cao

## 1. Lệnh `yum` và `dnf` - Quản lý gói package

### YUM (CentOS/RHEL cũ) và DNF (Fedora/RHEL 8+/CentOS 8+)

### Cú pháp cơ bản:

```
yum [options] [command] [package...]
dnf [options] [command] [package...]
```

### Các ví dụ sử dụng cụ thể:

**Cài đặt package:**

```
# Cài đặt web server
yum install httpd
dnf install nginx

# Cài đặt nhiều package cùng lúc
yum install vim wget curl git
dnf install python3 python3-pip nodejs npm
```

**Cập nhật hệ thống:**

```
# Cập nhật tất cả packages
yum update
dnf upgrade

# Cập nhật chỉ security patches
yum update --security
dnf upgrade --security
```

**Tìm kiếm và xem thông tin:**

```
# Tìm package chứa keyword
yum search database
dnf search "web server"

# Xem thông tin chi tiết package
yum info mysql-server
dnf info postgresql-server

# Xem package nào cung cấp file
yum provides /usr/bin/docker
dnf provides */netstat
```

**Quản lý repositories:**

```
# Liệt kê repositories
yum repolist
dnf repolist

# Cài đặt package từ repo cụ thể
yum install --enablerepo=epel htop
dnf install --enablerepo=powertools development-tools
```

**Ví dụ thực tế:**

```
# Cài đặt LAMP stack
yum install httpd mariadb-server php php-mysql
systemctl start httpd mariadb
systemctl enable httpd mariadb

# Cài đặt Docker
dnf config-manager --add-repo=https://download.docker.com/linux/centos/docker-ce.repo
dnf install docker-ce docker-ce-cli containerd.io

# Xem lịch sử cài đặt và rollback
yum history
yum history undo 5
```

## 2. Lệnh `ps` - Xem processes đang chạy

### Cú pháp cơ bản:

```
ps [options]
```

### Các ví dụ sử dụng cụ thể:

**Xem processes cơ bản:**

```
# Xem process của user hiện tại
ps

# Xem tất cả processes
ps aux

# Xem processes dạng tree
ps auxf
```

**Tìm kiếm process cụ thể:**

```
# Tìm process theo tên
ps aux | grep apache
ps aux | grep mysql

# Tìm process theo PID
ps -p 1234

# Tìm process theo user
ps -u apache
ps -u root | grep ssh
```

**Xem thông tin chi tiết:**

```
# Xem với nhiều thông tin
ps -eo pid,ppid,cmd,user,time,cpu,mem

# Xem process theo memory usage
ps aux --sort=-%mem | head -10

# Xem process theo CPU usage
ps aux --sort=-%cpu | head -10
```

**Ví dụ thực tế:**

```
# Kiểm tra web server có chạy không
ps aux | grep httpd | grep -v grep

# Tìm process zombie
ps aux | grep zombie

# Xem process của database
ps aux | grep -E "(mysql|postgres|mongo)"

# Kill process theo tên
ps aux | grep "process_name" | grep -v grep | awk '{print $2}' | xargs kill
```

## 3. Lệnh `diff` - So sánh file

### Cú pháp cơ bản:

```
diff [options] file1 file2
```

### Các ví dụ sử dụng cụ thể:

**So sánh cơ bản:**

```
# So sánh 2 file
diff file1.txt file2.txt

# So sánh với output dễ đọc
diff -u file1.txt file2.txt

# So sánh side-by-side
diff -y file1.txt file2.txt
```

**So sánh thư mục:**

```
# So sánh 2 thư mục
diff -r dir1/ dir2/

# So sánh và hiển thị chỉ tên file khác nhau
diff -rq dir1/ dir2/
```

**Các options hữu ích:**

```
# Bỏ qua whitespace
diff -w file1.txt file2.txt

# Bỏ qua case sensitivity
diff -i file1.txt file2.txt

# Tạo patch file
diff -u original.txt modified.txt > changes.patch
```

**Ví dụ thực tế:**

```
# So sánh config files
diff /etc/httpd/conf/httpd.conf /etc/httpd/conf/httpd.conf.bak

# So sánh trước và sau khi cập nhật
diff -u /etc/passwd.old /etc/passwd

# So sánh code repositories
diff -r --exclude=".git" project1/ project2/

# Kiểm tra thay đổi trong log
diff yesterday.log today.log | grep "ERROR"
```

## 4. Lệnh `dig` - DNS lookup tool

### Cú pháp cơ bản:

```
dig [options] [domain] [record_type]
```

### Các ví dụ sử dụng cụ thể:

**Truy vấn DNS cơ bản:**

```
# Truy vấn A record
dig google.com

# Truy vấn record type cụ thể
dig google.com MX
dig google.com NS
dig google.com TXT
```

**Truy vấn với DNS server cụ thể:**

```
# Sử dụng Google DNS
dig @8.8.8.8 example.com

# Sử dụng Cloudflare DNS
dig @1.1.1.1 example.com A

# Sử dụng DNS server local
dig @192.168.1.1 internal.domain.com
```

**Reverse DNS lookup:**

```
# Tra cứu ngược từ IP
dig -x 8.8.8.8
dig -x 192.168.1.1
```

**Ví dụ thực tế:**

```
# Troubleshoot DNS issues
dig +trace google.com

# Kiểm tra mail server
dig gmail.com MX

# Kiểm tra CDN setup
dig cdn.example.com CNAME

# Kiểm tra SPF record
dig example.com TXT | grep "v=spf1"

# Benchmark DNS response time
dig google.com | grep "Query time"
```

## 5. Lệnh `xargs` - Xử lý arguments từ input

### Cú pháp cơ bản:

```
xargs [options] [command]
```

### Các ví dụ sử dụng cụ thể:

**Xử lý file:**

```
# Xóa nhiều file
find . -name "*.tmp" | xargs rm

# Tìm và xóa file với xác nhận
find . -name "*.log" -print0 | xargs -0 -I {} rm -i {}

# Copy nhiều file
ls *.txt | xargs -I {} cp {} backup/
```

**Xử lý với command khác:**

```
# Chạy command cho mỗi item
echo "file1 file2 file3" | xargs -n 1 ls -l

# Chạy parallel processing
echo "1 2 3 4 5" | xargs -n 1 -P 3 sleep
```

**Ví dụ thực tế:**

```
# Tìm và kill processes
ps aux | grep "process_name" | awk '{print $2}' | xargs kill

# Backup multiple databases
mysql -e "SHOW DATABASES" | tail -n +2 | xargs -I {} mysqldump {} > {}.sql

# Download multiple URLs
cat urls.txt | xargs -n 1 -P 5 wget

# Find and compress files
find /var/log -name "*.log" -mtime +7 | xargs gzip

# Change permissions on multiple files
find . -name "*.sh" | xargs chmod +x
```

## 6. Lệnh `top` - Monitor system real-time

### Cú pháp cơ bản:

```
top [options]
```

### Các phím tắt trong top:
- `q`: Thoát
- `k`: Kill process
- `r`: Renice process
- `P`: Sắp xếp theo CPU
- `M`: Sắp xếp theo Memory
- `T`: Sắp xếp theo Time
- `u`: Lọc theo user
- `1`: Hiển thị từng CPU core

### Các ví dụ sử dụng cụ thể:

**Monitor cơ bản:**

```
# Chạy top standard
top

# Chạy với update interval
top -d 5

# Chạy với số lần update cụ thể
top -n 3
```

**Monitor user cụ thể:**

```
# Monitor processes của user
top -u apache
top -u mysql

# Monitor với PID cụ thể
top -p 1234,5678
```

**Ví dụ thực tế:**

```
# Monitor và save output
top -b -n 1 > system_snapshot.txt

# Monitor database performance
top -u mysql -d 2

# Monitor web server
top -u apache -d 1

# Batch mode for logging
top -b -d 60 >> performance.log &
```

## 7. Lệnh `mount` - Mount/unmount filesystems

### Cú pháp cơ bản:

```
mount [options] device mountpoint
umount [options] mountpoint
```

### Các ví dụ sử dụng cụ thể:

**Mount cơ bản:**

```
# Mount USB drive
mount /dev/sdb1 /mnt/usb

# Mount với filesystem type
mount -t ext4 /dev/sdb1 /mnt/disk

# Mount CD/DVD
mount /dev/cdrom /mnt/cdrom
```

**Mount với options:**

```
# Mount read-only
mount -o ro /dev/sdb1 /mnt/readonly

# Mount với permissions
mount -o uid=1000,gid=1000 /dev/sdb1 /mnt/usb

# Mount network filesystem
mount -t nfs 192.168.1.100:/share /mnt/nfs
```

**Ví dụ thực tế:**

```
# Mount và auto-mount
echo "/dev/sdb1 /mnt/backup ext4 defaults 0 0" >> /etc/fstab
mount /mnt/backup

# Mount ISO file
mount -o loop disk.iso /mnt/iso

# Mount SMB/CIFS share
mount -t cifs //server/share /mnt/share -o username=user,password=pass

# Unmount safely
umount /mnt/usb
# Hoặc force unmount
umount -f /mnt/usb
```

## 8. Lệnh `tcpdump` - Capture network traffic

### Cú pháp cơ bản:

```
tcpdump [options] [filter]
```

### Các ví dụ sử dụng cụ thể:

**Capture cơ bản:**

```
# Capture tất cả traffic
tcpdump

# Capture trên interface cụ thể
tcpdump -i eth0

# Capture và save to file
tcpdump -w capture.pcap
```

**Filter traffic:**

```
# Capture HTTP traffic
tcpdump port 80

# Capture SSH traffic
tcpdump port 22

# Capture traffic từ/đến IP cụ thể
tcpdump host 192.168.1.100

# Capture traffic between 2 hosts
tcpdump host 192.168.1.100 and 192.168.1.200
```

**Ví dụ thực tế:**

```
# Debug web server issues
tcpdump -i eth0 port 80 -A

# Monitor database connections
tcpdump -i eth0 port 3306 -c 100

# Capture DNS queries
tcpdump -i eth0 port 53 -v

# Monitor mail server
tcpdump -i eth0 port 25 or port 110 or port 143

# Capture với timestamp
tcpdump -i eth0 -tttt port 80 > web_traffic.log
```

## 9. Lệnh `lsof` - List open files

### Cú pháp cơ bản:

```
lsof [options] [names]
```

### Các ví dụ sử dụng cụ thể:

**List files cơ bản:**

```
# List tất cả open files
lsof

# List files opened by user
lsof -u apache
lsof -u mysql

# List files opened by process
lsof -p 1234
```

**Network connections:**

```
# List network connections
lsof -i

# List connections on port
lsof -i :80
lsof -i :22

# List TCP connections
lsof -i tcp
lsof -i udp
```

**Ví dụ thực tế:**

```
# Tìm process sử dụng port
lsof -i :80
lsof -i :3306

# Tìm file được mở bởi process
lsof -c httpd
lsof -c mysql

# Tìm process sử dụng file
lsof /var/log/messages
lsof /etc/passwd

# Troubleshoot "device busy" errors
lsof /mnt/usb

# Monitor log file access
lsof +D /var/log/
```

## 10. Lệnh `scp` - Secure copy over SSH

### Cú pháp cơ bản:

```
scp [options] source destination
```

### Các ví dụ sử dụng cụ thể:

**Copy file:**

```
# Copy file to remote server
scp file.txt user@server:/path/to/destination/

# Copy file from remote server
scp user@server:/path/to/file.txt ./

# Copy với port SSH khác
scp -P 2222 file.txt user@server:/path/
```

**Copy directory:**

```
# Copy directory recursively
scp -r directory/ user@server:/path/

# Copy với preserve permissions
scp -rp directory/ user@server:/path/
```

**Ví dụ thực tế:**

```
# Backup database
mysqldump database | gzip | scp - user@backup-server:/backups/db_$(date +%Y%m%d).sql.gz

# Copy log files
scp /var/log/httpd/access.log admin@log-server:/logs/web/

# Copy SSH keys
scp ~/.ssh/id_rsa.pub user@server:~/.ssh/authorized_keys

# Copy với compression
scp -C large_file.tar.gz user@server:/tmp/

# Copy multiple files
scp file1.txt file2.txt file3.txt user@server:/path/
```

## Kết hợp các lệnh

### Pipeline examples:


```
# Find high CPU processes và kill chúng
ps aux --sort=-%cpu | head -5 | awk '{print $2}' | xargs kill

# Monitor network connections và log
lsof -i | grep ESTABLISHED | tee network_connections.log

# Backup và verify
tar czf backup.tar.gz /important/data && 
scp backup.tar.gz user@backup-server:/backups/ && 
ssh user@backup-server "cd /backups && tar -tzf backup.tar.gz > /dev/null && echo 'Backup verified'"

# System health check
(ps aux --sort=-%cpu | head -10; echo "---"; lsof -i | grep LISTEN; echo "---"; df -h) | mail -s "System Report" admin@company.com
```

### Tips quan trọng:

1. **Luôn test commands** trước khi chạy trên production
2. **Sử dụng man pages** để tìm hiểu thêm options
3. **Kết hợp với grep, awk, sed** để xử lý output
4. **Sử dụng screen/tmux** cho các lệnh chạy lâu như tcpdump
5. **Backup trước khi sử dụng** các lệnh có thể thay đổi hệ thống