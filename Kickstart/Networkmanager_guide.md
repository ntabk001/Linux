# Hướng dẫn quản lý mạng dùng NetworkManager `trên CentOS 8`

## 1. Giới thiệu về NetworkManager

NetworkManager là một công cụ mạnh mẽ hỗ trợ quản lý và thiết lập mạng trên `CentOS 8` với các tính năng:

- Quản lý mạng bằng dòng lệnh và giao diện đồ họa

- Cung cấp API thông qua D-Bus

- Hỗ trợ cấu hình linh hoạt

- Tích hợp với Cockpit web console

- Hỗ trợ các lệnh tùy chỉnh

## 2. Cài đặt và kiểm tra NetworkManager

### Kiểm tra phiên bản NetworkManager:

```
[root@localhost ~]# NetworkManager -V
1.14.0-14.el8
```

### Cài đặt NetworkManager (nếu chưa có):

```
[root@localhost ~]# dnf install NetworkManager
```

### Kiểm tra file cấu hình:

```
[root@localhost ~]# ls -l /etc/NetworkManager/NetworkManager.conf
-rw-r--r--. 1 root root 2145 May 11 2019 /etc/NetworkManager/NetworkManager.conf

[root@localhost ~]# ls -l /etc/NetworkManager/
total 4
drwxr-xr-x. 2 root root 6 May 11 2019 conf.d
drwxr-xr-x. 5 root root 93 Nov 6 03:14 dispatcher.d
drwxr-xr-x. 2 root root 6 May 11 2019 dnsmasq.d
drwxr-xr-x. 2 root root 6 May 11 2019 dnsmasq-shared.d
-rw-r--r--. 1 root root 2145 May 11 2019 NetworkManager.conf
drwxr-xr-x. 2 root root 6 May 11 2019 system-connections
```

## 3. Quản lý NetworkManager bằng systemctl

### Kiểm tra trạng thái hoạt động:

```
[root@localhost ~]# systemctl is-active NetworkManager
active

[root@localhost ~]# systemctl is-enabled NetworkManager
enabled
```

### Kiểm tra trạng thái chi tiết:

```
[root@localhost ~]# systemctl status NetworkManager
● NetworkManager.service - Network Manager
   Loaded: loaded (/usr/lib/systemd/system/NetworkManager.service; enabled; vendor preset: enabled)
   Active: active (running) since Sun 2019-12-08 10:28:05 EST; 1min 57s ago
   Docs: man:NetworkManager(8)
   Main PID: 958 (NetworkManager)
   Tasks: 3 (limit: 17812)
   Memory: 9.5M
   CGroup: /system.slice/NetworkManager.service
           └─958 /usr/sbin/NetworkManager --no-daemon
```

### Các lệnh quản lý dịch vụ:

```
# Khởi động NetworkManager
systemctl start NetworkManager

# Dừng NetworkManager
systemctl stop NetworkManager

# Khởi động lại NetworkManager
systemctl restart NetworkManager

# Tải lại cấu hình
systemctl reload NetworkManager
```

## 4. Sử dụng nmcli để quản lý mạng

### Liệt kê các thiết bị mạng:

```
[root@localhost ~]# nmcli device
DEVICE   TYPE      STATE      CONNECTION
ens192   ethernet  connected  ens192
lo       loopback  unmanaged  --

# Hoặc sử dụng:
[root@localhost ~]# nmcli device status
```

### Xem các kết nối hoạt động:

```
[root@localhost ~]# nmcli connection show -a
NAME    UUID                                  TYPE      DEVICE
ens192  c94d64af-3f07-4480-a65f-2093c869608a  ethernet  ens192
```

### Kiểm tra trạng thái NetworkManager:

```
[root@localhost ~]# nmcli -t -f RUNNING general
running

[root@localhost ~]# nmcli general
STATE      CONNECTIVITY  WIFI-HW  WIFI     WWAN-HW  WWAN
connected  full          enabled  enabled  enabled  enabled
```

### Liệt kê tất cả các kết nối:

```
[root@localhost ~]# nmcli con show
NAME    UUID                                  TYPE      DEVICE
ens192  7ce724d6-0326-3f7a-9b6a-630504393a99  ethernet  ens192
```

### Xem chi tiết cấu hình interface:

```
[root@localhost ~]# nmcli con show ens192
connection.id:                     ens192
connection.uuid:                   7ce724d6-0326-3f7a-9b6a-630504393a99
connection.stable-id:              --
connection.type:                   802-3-ethernet
connection.interface-name:         ens192
connection.autoconnect:            yes
connection.autoconnect-priority:   0
connection.autoconnect-retries:    -1 (default)
...
```

## 5. Quản lý hostname

### Kiểm tra hostname hiện tại:

```
[root@localhost ~]# nmcli general hostname
blogd-net-lab01
```

### Thay đổi hostname:

