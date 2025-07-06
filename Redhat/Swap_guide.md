# Swap Memory: Khái niệm và Cách sử dụng

## 1. Swap là gì?

**Swap** (Virtual Memory) là một phần không gian lưu trữ trên ổ cứng được sử dụng như RAM ảo khi hệ thống hết RAM vật lý. Khi RAM đầy, hệ thống sẽ di chuyển các trang memory ít được sử dụng sang swap space để giải phóng RAM cho các ứng dụng đang hoạt động.

### Tại sao cần Swap?

1. **Mở rộng bộ nhớ**: Tăng tổng dung lượng memory khả dụng

2. **Hibernation**: Cần thiết để hệ thống có thể hibernate

3. **Ổn định hệ thống**: Tránh crash khi hết RAM

4. **Tối ưu hiệu suất**: Cho phép hệ thống quản lý memory hiệu quả hơn

### Nhược điểm của Swap:

1. **Chậm hơn RAM**: Tốc độ truy cập chậm hơn RAM rất nhiều

2. **Wear SSD**: Có thể làm giảm tuổi thọ SSD nếu swap nhiều

3. **Hiệu suất giảm**: Nếu hệ thống swap quá nhiều (thrashing)

## 2. Kiểm tra Swap hiện tại

### Xem thông tin swap:

```
# Xem tổng quan swap
free -h

# Xem chi tiết swap
swapon --show

# Xem từ /proc/swaps
cat /proc/swaps

# Xem thông tin memory và swap
cat /proc/meminfo | grep -i swap
```

### Ví dụ output:

```
$ free -h
              total        used        free      shared  buff/cache   available
Mem:          7.7Gi       2.1Gi       3.2Gi       234Mi       2.4Gi       5.1Gi
Swap:         2.0Gi          0B       2.0Gi

$ swapon --show
NAME      TYPE      SIZE USED PRIO
/swapfile file        2G   0B   -2
```

## 3. Cách tạo Swap

### Phương pháp 1: Tạo Swap File (Khuyến nghị)

#### Bước 1: Tạo swap file

```
# Tạo file swap 2GB
sudo fallocate -l 2G /swapfile

# Hoặc sử dụng dd (chậm hơn)
sudo dd if=/dev/zero of=/swapfile bs=1024 count=2097152
```

#### Bước 2: Phân quyền bảo mật

```
# Chỉ cho root đọc/ghi
sudo chmod 600 /swapfile

# Kiểm tra permissions
ls -lh /swapfile
```

#### Bước 3: Thiết lập swap

```
# Tạo swap area
sudo mkswap /swapfile

# Kích hoạt swap
sudo swapon /swapfile

# Kiểm tra
sudo swapon --show
```

#### Bước 4: Tự động mount khi boot

```
# Backup fstab
sudo cp /etc/fstab /etc/fstab.bak

# Thêm vào fstab
echo '/swapfile none swap sw 0 0' | sudo tee -a /etc/fstab

# Kiểm tra fstab
cat /etc/fstab | grep swap
```

### Phương pháp 2: Tạo Swap Partition

#### Bước 1: Tạo partition

```
# Sử dụng fdisk để tạo partition
sudo fdisk /dev/sdb

# Trong fdisk:
# n - tạo partition mới
# p - primary partition
# 1 - partition number
# Enter - first sector (default)
# +2G - size 2GB
# t - change partition type
# 82 - Linux swap
# w - write changes
```

#### Bước 2: Tạo swap filesystem

```
# Tạo swap trên partition
sudo mkswap /dev/sdb1

# Kích hoạt swap
sudo swapon /dev/sdb1

# Kiểm tra
sudo swapon --show
```

#### Bước 3: Tự động mount

```
# Thêm vào fstab
echo '/dev/sdb1 none swap sw 0 0' | sudo tee -a /etc/fstab
```

### Phương pháp 3: Sử dụng LVM

#### Tạo logical volume cho swap:

```
# Tạo logical volume
sudo lvcreate -L 2G -n swap_lv volume_group

# Tạo swap filesystem
sudo mkswap /dev/volume_group/swap_lv

# Kích hoạt swap
sudo swapon /dev/volume_group/swap_lv

# Thêm vào fstab
echo '/dev/volume_group/swap_lv none swap sw 0 0' | sudo tee -a /etc/fstab
```

## 4. Quản lý Swap

### Tắt swap:

