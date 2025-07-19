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

## Tạo báo cáo từ Command Line Linux

### Hiểu cấu trúc file nmon

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

### Bước 1: Cài đặt các công cụ cần thiết


```
# Cài đặt các package cần thiết
sudo dnf install -y gawk sed grep coreutils bc gnuplot

# Cài đặt Python và pandas (optional cho advanced analysis)
sudo dnf install -y python3 python3-pip
pip3 install pandas matplotlib seaborn
```

### Bước 2: Tạo script parser cho nmon data


```
#!/bin/bash
# /usr/local/bin/nmon_parser.sh

NMON_FILE="$1"
OUTPUT_DIR="$2"
REPORT_NAME="$(basename $NMON_FILE .nmon)"

if [ -z "$NMON_FILE" ] || [ -z "$OUTPUT_DIR" ]; then
    echo "Usage: $0 <nmon_file> <output_directory>"
    exit 1
fi

mkdir -p "$OUTPUT_DIR"

# Parse system information
parse_system_info() {
    echo "=== SYSTEM INFORMATION ===" > "$OUTPUT_DIR/${REPORT_NAME}_system_info.txt"
    grep "^AAA," "$NMON_FILE" | while IFS=, read -r tag key value; do
        case $key in
            "host") echo "Hostname: $value" ;;
            "user") echo "User: $value" ;;
            "OS") echo "Operating System: $value" ;;
            "version") echo "OS Version: $value" ;;
            "kernel") echo "Kernel: $value" ;;
            "machine") echo "Machine: $value" ;;
            "cpus") echo "CPU Count: $value" ;;
            "memory") echo "Total Memory: $value MB" ;;
            "date") echo "Collection Date: $value" ;;
            "time") echo "Collection Time: $value" ;;
        esac
    done >> "$OUTPUT_DIR/${REPORT_NAME}_system_info.txt"
}

# Parse CPU data
parse_cpu_data() {
    echo "=== CPU ANALYSIS ===" > "$OUTPUT_DIR/${REPORT_NAME}_cpu_report.txt"
    
    # Extract CPU_ALL data
    grep "^CPU_ALL," "$NMON_FILE" | grep -v "CPU_ALL,T" > "$OUTPUT_DIR/${REPORT_NAME}_cpu_raw.csv"
    
    # Calculate CPU statistics
    awk -F',' '
    BEGIN { 
        sum_user=0; sum_sys=0; sum_wait=0; sum_idle=0; count=0;
        max_user=0; max_sys=0; max_wait=0; min_idle=100;
    }
    NR>1 {
        if(NF>=5) {
            user=$3; sys=$4; wait=$5; idle=$6;
            sum_user+=user; sum_sys+=sys; sum_wait+=wait; sum_idle+=idle;
            if(user>max_user) max_user=user;
            if(sys>max_sys) max_sys=sys;
            if(wait>max_wait) max_wait=wait;
            if(idle<min_idle) min_idle=idle;
            count++;
        }
    }
    END {
        if(count>0) {
            printf "Average CPU User: %.2f%%\n", sum_user/count;
            printf "Average CPU System: %.2f%%\n", sum_sys/count;
            printf "Average CPU Wait: %.2f%%\n", sum_wait/count;
            printf "Average CPU Idle: %.2f%%\n", sum_idle/count;
            printf "Peak CPU User: %.2f%%\n", max_user;
            printf "Peak CPU System: %.2f%%\n", max_sys;
            printf "Peak CPU Wait: %.2f%%\n", max_wait;
            printf "Minimum CPU Idle: %.2f%%\n", min_idle;
            printf "Total CPU Utilization: %.2f%%\n", 100-sum_idle/count;
        }
    }' "$OUTPUT_DIR/${REPORT_NAME}_cpu_raw.csv" >> "$OUTPUT_DIR/${REPORT_NAME}_cpu_report.txt"
}

# Parse Memory data
parse_memory_data() {
    echo "=== MEMORY ANALYSIS ===" > "$OUTPUT_DIR/${REPORT_NAME}_memory_report.txt"
    
    # Extract MEM data
    grep "^MEM," "$NMON_FILE" | grep -v "MEM,T" > "$OUTPUT_DIR/${REPORT_NAME}_memory_raw.csv"
    
    # Calculate Memory statistics
    awk -F',' '
    BEGIN { 
        sum_total=0; sum_used=0; sum_free=0; sum_buffers=0; sum_cached=0; count=0;
        max_used=0; min_free=999999;
    }
    NR>1 {
        if(NF>=6) {
            total=$3; used=$4; free=$5; buffers=$6; cached=$7;
            sum_total+=total; sum_used+=used; sum_free+=free; 
            sum_buffers+=buffers; sum_cached+=cached;
            if(used>max_used) max_used=used;
            if(free<min_free) min_free=free;
            count++;
        }
    }
    END {
        if(count>0) {
            printf "Average Total Memory: %.2f MB\n", sum_total/count;
            printf "Average Used Memory: %.2f MB\n", sum_used/count;
            printf "Average Free Memory: %.2f MB\n", sum_free/count;
            printf "Average Buffers: %.2f MB\n", sum_buffers/count;
            printf "Average Cached: %.2f MB\n", sum_cached/count;
            printf "Peak Memory Usage: %.2f MB\n", max_used;
            printf "Minimum Free Memory: %.2f MB\n", min_free;
            printf "Average Memory Utilization: %.2f%%\n", (sum_used/count)/(sum_total/count)*100;
        }
    }' "$OUTPUT_DIR/${REPORT_NAME}_memory_raw.csv" >> "$OUTPUT_DIR/${REPORT_NAME}_memory_report.txt"
}

# Parse Disk data
parse_disk_data() {
    echo "=== DISK ANALYSIS ===" > "$OUTPUT_DIR/${REPORT_NAME}_disk_report.txt"
    
    # Extract DISKBUSY data
    grep "^DISKBUSY," "$NMON_FILE" | grep -v "DISKBUSY,T" > "$OUTPUT_DIR/${REPORT_NAME}_disk_raw.csv"
    
    # Get disk names
    DISKS=$(grep "^DISKBUSY,T" "$NMON_FILE" | head -1 | cut -d',' -f3- | tr ',' '\n' | grep -v "^$" | sort -u)
    
    for disk in $DISKS; do
        echo "--- Disk: $disk ---" >> "$OUTPUT_DIR/${REPORT_NAME}_disk_report.txt"
        
        # Find column position for this disk
        COL=$(grep "^DISKBUSY,T" "$NMON_FILE" | head -1 | tr ',' '\n' | grep -n "^$disk$" | cut -d: -f1)
        if [ -n "$COL" ]; then
            COL=$((COL + 2))  # Adjust for CSV structure
            
            awk -F',' -v col=$COL '
            BEGIN { sum=0; count=0; max=0; }
            NR>1 && NF>=col {
                val=$col;
                if(val != "" && val > 0) {
                    sum+=val; count++;
                    if(val>max) max=val;
                }
            }
            END {
                if(count>0) {
                    printf "Average Busy: %.2f%%\n", sum/count;
                    printf "Peak Busy: %.2f%%\n", max;
                }
            }' "$OUTPUT_DIR/${REPORT_NAME}_disk_raw.csv" >> "$OUTPUT_DIR/${REPORT_NAME}_disk_report.txt"
        fi
    done
}

# Parse Network data
parse_network_data() {
    echo "=== NETWORK ANALYSIS ===" > "$OUTPUT_DIR/${REPORT_NAME}_network_report.txt"
    
    # Extract NET data
    grep "^NET," "$NMON_FILE" | grep -v "NET,T" > "$OUTPUT_DIR/${REPORT_NAME}_network_raw.csv"
    
    # Get network interfaces
    INTERFACES=$(grep "^NET,T" "$NMON_FILE" | head -1 | cut -d',' -f3- | tr ',' '\n' | grep -v "^$" | sort -u)
    
    for interface in $INTERFACES; do
        echo "--- Interface: $interface ---" >> "$OUTPUT_DIR/${REPORT_NAME}_network_report.txt"
        
        # Process network statistics for this interface
        awk -F',' -v iface="$interface" '
        BEGIN { 
            sum_read=0; sum_write=0; count=0; max_read=0; max_write=0;
        }
        /^NET,/ && NF>=4 {
            for(i=3; i<=NF; i+=2) {
                if($i == iface) {
                    read=$(i+1); write=$(i+2);
                    sum_read+=read; sum_write+=write;
                    if(read>max_read) max_read=read;
                    if(write>max_write) max_write=write;
                    count++;
                    break;
                }
            }
        }
        END {
            if(count>0) {
                printf "Average Read: %.2f KB/s\n", sum_read/count;
                printf "Average Write: %.2f KB/s\n", sum_write/count;
                printf "Peak Read: %.2f KB/s\n", max_read;
                printf "Peak Write: %.2f KB/s\n", max_write;
            }
        }' "$NMON_FILE" >> "$OUTPUT_DIR/${REPORT_NAME}_network_report.txt"
    done
}

# Execute all parsing functions
echo "Parsing nmon file: $NMON_FILE"
parse_system_info
parse_cpu_data
parse_memory_data
parse_disk_data
parse_network_data

echo "Reports generated in: $OUTPUT_DIR"
```