```
[root@localhost ~]# nmcli general hostname blogd-net-lab02
[root@localhost ~]# reboot

# Sau khi reboot:
[root@localhost ~]# nmcli general hostname
blogd-net-lab02
```

## 6. Chỉnh sửa cấu hình mạng

### Chỉnh sửa địa chỉ IP bằng editor tương tác:

```
[root@localhost ~]# nmcli con edit ens192
===| nmcli interactive connection editor |===
Editing existing '802-3-ethernet' connection: 'ens192'

nmcli> print ipv4.addresses
ipv4.addresses: 192.168.0.2/24

nmcli> remove ipv4.address "192.168.0.2/24"
nmcli> set ipv4.address 192.168.0.3/24
Do you also want to set 'ipv4.method' to 'manual'? [yes]: yes

nmcli> verify
Verify connection: OK

nmcli> save
Connection 'ens192' successfully updated.

nmcli> quit
```

### Kiểm tra thay đổi:

```
[root@localhost ~]# egrep IPADDR /etc/sysconfig/network-scripts/ifcfg-ens192
IPADDR=192.168.0.3
```

## 7. Bảng so sánh lệnh nmcli và file cấu hình

### IPv4 Configuration:

| Lệnh nmcli | File ifcfg | Chức năng |
|------------|------------|-----------|
| `ipv4.method manual` | `BOOTPROTO=none` | Địa chỉ IPv4 cấu hình tĩnh |
| `ipv4.method auto` | `BOOTPROTO=dhcp` | Địa chỉ IPv4 được cấp tự động |
| `ipv4.address 192.168.0.1/24` | `IPADDR=192.168.0.1 PREFIX=24` | Đặt địa chỉ IPv4 tĩnh |
| `ipv4.gateway 192.168.0.1` | `GATEWAY=192.168.0.1` | Đặt IPv4 Gateway |
| `ipv4.dns 8.8.8.8` | `DNS1=8.8.8.8` | Cấu hình DNS server |
| `connection.autoconnect yes` | `ONBOOT=yes` | Tự động kết nối khi khởi động |

### IPv6 Configuration:

| Lệnh nmcli | File ifcfg | Chức năng |
|------------|------------|-----------|
| `ipv6.method manual` | `BOOTPROTO=none` | Địa chỉ IPv6 cấu hình tĩnh |
| `ipv6.method auto` | `IPV6_AUTOCONF=yes` | Cấu hình mạng bằng SLAAC |
| `ipv6.method dhcp` | `IPV6_AUTOCONF=no DHCPV6C=yes` | Cấu hình mạng bằng DHCPv6 |

## 8. Tạo kết nối mới

### Tạo kết nối Ethernet với IP tĩnh:

```
[root@localhost ~]# nmcli con add con-name ens37 type ethernet ifname ens37 \
    ipv4.method manual ipv4.address 10.10.10.5/24 ipv4.gateway 10.10.10.1
Connection 'ens37' successfully added.
```

### Kiểm tra file cấu hình được tạo:

```
[root@localhost ~]# cat /etc/sysconfig/network-scripts/ifcfg-ens37
TYPE=Ethernet
PROXY_METHOD=none
BROWSER_ONLY=no
BOOTPROTO=none
IPADDR=10.10.10.5
PREFIX=24
GATEWAY=10.10.10.1
DEFROUTE=yes
IPV4_FAILURE_FATAL=no
IPV6INIT=yes
IPV6_AUTOCONF=yes
IPV6_DEFROUTE=yes
IPV6_FAILURE_FATAL=no
IPV6_ADDR_GEN_MODE=stable-privacy
NAME=ens37
UUID=ce30feec-0926-4c31-8579-3b8d3906fba4
DEVICE=ens37
ONBOOT=yes
```

### Tạo kết nối Ethernet với DHCP:

```
[root@localhost ~]# nmcli con add con-name ens39 type ethernet ifname ens39 ipv4.method auto
Connection 'ens39' successfully added.

[root@localhost ~]# egrep BOOTPROTO /etc/sysconfig/network-scripts/ifcfg-ens39
BOOTPROTO=dhcp
```

## 9. Chỉnh sửa kết nối hiện có

### Chuyển từ DHCP sang IP tĩnh:

```
# Kiểm tra trạng thái hiện tại
[root@localhost ~]# egrep BOOTPROTO /etc/sysconfig/network-scripts/ifcfg-ens39
BOOTPROTO=dhcp

# Chuyển sang IP tĩnh
[root@localhost ~]# nmcli con mod ens39 ipv4.method manual \
    ipv4.address 10.10.20.5/24 ipv4.gateway 10.10.20.1

# Kiểm tra kết quả
[root@localhost ~]# egrep 'BOOTPROTO|IPADDR|PREFIX|GATEWAY' /etc/sysconfig/network-scripts/ifcfg-ens39
BOOTPROTO=none
IPADDR=10.10.20.5
PREFIX=24
GATEWAY=10.10.20.1
```