```
# Tắt swap file cụ thể
sudo swapoff /swapfile

# Tắt tất cả swap
sudo swapoff -a

# Bật lại tất cả swap
sudo swapon -a
```

### Thay đổi kích thước swap file:

```
# Tắt swap
sudo swapoff /swapfile

# Thay đổi kích thước (4GB)
sudo fallocate -l 4G /swapfile

# Hoặc tạo file mới
sudo dd if=/dev/zero of=/swapfile bs=1024 count=4194304

# Tạo lại swap
sudo mkswap /swapfile

# Kích hoạt lại
sudo swapon /swapfile
```

### Xóa swap:

```
# Tắt swap
sudo swapoff /swapfile

# Xóa dòng trong fstab
sudo sed -i '/swapfile/d' /etc/fstab

# Xóa file
sudo rm /swapfile
```

## 5. Cấu hình Swappiness

### Swappiness là gì?

`swappiness` là một thuộc tính xác định tần suất hệ thống sẽ sử dụng phân vùng swap.

`swappiness` có thể có giá trị từ `0` đến `100` chúng ta có thể chọn giá trị `swappiness` sau cho phù hợp nhất đối với chúng ta ý nghĩa các số từ `0 - 10` qua ba ví dụ sau:

`swappiness = 0`: swap chỉ được dùng khi RAM được sử dụng hết.

`swappiness = 50`: swap được sử dụng khi RAM còn 50%.

`swappiness = 100`: swap được uư tiên sử dụng hơn RAM.

Giá trị swappiness mặc định trên CentOS `7` là `30`. Chúng ta có thể kiểm tra giá trị trao đổi hiện tại bằng cách nhập lệnh sau:

```
cat /proc/sys/vm/swappiness
30
```

Chúng ta có thể đặt `swappiness` thành một giá trị khác bằng cách sử dụng lệnh `sysctl`. Để đặt `swappiness` thành `20`, thực hiện như bên dưới:

```
sysctl -w vm.swappiness=20
vm.swappiness = 20
```

### Xem swappiness hiện tại:

```
# Xem giá trị swappiness
cat /proc/sys/vm/swappiness

# Hoặc
sysctl vm.swappiness
```

### Thay đổi swappiness tạm thời:

```
# Đặt swappiness = 10
sudo sysctl vm.swappiness=10

# Hoặc
echo 10 | sudo tee /proc/sys/vm/swappiness
```

### Thay đổi swappiness vĩnh viễn:

```
# Thêm vào /etc/sysctl.conf
echo 'vm.swappiness=10' | sudo tee -a /etc/sysctl.conf

# Hoặc tạo file riêng
echo 'vm.swappiness=10' | sudo tee /etc/sysctl.d/99-swappiness.conf

# Reload sysctl
sudo sysctl -p
```

### Khuyến nghị Swappiness:

| Loại hệ thống | Swappiness | Lý do |
|---------------|------------|-------|
| Desktop/Laptop | 10-20 | Ưu tiên responsive, ít swap |
| Server Database | 1-5 | Tránh swap DB pages |
| Web Server | 10-30 | Cân bằng giữa cache và swap |
| File Server | 20-40 | Có thể swap một số processes |
| Workstation | 5-15 | Ưu tiên hiệu suất ứng dụng |

## 6. Tối ưu hóa Swap

### Sử dụng nhiều swap devices:

```
# Tạo nhiều swap với priority khác nhau
sudo swapon -p 5 /swapfile1    # Priority cao
sudo swapon -p 3 /swapfile2    # Priority thấp

# Trong fstab:
/swapfile1 none swap sw,pri=5 0 0
/swapfile2 none swap sw,pri=3 0 0
```

### Swap trên SSD vs HDD:

```
# SSD: Sử dụng discard option
/swapfile none swap sw,discard 0 0

# HDD: Không cần discard
/swapfile none swap sw 0 0
```

### Monitoring swap usage:

```
# Script monitor swap
#!/bin/bash
while true; do
    clear
    echo "=== Swap Usage Monitor ==="
    echo "Date: $(date)"
    echo
    free -h
    echo
    echo "=== Top Swap Users ==="
    for dir in /proc/*/; do
        pid=$(basename $dir)
        if [[ $pid =~ ^[0-9]+$ ]]; then
            if [[ -r $dir/smaps ]]; then
                swap=$(grep "Swap:" $dir/smaps 2>/dev/null | awk '{sum+=$2} END {print sum}')
                if [[ $swap -gt 0 ]]; then
                    comm=$(cat $dir/comm 2>/dev/null)
                    echo "PID: $pid, Swap: ${swap}KB, Command: $comm"
                fi
            fi
        fi
    done | sort -k4 -nr | head -10
    sleep 5
done
```