### Bước 3: Tạo script tổng hợp báo cáo

```
#!/bin/bash
# /usr/local/bin/nmon_report_generator.sh

NMON_FILE="$1"
OUTPUT_DIR="$2"
REPORT_NAME="$(basename $NMON_FILE .nmon)"

if [ -z "$NMON_FILE" ] || [ -z "$OUTPUT_DIR" ]; then
    echo "Usage: $0 <nmon_file> <output_directory>"
    exit 1
fi

mkdir -p "$OUTPUT_DIR"

# Generate comprehensive HTML report
generate_html_report() {
    cat > "$OUTPUT_DIR/${REPORT_NAME}_report.html" << 'EOF'
<!DOCTYPE html>
<html>
<head>
    <title>nmon Performance Report</title>
    <style>
        body { font-family: Arial, sans-serif; margin: 20px; }
        .header { background-color: #f0f0f0; padding: 15px; border-radius: 5px; }
        .section { margin: 20px 0; padding: 15px; border: 1px solid #ddd; border-radius: 5px; }
        .metric { display: inline-block; margin: 10px; padding: 10px; background: #f9f9f9; border-radius: 3px; }
        .alert { background-color: #ffebee; border-left: 4px solid #f44336; padding: 10px; margin: 10px 0; }
        .warning { background-color: #fff3e0; border-left: 4px solid #ff9800; padding: 10px; margin: 10px 0; }
        .normal { background-color: #e8f5e8; border-left: 4px solid #4caf50; padding: 10px; margin: 10px 0; }
        table { width: 100%; border-collapse: collapse; margin: 10px 0; }
        th, td { padding: 8px; text-align: left; border-bottom: 1px solid #ddd; }
        th { background-color: #f2f2f2; }
    </style>
</head>
<body>
EOF

    # Add system information
    echo '<div class="header"><h1>System Performance Report</h1>' >> "$OUTPUT_DIR/${REPORT_NAME}_report.html"
    echo "<h2>Generated: $(date)</h2></div>" >> "$OUTPUT_DIR/${REPORT_NAME}_report.html"
    
    # Add system info section
    echo '<div class="section"><h2>System Information</h2>' >> "$OUTPUT_DIR/${REPORT_NAME}_report.html"
    echo '<table>' >> "$OUTPUT_DIR/${REPORT_NAME}_report.html"
    
    grep "^AAA," "$NMON_FILE" | while IFS=, read -r tag key value; do
        case $key in
            "host"|"OS"|"version"|"kernel"|"machine"|"cpus"|"memory"|"date"|"time")
                echo "<tr><td><strong>$key</strong></td><td>$value</td></tr>" >> "$OUTPUT_DIR/${REPORT_NAME}_report.html"
                ;;
        esac
    done
    
    echo '</table></div>' >> "$OUTPUT_DIR/${REPORT_NAME}_report.html"
    
    # Add performance summary
    echo '<div class="section"><h2>Performance Summary</h2>' >> "$OUTPUT_DIR/${REPORT_NAME}_report.html"
    
    # CPU Summary
    CPU_AVG=$(grep "^CPU_ALL," "$NMON_FILE" | grep -v "CPU_ALL,T" | awk -F',' '
        BEGIN { sum=0; count=0; }
        NR>1 && NF>=6 { sum+=(100-$6); count++; }
        END { if(count>0) printf "%.1f", sum/count; }')
    
    if [ -n "$CPU_AVG" ]; then
        if (( $(echo "$CPU_AVG > 80" | bc -l) )); then
            echo "<div class=\"alert\">CPU Usage: ${CPU_AVG}% (HIGH)</div>" >> "$OUTPUT_DIR/${REPORT_NAME}_report.html"
        elif (( $(echo "$CPU_AVG > 60" | bc -l) )); then
            echo "<div class=\"warning\">CPU Usage: ${CPU_AVG}% (MODERATE)</div>" >> "$OUTPUT_DIR/${REPORT_NAME}_report.html"
        else
            echo "<div class=\"normal\">CPU Usage: ${CPU_AVG}% (NORMAL)</div>" >> "$OUTPUT_DIR/${REPORT_NAME}_report.html"
        fi
    fi
    
    # Memory Summary
    MEM_AVG=$(grep "^MEM," "$NMON_FILE" | grep -v "MEM,T" | awk -F',' '
        BEGIN { sum_used=0; sum_total=0; count=0; }
        NR>1 && NF>=6 { sum_used+=$4; sum_total+=$3; count++; }
        END { if(count>0) printf "%.1f", (sum_used/count)/(sum_total/count)*100; }')
    
    if [ -n "$MEM_AVG" ]; then
        if (( $(echo "$MEM_AVG > 90" | bc -l) )); then
            echo "<div class=\"alert\">Memory Usage: ${MEM_AVG}% (CRITICAL)</div>" >> "$OUTPUT_DIR/${REPORT_NAME}_report.html"
        elif (( $(echo "$MEM_AVG > 75" | bc -l) )); then
            echo "<div class=\"warning\">Memory Usage: ${MEM_AVG}% (HIGH)</div>" >> "$OUTPUT_DIR/${REPORT_NAME}_report.html"
        else
            echo "<div class=\"normal\">Memory Usage: ${MEM_AVG}% (NORMAL)</div>" >> "$OUTPUT_DIR/${REPORT_NAME}_report.html"
        fi
    fi
    
    echo '</div>' >> "$OUTPUT_DIR/${REPORT_NAME}_report.html"
    
    # Close HTML
    echo '</body></html>' >> "$OUTPUT_DIR/${REPORT_NAME}_report.html"
}

# Generate CSV reports for spreadsheet import
generate_csv_reports() {
    # CPU CSV
    echo "Timestamp,User%,System%,Wait%,Idle%" > "$OUTPUT_DIR/${REPORT_NAME}_cpu.csv"
    grep "^CPU_ALL," "$NMON_FILE" | grep -v "CPU_ALL,T" | awk -F',' '
    NR>1 && NF>=6 { 
        printf "%s,%.2f,%.2f,%.2f,%.2f\n", $2, $3, $4, $5, $6 
    }' >> "$OUTPUT_DIR/${REPORT_NAME}_cpu.csv"
    
    # Memory CSV
    echo "Timestamp,Total,Used,Free,Buffers,Cached" > "$OUTPUT_DIR/${REPORT_NAME}_memory.csv"
    grep "^MEM," "$NMON_FILE" | grep -v "MEM,T" | awk -F',' '
    NR>1 && NF>=7 { 
        printf "%s,%.2f,%.2f,%.2f,%.2f,%.2f\n", $2, $3, $4, $5, $6, $7 
    }' >> "$OUTPUT_DIR/${REPORT_NAME}_memory.csv"
    
    # Disk CSV
    echo "Timestamp,Disk,Busy%" > "$OUTPUT_DIR/${REPORT_NAME}_disk.csv"
    grep "^DISKBUSY," "$NMON_FILE" | grep -v "DISKBUSY,T" | awk -F',' '
    NR>1 {
        timestamp=$2;
        for(i=3; i<=NF; i+=2) {
            if($i != "" && $(i+1) != "") {
                printf "%s,%s,%.2f\n", timestamp, $i, $(i+1);
            }
        }
    }' >> "$OUTPUT_DIR/${REPORT_NAME}_disk.csv"
}

# Generate text summary
generate_text_summary() {
    cat > "$OUTPUT_DIR/${REPORT_NAME}_summary.txt" << EOF
NMON PERFORMANCE REPORT SUMMARY
===============================
Generated: $(date)
Source File: $NMON_FILE

SYSTEM INFORMATION:
$(grep "^AAA," "$NMON_FILE" | grep -E "host|OS|version|kernel|cpus|memory" | cut -d',' -f2-3 | sed 's/,/: /')

PERFORMANCE METRICS:
EOF

    # Add CPU summary
    echo "CPU ANALYSIS:" >> "$OUTPUT_DIR/${REPORT_NAME}_summary.txt"
    grep "^CPU_ALL," "$NMON_FILE" | grep -v "CPU_ALL,T" | awk -F',' '
    BEGIN { 
        sum_user=0; sum_sys=0; sum_wait=0; sum_idle=0; count=0;
        max_user=0; max_total=0;
    }
    NR>1 && NF>=6 {
        user=$3; sys=$4; wait=$5; idle=$6;
        total_used = 100-idle;
        sum_user+=user; sum_sys+=sys; sum_wait+=wait; sum_idle+=idle;
        if(user>max_user) max_user=user;
        if(total_used>max_total) max_total=total_used;
        count++;
    }
    END {
        if(count>0) {
            printf "  Average CPU Usage: %.1f%%\n", 100-sum_idle/count;
            printf "  Peak CPU Usage: %.1f%%\n", max_total;
            printf "  Average User: %.1f%%, System: %.1f%%, Wait: %.1f%%\n", 
                   sum_user/count, sum_sys/count, sum_wait/count;
        }
    }' >> "$OUTPUT_DIR/${REPORT_NAME}_summary.txt"
    
    # Add Memory summary
    echo "MEMORY ANALYSIS:" >> "$OUTPUT_DIR/${REPORT_NAME}_summary.txt"
    grep "^MEM," "$NMON_FILE" | grep -v "MEM,T" | awk -F',' '
    BEGIN { 
        sum_total=0; sum_used=0; count=0; max_used_pct=0;
    }
    NR>1 && NF>=6 {
        total=$3; used=$4;
        sum_total+=total; sum_used+=used;
        used_pct = (used/total)*100;
        if(used_pct>max_used_pct) max_used_pct=used_pct;
        count++;
    }
    END {
        if(count>0) {
            printf "  Average Memory Usage: %.1f%%\n", (sum_used/count)/(sum_total/count)*100;
            printf "  Peak Memory Usage: %.1f%%\n", max_used_pct;
            printf "  Average Used: %.0f MB of %.0f MB\n", sum_used/count, sum_total/count;
        }
    }' >> "$OUTPUT_DIR/${REPORT_NAME}_summary.txt"
}

# Execute all report generation functions
echo "Generating reports for: $NMON_FILE"
generate_html_report
generate_csv_reports
generate_text_summary

echo "Reports generated:"
echo "  HTML Report: $OUTPUT_DIR/${REPORT_NAME}_report.html"
echo "  CSV Files: $OUTPUT_DIR/${REPORT_NAME}_*.csv"
echo "  Text Summary: $OUTPUT_DIR/${REPORT_NAME}_summary.txt"
```