### Chuyển từ IP tĩnh sang DHCP:

```
[root@localhost ~]# nmcli con mod ens37 ipv4.method auto
```

### Thay đổi ONBOOT:

```
# Kiểm tra trạng thái hiện tại
[root@localhost ~]# egrep 'ONBOOT' /etc/sysconfig/network-scripts/ifcfg-ens39
ONBOOT=yes

# Vô hiệu hóa ONBOOT
[root@localhost ~]# nmcli con mod ens39 connection.autoconnect no

# Kiểm tra kết quả
[root@localhost ~]# egrep 'ONBOOT' /etc/sysconfig/network-scripts/ifcfg-ens39
ONBOOT=no
```

### Thay đổi DEFROUTE:

```
# Kiểm tra trạng thái hiện tại
[root@localhost ~]# egrep '^DEFROUTE' /etc/sysconfig/network-scripts/ifcfg-ens39
DEFROUTE=yes

# Vô hiệu hóa default route
[root@localhost ~]# nmcli con mod ens39 ipv4.never-default yes

# Kiểm tra kết quả
[root@localhost ~]# egrep '^DEFROUTE' /etc/sysconfig/network-scripts/ifcfg-ens39
DEFROUTE=no
```

### Vô hiệu hóa IPv6:

```
# Kiểm tra trạng thái hiện tại
[root@localhost ~]# egrep 'IPV6INIT' /etc/sysconfig/network-scripts/ifcfg-ens39
IPV6INIT=yes

# Tắt IPv6
[root@localhost ~]# nmcli con mod ens39 ipv6.method ignore

# Kiểm tra kết quả
[root@localhost ~]# egrep 'IPV6INIT' /etc/sysconfig/network-scripts/ifcfg-ens39
IPV6INIT=no
```

## 10. Quản lý DNS

### Thêm DNS server đơn:

```
[root@localhost ~]# nmcli con mod ens37 ipv4.dns 8.8.8.8

[root@localhost ~]# egrep DNS /etc/sysconfig/network-scripts/ifcfg-ens37
DNS1=8.8.8.8
```

### Thêm DNS server bổ sung:

```
[root@localhost ~]# nmcli con mod ens37 +ipv4.dns 8.4.4.4

[root@localhost ~]# egrep DNS /etc/sysconfig/network-scripts/ifcfg-ens37
DNS1=8.8.8.8
DNS2=8.4.4.4
```

### Xóa DNS server:

```
[root@localhost ~]# nmcli con mod ens37 -ipv4.dns 8.4.4.4
[root@localhost ~]# nmcli con mod ens37 -ipv4.dns 8.8.8.8
```

## 11. Kích hoạt/Vô hiệu hóa kết nối

### Kiểm tra các kết nối hoạt động:

```
[root@localhost ~]# nmcli con show --active
NAME   UUID                                  TYPE      DEVICE
ens33  3067faf9-4b17-4028-b4db-1e6c721cfb20  ethernet  ens33
```

### Kích hoạt kết nối:

```
[root@localhost ~]# nmcli con up ens37
Connection successfully activated (D-Bus active path: /org/freedesktop/NetworkManager/ActiveConnection/3)
```

### Vô hiệu hóa kết nối:

```
[root@localhost ~]# nmcli con down ens37
Connection 'ens37' successfully deactivated
```

## 12. Cài đặt network-scripts (tuỳ chọn)

CentOS 8 mặc định không cài đặt network-scripts cũ. Nếu cần sử dụng:

```
[root@localhost ~]# yum install network-scripts
```

Sau khi cài đặt, bạn có thể sử dụng các lệnh `ifup` và `ifdown` truyền thống.

## 13. Lưu ý quan trọng

1. **Luôn sử dụng nmcli**: Đây là công cụ chính thức và được khuyến nghị cho `CentOS 8`

2. **Restart NetworkManager**: Sau khi thay đổi cấu hình, hãy restart NetworkManager để áp dụng thay đổi

3. **Backup cấu hình**: Luôn sao lưu file cấu hình trước khi thay đổi

4. **Kiểm tra kết nối**: Sau mỗi thay đổi, hãy kiểm tra kết nối mạng để đảm bảo hoạt động bình thường

## 14. Các lệnh hữu ích khác

```
# Reload tất cả các kết nối
nmcli connection reload

# Xem thông tin chi tiết thiết bị
nmcli device show ens192

# Xem log NetworkManager
journalctl -u NetworkManager

# Kiểm tra routing table
ip route show

# Kiểm tra địa chỉ IP
ip addr show
```

NetworkManager là công cụ mạnh mẽ giúp quản lý mạng hiệu quả trên CentOS 8. Việc nắm vững các lệnh nmcli sẽ giúp bạn cấu hình và quản lý mạng một cách linh hoạt và chính xác.