## 7. Troubleshooting Swap

### Swap không hoạt động:

```
# Kiểm tra swap có enable không
cat /proc/swaps

# Kiểm tra fstab syntax
sudo mount -a

# Kiểm tra permissions
ls -l /swapfile

# Kiểm tra filesystem
sudo file /swapfile
```

### Swap quá chậm:

```
# Kiểm tra I/O wait
iostat -x 1

# Kiểm tra swap usage
sar -S 1

# Điều chỉnh swappiness
sudo sysctl vm.swappiness=5
```

### Swap đầy:

```
# Xem process sử dụng swap nhiều
sudo find /proc -name "smaps" -exec grep -l "Swap:" {} \; | \
while read file; do
    pid=$(echo $file | sed 's/.*\/proc\/\([0-9]*\)\/.*/\1/')
    swap=$(grep "Swap:" $file | awk '{sum+=$2} END {print sum}')
    if [[ $swap -gt 1000 ]]; then
        echo "PID: $pid, Swap: ${swap}KB"
    fi
done
```

## 8. Best Practices

### 1. Kích thước swap phù hợp:

```
# RAM <= 2GB: Swap = 2 * RAM
# RAM 2-8GB: Swap = RAM
# RAM > 8GB: Swap = 0.5 * RAM (hoặc 4GB minimum)

# Ví dụ script tính toán:
#!/bin/bash
RAM_GB=$(free -g | awk 'NR==2{print $2}')
if [[ $RAM_GB -le 2 ]]; then
    SWAP_GB=$((RAM_GB * 2))
elif [[ $RAM_GB -le 8 ]]; then
    SWAP_GB=$RAM_GB
else
    SWAP_GB=$((RAM_GB / 2))
    if [[ $SWAP_GB -lt 4 ]]; then
        SWAP_GB=4
    fi
fi
echo "Recommended swap size: ${SWAP_GB}GB"
```

### 2. Security considerations:

```
# Encrypt swap (Ubuntu/Debian)
sudo apt install cryptsetup

# Thêm vào /etc/crypttab
echo "swap /dev/sdb1 /dev/urandom swap,cipher=aes-xts-plain64,size=256" | sudo tee -a /etc/crypttab

# Thêm vào fstab
echo "/dev/mapper/swap none swap sw 0 0" | sudo tee -a /etc/fstab
```

### 3. Monitoring script:

```
#!/bin/bash
# /usr/local/bin/swap-monitor.sh

SWAP_THRESHOLD=80
SWAP_USAGE=$(free | awk 'NR==3{printf "%.0f", $3/$2*100}')

if [[ $SWAP_USAGE -gt $SWAP_THRESHOLD ]]; then
    echo "WARNING: Swap usage is ${SWAP_USAGE}%" | \
    mail -s "High Swap Usage Alert" admin@example.com
fi
```

### 4. Automatic swap management:

```
#!/bin/bash
# Script tự động tăng swap khi cần thiết

SWAP_FILE="/swapfile"
CURRENT_SWAP=$(free -g | awk 'NR==3{print $2}')
FREE_DISK=$(df / | awk 'NR==2{print $4}')

if [[ $CURRENT_SWAP -eq 0 ]] && [[ $FREE_DISK -gt 2000000 ]]; then
    echo "Creating emergency swap..."
    sudo fallocate -l 2G $SWAP_FILE
    sudo chmod 600 $SWAP_FILE
    sudo mkswap $SWAP_FILE
    sudo swapon $SWAP_FILE
    echo "Emergency swap created and activated"
fi
```

## Tóm tắt

Swap là công cụ quan trọng để quản lý memory trong Linux. Việc cấu hình đúng swap và swappiness có thể cải thiện đáng kể hiệu suất hệ thống. Hãy nhớ:

1. **Swap file** linh hoạt hơn swap partition

2. **Swappiness=10** thường tốt cho desktop

3. **Swappiness=1-5** phù hợp cho servers

4. **Monitor swap usage** thường xuyên

5. **Backup** trước khi thay đổi cấu hình