### Bước 4: Tạo script với gnuplot để tạo graphs


```
#!/bin/bash
# /usr/local/bin/nmon_graphs.sh

NMON_FILE="$1"
OUTPUT_DIR="$2"
REPORT_NAME="$(basename $NMON_FILE .nmon)"

# Generate CPU graph
generate_cpu_graph() {
    # Prepare data
    grep "^CPU_ALL," "$NMON_FILE" | grep -v "CPU_ALL,T" | awk -F',' '
    NR>1 && NF>=6 { 
        gsub(/T/, "", $2);
        printf "%d %.2f %.2f %.2f %.2f\n", NR-1, $3, $4, $5, $6 
    }' > "$OUTPUT_DIR/${REPORT_NAME}_cpu_data.dat"
    
    # Generate gnuplot script
    cat > "$OUTPUT_DIR/cpu_plot.gp" << 'EOF'
set terminal png size 800,600
set output "OUTPUT_DIR/REPORT_NAME_cpu_usage.png"
set title "CPU Usage Over Time"
set xlabel "Time Interval"
set ylabel "CPU Percentage (%)"
set grid
set key outside
plot "OUTPUT_DIR/REPORT_NAME_cpu_data.dat" using 1:2 with lines title "User" lw 2, \
     "" using 1:3 with lines title "System" lw 2, \
     "" using 1:4 with lines title "Wait" lw 2, \
     "" using 1:5 with lines title "Idle" lw 2
EOF
    
    # Replace placeholders
    sed -i "s|OUTPUT_DIR|$OUTPUT_DIR|g" "$OUTPUT_DIR/cpu_plot.gp"
    sed -i "s|REPORT_NAME|$REPORT_NAME|g" "$OUTPUT_DIR/cpu_plot.gp"
    
    # Generate graph
    gnuplot "$OUTPUT_DIR/cpu_plot.gp"
}

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