# Hướng dẫn sử dụng nmon trên RHEL 8 và nmon Analyzer

## Mục lục
1. [Cài đặt nmon trên RHEL 8](#cài-đặt-nmon-trên-rhel-8)
2. [Giám sát hệ thống real-time](#giám-sát-hệ-thống-real-time)
3. [Thu thập dữ liệu cho báo cáo](#thu-thập-dữ-liệu-cho-báo-cáo)
4. [Cài đặt và sử dụng nmon Analyzer](#cài-đặt-và-sử-dụng-nmon-analyzer)
5. [Tạo báo cáo Excel chi tiết](#tạo-báo-cáo-excel-chi-tiết)
6. [Tự động hóa quy trình](#tự-động-hóa-quy-trình)
7. [Troubleshooting](#troubleshooting)

---

## Cài đặt nmon trên RHEL 8

### Bước 1: Cập nhật hệ thống

```
sudo dnf update -y
```

### Bước 2: Cài đặt EPEL repository

```
# Cài đặt EPEL repository
sudo dnf install -y epel-release

# Hoặc cài đặt trực tiếp từ Fedora
sudo dnf install -y https://dl.fedoraproject.org/pub/epel/epel-release-latest-8.noarch.rpm
```

### Bước 3: Cài đặt nmon

```
# Cài đặt nmon từ EPEL
sudo dnf install -y nmon

# Kiểm tra version
nmon -h
```

### Bước 4: Xác minh cài đặt

```
# Kiểm tra nmon đã cài đặt
which nmon
nmon -V
```

---

## Giám sát hệ thống real-time

### Khởi động nmon interactive mode

```
# Chạy nmon
nmon
```

### Các phím điều khiển chính

| Phím | Chức năng | Mô tả |
|------|-----------|--------|
| **c** | CPU Usage | Hiển thị sử dụng CPU theo core |
| **m** | Memory | Hiển thị memory usage |
| **d** | Disk I/O | Hiển thị disk statistics |
| **n** | Network | Hiển thị network I/O |
| **t** | Top Processes | Hiển thị top processes |
| **k** | Kernel | Hiển thị kernel statistics |
| **r** | Resources | Hiển thị resource summary |
| **j** | Filesystems | Hiển thị filesystem usage |
| **l** | Long term averages | Hiển thị long-term CPU average |
| **V** | Virtual Memory | Hiển thị VM stats |
| **g** | User Groups | Hiển thị user defined groups |
| **h** | Help | Hiển thị help screen |
| **q** | Quit | Thoát nmon |

### Các thông số quan trọng cần theo dõi

**CPU Metrics:**
- User%: Thời gian CPU dành cho user processes
- Sys%: Thời gian CPU dành cho system processes  
- Wait%: Thời gian CPU chờ I/O
- Idle%: Thời gian CPU không hoạt động

**Memory Metrics:**
- Total: Tổng RAM
- Used: RAM đã sử dụng
- Free: RAM còn trống
- Buffers: Buffer cache
- Cached: Page cache

**Disk Metrics:**
- Read KB/s: Tốc độ đọc
- Write KB/s: Tốc độ ghi
- IO/s: Operations per second
- Busy%: Phần trăm disk busy

---

## Thu thập dữ liệu cho báo cáo

### Tạo thư mục lưu trữ dữ liệu

```
sudo mkdir -p /var/log/nmon
sudo chown $USER:$USER /var/log/nmon
```

### Thu thập dữ liệu với các khoảng thời gian khác nhau

**Thu thập dữ liệu trong 1 giờ (mỗi 30 giây):**

```
nmon -f -s 30 -c 120 -m /var/log/nmon
```

**Thu thập dữ liệu trong 24 giờ (mỗi 5 phút):**

```
nmon -f -s 300 -c 288 -m /var/log/nmon
```

**Thu thập dữ liệu trong 1 tuần (mỗi 15 phút):**

```
nmon -f -s 900 -c 672 -m /var/log/nmon
```

**Thu thập dữ liệu với timestamp:**

```
nmon -f -T -s 60 -c 1440 -m /var/log/nmon
```

### Các tham số nmon quan trọng

| Tham số | Mô tả | Ví dụ |
|---------|-------|-------|
| `-f` | Tạo file output | `-f` |
| `-s` | Khoảng thời gian giữa các lần thu thập (giây) | `-s 60` |
| `-c` | Số lần thu thập | `-c 1440` |
| `-T` | Thêm timestamp vào tên file | `-T` |
| `-m` | Thư mục lưu file output | `-m /var/log/nmon` |
| `-p` | Include process data | `-p` |
| `-P` | Include all process data | `-P` |
| `-d` | Include disk service time | `-d` |
| `-g` | Include disk groups | `-g filename` |

### Tạo script thu thập dữ liệu tự động

```
#!/bin/bash
# /usr/local/bin/nmon_collector.sh

# Cấu hình
NMON_DIR="/var/log/nmon"
HOSTNAME=$(hostname -s)
DATE=$(date +%Y%m%d_%H%M%S)
LOG_FILE="$NMON_DIR/nmon_collector.log"

# Tạo thư mục nếu chưa có
mkdir -p $NMON_DIR

# Function ghi log
log_message() {
    echo "$(date '+%Y-%m-%d %H:%M:%S') - $1" >> $LOG_FILE
}

# Bắt đầu thu thập
log_message "Starting nmon data collection for $HOSTNAME"

# Thu thập dữ liệu trong 24 giờ, mỗi 5 phút
nmon -f -T -s 300 -c 288 -m $NMON_DIR -p

log_message "nmon data collection started. PID: $(pgrep nmon)"
log_message "Collection will run for 24 hours with 5-minute intervals"

echo "nmon data collection started successfully!"
echo "Log file: $LOG_FILE"
echo "Data directory: $NMON_DIR"
```

### Thiết lập quyền và chạy script

```
sudo chmod +x /usr/local/bin/nmon_collector.sh
sudo /usr/local/bin/nmon_collector.sh
```

---

## Cài đặt và sử dụng nmon Analyzer

### Bước 1: Tải nmon Analyzer

**Từ IBM Developer:**

```
# Tải về máy Windows hoặc Linux có GUI
wget https://www.ibm.com/developerworks/aix/library/au-analyze_aix/nmon_analyser_v66.xlsm
```

**Từ GitHub alternative:**

```
git clone https://github.com/power-devops/nmon-for-linux.git
cd nmon-for-linux
# File analyzer sẽ ở trong thư mục tools/
```

### Bước 2: Chuẩn bị môi trường

**Trên Windows:**
- Microsoft Excel 2010 hoặc mới hơn
- Enable Macros trong Excel

**Trên Linux:**
- LibreOffice Calc với macro support

```
sudo dnf install -y libreoffice-calc
```

### Bước 3: Cấu trúc file nmon

File nmon có cấu trúc như sau:

```
AAA,progname,nmon
AAA,command,nmon -f -T -s 300 -c 288
AAA,version,16i
AAA,disks,2
BBBP,T0001,15:30:00,15-DEC-2023
CPU_ALL,T0001,15.2,8.1,2.3,74.4
MEM,T0001,8192,4096,2048,1024,1024
DISKBUSY,T0001,sda,12.5,sdb,8.3
NET,T0001,eth0,1024,2048,0,0
```

### Bước 4: Chuyển file từ RHEL 8 sang máy có Excel

**Sử dụng SCP:**

```
# Từ máy Windows (với putty/winscp)
scp user@rhel8-server:/var/log/nmon/*.nmon C:\nmon_data\

# Từ máy Linux
scp user@rhel8-server:/var/log/nmon/*.nmon /home/user/nmon_data/
```

**Sử dụng rsync:**

```
rsync -avz user@rhel8-server:/var/log/nmon/ /local/nmon_data/
```

---

## Tạo báo cáo Excel chi tiết

### Bước 1: Mở nmon Analyzer

1. Mở file `nmon_analyser_v66.xlsm`
2. Khi được hỏi về Macros, chọn **"Enable Macros"**
3. Đợi analyzer load hoàn tất

### Bước 2: Import dữ liệu nmon

**Phương pháp 1: Sử dụng button "Analyse nmon data"**
1. Click vào button **"Analyse nmon data"** 
2. Browse và chọn file `.nmon` của bạn
3. Chờ analyzer parse dữ liệu (có thể mất vài phút)

**Phương pháp 2: Manual import**
1. Mở file `.nmon` bằng notepad
2. Select All và Copy
3. Paste vào sheet **"nmon data"** trong analyzer
4. Click **"Analyse nmon data"**

### Bước 3: Các sheet phân tích chính

**Sheet "SYS_SUMM" (System Summary):**
- Hostname và thông tin hệ thống
- OS version và kernel version
- Hardware specifications
- Thời gian thu thập dữ liệu
- Tổng quan các metrics

**Sheet "CPU_ALL" (CPU Analysis):**
- CPU utilization charts theo thời gian
- User%, System%, Wait%, Idle%
- CPU load trends
- Peak CPU usage times

**Sheet "MEM" (Memory Analysis):**
- Memory usage over time
- Total, Used, Free, Buffers, Cached
- Memory utilization percentage
- Memory trends và patterns

**Sheet "DISK_BUSY" (Disk Activity):**
- Disk I/O statistics
- Read/Write operations per second
- Disk utilization percentage
- Disk performance trends

**Sheet "NET" (Network Analysis):**
- Network throughput charts
- Packets per second
- Bytes per second
- Network errors và retransmissions

### Bước 4: Tùy chỉnh báo cáo

**Thay đổi thời gian hiển thị:**
```excel
1. Click vào chart cần chỉnh sửa
2. Right-click → "Select Data"
3. Chọn "Edit" cho Category Labels
4. Điều chỉnh range thời gian: =Sheet1!$A$2:$A$100
```

**Thêm calculated fields:**
```excel
// CPU Average
=AVERAGE(CPU_ALL!B:B)

// Memory Peak Usage
=MAX(MEM!C:C)

// Disk I/O Average
=AVERAGE(DISK_BUSY!D:D)

// Network Throughput Average
=AVERAGE(NET!E:E)
```

**Tạo conditional formatting:**
```excel
1. Select data range
2. Home → Conditional Formatting
3. New Rule → Use Formula
4. Formula: =B2>80 (for CPU > 80%)
5. Format: Red background
```

### Bước 5: Tạo Dashboard tổng hợp

**Tạo sheet mới "Executive Dashboard":**
1. Insert → New Sheet
2. Rename thành "Executive Dashboard"
3. Copy các charts quan trọng từ sheets khác
4. Resize và arrange theo layout mong muốn

**Thêm summary metrics:**
```excel
// System Information
Server Name: =SYS_SUMM!B2
OS Version: =SYS_SUMM!B3
Collection Period: =SYS_SUMM!B4

// Performance Summary
Average CPU: =AVERAGE(CPU_ALL!B:B)
Peak CPU: =MAX(CPU_ALL!B:B)
Average Memory: =AVERAGE(MEM!C:C)
Peak Memory: =MAX(MEM!C:C)
```

**Thêm alerts và thresholds:**
```excel
// CPU Alert
=IF(AVERAGE(CPU_ALL!B:B)>80,"HIGH","NORMAL")

// Memory Alert  
=IF(MAX(MEM!C:C)>90,"CRITICAL","OK")

// Disk Alert
=IF(MAX(DISK_BUSY!D:D)>80,"WARNING","OK")
```

---

## Tự động hóa quy trình

### Script tự động thu thập và xử lý


```
#!/bin/bash
# /usr/local/bin/nmon_automation.sh

# Cấu hình
NMON_DIR="/var/log/nmon"
ARCHIVE_DIR="/var/log/nmon/archive"
REPORT_DIR="/var/log/nmon/reports"
DAYS_TO_KEEP=7
EMAIL_RECIPIENTS="admin@company.com"

# Tạo thư mục
mkdir -p $NMON_DIR $ARCHIVE_DIR $REPORT_DIR

# Function thu thập dữ liệu
collect_data() {
    local duration=$1
    local interval=$2
    local count=$((duration * 60 / interval))
    
    echo "Starting nmon collection for $duration minutes with $interval second intervals"
    nmon -f -T -s $interval -c $count -m $NMON_DIR -p
    
    echo "Collection started. PID: $(pgrep nmon)"
}

# Function dọn dẹp file cũ
cleanup_old_files() {
    find $NMON_DIR -name "*.nmon" -mtime +$DAYS_TO_KEEP -exec mv {} $ARCHIVE_DIR \;
    find $ARCHIVE_DIR -name "*.nmon" -mtime +30 -delete
    
    echo "Cleanup completed. Old files moved to archive."
}

# Function gửi email báo cáo
send_report() {
    local subject="Daily nmon Report - $(hostname) - $(date +%Y-%m-%d)"
    local body="Please find attached the daily system performance report."
    
    # Tạo summary report
    cat > $REPORT_DIR/daily_summary.txt << EOF
System Performance Summary - $(date)
===================================
Hostname: $(hostname)
Uptime: $(uptime)
Load Average: $(cat /proc/loadavg)
Memory Usage: $(free -h | grep Mem)
Disk Usage: $(df -h / | tail -1)
EOF
    
    # Gửi email (cần cài đặt mailutils)
    mail -s "$subject" $EMAIL_RECIPIENTS < $REPORT_DIR/daily_summary.txt
}

# Main execution
case "$1" in
    "start")
        collect_data 1440 300  # 24 hours, 5 minutes
        ;;
    "short")
        collect_data 60 30     # 1 hour, 30 seconds
        ;;
    "cleanup")
        cleanup_old_files
        ;;
    "report")
        send_report
        ;;
    *)
        echo "Usage: $0 {start|short|cleanup|report}"
        exit 1
        ;;
esac
```

### Cron job tự động


```
# Cài đặt cron job
crontab -e

# Thêm các dòng sau:
# Thu thập dữ liệu hàng ngày lúc 00:00
0 0 * * * /usr/local/bin/nmon_automation.sh start

# Dọn dẹp file cũ hàng tuần
0 2 * * 0 /usr/local/bin/nmon_automation.sh cleanup

# Gửi báo cáo hàng ngày lúc 08:00
0 8 * * * /usr/local/bin/nmon_automation.sh report
```

### Systemd service cho nmon


```
# Tạo service file
sudo tee /etc/systemd/system/nmon-collector.service << EOF
[Unit]
Description=nmon Data Collector
After=network.target

[Service]
Type=forking
ExecStart=/usr/bin/nmon -f -T -s 300 -c 288 -m /var/log/nmon -p
ExecStop=/usr/bin/killall nmon
Restart=always
RestartSec=60
User=nmon
Group=nmon

[Install]
WantedBy=multi-user.target
EOF

# Tạo user cho service
sudo useradd -r -s /bin/false nmon
sudo mkdir -p /var/log/nmon
sudo chown nmon:nmon /var/log/nmon

# Enable và start service
sudo systemctl daemon-reload
sudo systemctl enable nmon-collector.service
sudo systemctl start nmon-collector.service

# Kiểm tra status
sudo systemctl status nmon-collector.service
```

---

## Troubleshooting

### Lỗi thường gặp và cách khắc phục

**1. nmon không khởi động được**

```
# Kiểm tra quyền
ls -la /usr/bin/nmon
chmod +x /usr/bin/nmon

# Kiểm tra dependencies
ldd /usr/bin/nmon
```

**2. File nmon quá lớn**

```
# Chia file thành chunks nhỏ hơn
split -l 10000 large_file.nmon split_file_

# Hoặc filter chỉ lấy dữ liệu cần thiết
awk '/AAA|BBB|CPU|MEM|DISK/' large_file.nmon > filtered_file.nmon
```

**3. Excel Analyzer không parse được dữ liệu**

```
# Kiểm tra format file nmon
head -20 file.nmon
tail -20 file.nmon

# Kiểm tra encoding
file -bi file.nmon

# Convert encoding nếu cần
iconv -f ISO-8859-1 -t UTF-8 file.nmon > file_utf8.nmon
```

**4. Macros không hoạt động trong Excel**
```
Giải pháp:
1. File → Options → Trust Center → Trust Center Settings
2. Macro Settings → Enable all macros
3. Trusted Locations → Add location chứa file analyzer
```

**5. Disk space đầy do file nmon**

```
# Kiểm tra disk usage
du -sh /var/log/nmon/*

# Compress file cũ
gzip /var/log/nmon/*.nmon

# Thiết lập log rotation
sudo tee /etc/logrotate.d/nmon << EOF
/var/log/nmon/*.nmon {
    daily
    rotate 7
    compress
    delaycompress
    missingok
    notifempty
    create 0644 nmon nmon
}
EOF
```

### Performance tuning cho nmon

**Tối ưu hóa thu thập dữ liệu:**

```
# Chỉ thu thập metrics cần thiết
nmon -f -s 60 -c 1440 -m /var/log/nmon \
     -c  # CPU only
     -m  # Memory only  
     -d  # Disk only
     -n  # Network only
```

**Tối ưu hóa storage:**

```
# Sử dụng compression
nmon -f -s 60 -c 1440 -m /var/log/nmon | gzip > output.nmon.gz

# Sử dụng ramdisk cho high-frequency collection
sudo mkdir /mnt/ramdisk
sudo mount -t tmpfs -o size=512M tmpfs /mnt/ramdisk
nmon -f -s 1 -c 3600 -m /mnt/ramdisk
```

---

## Kết luận

Hướng dẫn này cung cấp quy trình hoàn chỉnh để:

1. **Cài đặt và cấu hình nmon** trên RHEL 8
2. **Thu thập dữ liệu** hiệu suất hệ thống
3. **Sử dụng nmon Analyzer** để tạo báo cáo Excel chuyên nghiệp
4. **Tự động hóa** quy trình giám sát
5. **Khắc phục** các vấn đề thường gặp

Kết quả cuối cùng là một hệ thống giám sát hoàn chỉnh với báo cáo Excel tự động, giúp theo dõi và phân tích hiệu suất hệ thống một cách hiệu quả.

### Lợi ích chính:
- **Giám sát toàn diện**: CPU, Memory, Disk, Network
- **Báo cáo trực quan**: Charts và graphs chi tiết
- **Tự động hóa**: Không cần can thiệp thủ công
- **Lưu trữ lâu dài**: Dữ liệu lịch sử để phân tích trends
- **Cảnh báo**: Thông báo khi có anomalies

### Các bước tiếp theo:
1. Triển khai monitoring cho múltiple servers
2. Tích hợp với centralized logging systems
3. Tạo dashboards real-time với Grafana
4. Implement alerting với Nagios/Zabbix