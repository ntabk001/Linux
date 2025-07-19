# Hướng dẫn chi tiết lệnh ps theo dõi tiến trình Linux

## 1. Giới thiệu lệnh ps

Lệnh `ps` (Process Status) là công cụ mạnh mẽ để xem và theo dõi các tiến trình đang chạy trên hệ thống Linux. Nó cung cấp thông tin chi tiết về:

- Process ID (PID)

- Parent Process ID (PPID)

- Mức sử dụng CPU và bộ nhớ

- Trạng thái tiến trình

- Tên lệnh và đường dẫn

**Cú pháp cơ bản**: `ps [options]`

## 2. Các ví dụ cơ bản với lệnh ps

### Ví dụ 1: Hiển thị tiến trình trong Shell hiện tại

```
[root@localhost ~]# ps
PID TTY     TIME CMD
1701 pts/0  00:00:00 bash
1714 pts/0  00:00:00 ps
```

**Giải thích các cột:**

- **PID**: Process ID - định danh duy nhất của tiến trình

- **TTY**: Terminal Type - loại thiết bị đầu cuối

- **TIME**: Thời gian CPU tích lũy (phút:giây)

- **CMD**: Command - tên lệnh khởi chạy tiến trình

### Ví dụ 2: Hiển thị tất cả tiến trình trên hệ thống

```
# Cách 1: Sử dụng tùy chọn -A
[root@localhost ~]# ps -A
PID TTY     TIME CMD
1   ?       00:00:02 systemd
2   ?       00:00:00 kthreadd
3   ?       00:00:00 rcu_gp
4   ?       00:00:00 rcu_par_gp
...

# Cách 2: Sử dụng tùy chọn -e (tương đương -A)
[root@localhost ~]# ps -e
PID TTY     TIME CMD
1   ?       00:00:02 systemd
2   ?       00:00:00 kthreadd
...
```

### Ví dụ 3: Hiển thị định dạng BSD với thông tin chi tiết

```
# Sử dụng tùy chọn au
[root@localhost ~]# ps au
USER PID %CPU %MEM    VSZ   RSS TTY STAT START   TIME COMMAND
root 1030 0.0  0.0 110112  1836 tty1 Ss+  18:13   0:00 /sbin/agetty --noclear tty1 linux
root 1701 0.0  0.1 115448  3476 pts/0 Ss  18:30   0:00 -bash
root 1731 0.0  0.1 155368  3896 pts/0 R+  18:32   0:00 ps au

# Sử dụng tùy chọn axu (hiển thị tất cả tiến trình)
[root@localhost ~]# ps axu
USER PID %CPU %MEM    VSZ   RSS TTY STAT START   TIME COMMAND
root   1  0.1  0.3 128020  8036 ?   Ss   18:12   0:02 /usr/lib/systemd/systemd
root   2  0.0  0.0      0     0 ?   S    18:12   0:00 [kthreadd]
...
```

**Giải thích các cột bổ sung:**

- **USER**: Người dùng sở hữu tiến trình

- **%CPU**: Phần trăm CPU sử dụng

- **%MEM**: Phần trăm bộ nhớ sử dụng

- **VSZ**: Virtual Size - kích thước bộ nhớ ảo (KB)

- **RSS**: Resident Set Size - bộ nhớ vật lý đang sử dụng (KB)

- **STAT**: Trạng thái tiến trình

## 3. Lọc tiến trình theo tiêu chí cụ thể

### Ví dụ 4: Hiển thị định dạng đầy đủ

```
# Sử dụng -eF để hiển thị thông tin mở rộng
[root@localhost ~]# ps -eF
UID  PID PPID  C    SZ   RSS PSR STIME TTY      TIME CMD
root   1    0  0 32005  8036   0 18:12 ?     00:00:02 /usr/lib/systemd/systemd
root   2    0  0     0     0   0 18:12 ?     00:00:00 [kthreadd]
...

# Sử dụng -ef để hiển thị định dạng tiêu chuẩn
[root@localhost ~]# ps -ef
UID  PID PPID  C STIME TTY      TIME CMD
root   1    0  0 18:12 ?     00:00:02 /usr/lib/systemd/systemd
root   2    0  0 18:12 ?     00:00:00 [kthreadd]
...
```

### Ví dụ 5: Lọc theo người dùng

```
# Hiển thị tiến trình của người dùng root
[root@localhost ~]# ps -fU root
UID  PID PPID  C STIME TTY      TIME CMD
root   1    0  0 18:12 ?     00:00:02 /usr/lib/systemd/systemd
root   2    0  0 18:12 ?     00:00:00 [kthreadd]
...

# Hiển thị tiến trình với Real và Effective User ID
[root@localhost ~]# ps -U root -u root
PID TTY      TIME CMD
  1 ?     00:00:02 systemd
  2 ?     00:00:00 kthreadd
...
```

### Ví dụ 6: Lọc theo nhóm (Group)

```
# Theo tên nhóm
[root@localhost ~]# ps -fG apache
UID  PID PPID  C STIME TTY      TIME CMD
apache 2070 2069 0 18:50 ?     00:00:00 /usr/sbin/httpd -DFOREGROUND
apache 2071 2069 0 18:50 ?     00:00:00 /usr/sbin/httpd -DFOREGROUND
...

# Theo Group ID
[root@localhost ~]# ps -fG 48
UID  PID PPID  C STIME TTY      TIME CMD
apache 2070 2069 0 18:50 ?     00:00:00 /usr/sbin/httpd -DFOREGROUND
...
```

### Ví dụ 7: Lọc theo PID

```
# Hiển thị tiến trình theo PID cụ thể
[root@localhost ~]# ps -fp 1818
UID  PID PPID  C STIME TTY      TIME CMD
root 1818 1701 0 18:41 pts/0 00:00:00 ping 8.8.8.8

# Hiển thị nhiều PID cùng lúc
[root@localhost ~]# ps -fp 1821,2070,1853
UID  PID PPID  C STIME TTY      TIME CMD
root 1821 1701 0 18:42 pts/0 00:00:00 ping 8.8.8.8
root 1853 1701 0 18:46 pts/0 00:00:00 ping 8.8.8.8
apache 2070 2069 0 18:50 ?   00:00:00 /usr/sbin/httpd -DFOREGROUND
```

### Ví dụ 8: Lọc theo TTY

```
# Hiển thị tiến trình theo terminal cụ thể
[root@localhost ~]# ps -t pts/0
PID TTY      TIME CMD
1701 pts/0   00:00:00 bash
1818 pts/0   00:00:00 ping
1821 pts/0   00:00:00 ping
2259 pts/0   00:00:00 ps

[root@localhost ~]# ps -t tty1
PID TTY      TIME CMD
1030 tty1    00:00:00 agetty
```

## 4. Hiển thị dạng cây tiến trình

### Ví dụ 9: Cây tiến trình tổng quan

```
[root@localhost ~]# ps -e --forest
PID TTY      TIME CMD
  2 ?     00:00:00 kthreadd
  3 ?     00:00:00  \_ rcu_gp
  4 ?     00:00:00  \_ rcu_par_gp
  6 ?     00:00:00  \_ kworker/0:0H-kb
...
```

### Ví dụ 10: Cây tiến trình cho lệnh cụ thể

```
# Hiển thị cây cho tiến trình sshd
[root@localhost ~]# ps -f --forest -C sshd
UID  PID PPID  C STIME TTY      TIME CMD
root 1452    1 0 18:13 ?     00:00:00 /usr/sbin/sshd -D
root 1698 1452 0 18:30 ?     00:00:00  \_ sshd: root@pts/0

# Kết hợp với grep để lọc
[root@localhost ~]# ps -ef --forest | grep -v grep | grep sshd
root 1452    1 0 18:13 ?     00:00:00 /usr/sbin/sshd -D
root 1698 1452 0 18:30 ?     00:00:00  \_ sshd: root@pts/0
```

## 5. Hiển thị threads (luồng)

### Ví dụ 11: Hiển thị tất cả threads

```
[root@localhost ~]# ps -fL -C sshd
UID  PID PPID  LWP  C NLWP STIME TTY      TIME CMD
root 1452    1 1452 0    1 18:13 ?     00:00:00 /usr/sbin/sshd -D
root 1698 1452 1698 0    1 18:30 ?     00:00:00 sshd: root@pts/0
```

**Giải thích:**

- **LWP**: Light Weight Process (Thread ID)

- **NLWP**: Number of Light Weight Processes (số lượng threads)

## 6. Tùy chỉnh định dạng đầu ra

### Ví dụ 12: Hiển thị các cột tùy chỉnh

```
# Hiển thị PID, PPID, user, TTY, command
[root@localhost ~]# ps -eo pid,ppid,user,tty,cmd
PID PPID USER     TT   CMD
  1    0 root     ?    /usr/lib/systemd/systemd --switched-root
  2    0 root     ?    [kthreadd]
  3    2 root     ?    [rcu_gp]
...

# Hiển thị thông tin chi tiết với thời gian
[root@localhost ~]# ps -p 2195 -o pid,ppid,fgroup,ni,lstart,etime
PID PPID FGROUP   NI STARTED                     ELAPSED
2195    1 root      0 Fri Jul 26 19:01:02 2019    19:26
```

### Ví dụ 13: Tìm tên tiến trình bằng PID

```
# Chỉ hiển thị tên lệnh
[root@localhost ~]# ps -p 1818 -o comm=
ping

# Tìm tất cả PID của một tiến trình
[root@localhost ~]# ps -C ping -o pid=
1818
1821
1826
1853
```

### Ví dụ 14: Hiển thị thời gian chạy

```
# Hiển thị thời gian elapsed của tiến trình
[root@localhost ~]# ps -eo comm,etime,user | grep sshd
sshd         01:09:54 root
sshd            52:51 root
```

## 7. Sắp xếp và lọc tiến trình theo tài nguyên

### Ví dụ 15: Top tiến trình sử dụng bộ nhớ cao nhất

```
[root@localhost ~]# ps -eo pid,ppid,cmd,%mem,%cpu --sort=-%mem | head -10
PID PPID CMD                          %MEM %CPU
1033    1 /usr/bin/python -Es /usr/sb   1.9  0.0
1453    1 /usr/bin/python2 -Es /usr/s   1.0  0.0
991     1 /usr/lib/polkit-1/polkitd     0.8  0.0
1069    1 /usr/sbin/NetworkManager      0.6  0.0
1014    1 /usr/bin/vmtoolsd             0.5  0.0
1013    1 /usr/bin/VGAuthService        0.5  0.0
1698 1452 sshd: root@pts/0              0.4  0.0
1      0 /usr/lib/systemd/systemd      0.4  0.0
1452    1 /usr/sbin/sshd -D             0.3  0.0
```

### Ví dụ 16: Top tiến trình sử dụng CPU cao nhất

```
[root@localhost ~]# ps -eo pid,ppid,cmd,%mem,%cpu --sort=-%cpu | head -10
PID PPID CMD                          %MEM %CPU
1      0 /usr/lib/systemd/systemd      0.4  0.0
2      0 [kthreadd]                    0.0  0.0
3      2 [rcu_gp]                      0.0  0.0
...
```

## 8. Hiển thị thông tin bảo mật SELinux

### Ví dụ 17: Hiển thị context SELinux

```
# Hiển thị với context SELinux
[root@localhost ~]# ps -eM
LABEL                                    PID TTY      TIME CMD
system_u:system_r:init_t:s0                1 ?     00:00:02 systemd
system_u:system_r:kernel_t:s0              2 ?     00:00:00 kthreadd
...

# Hoặc sử dụng --context
[root@localhost ~]# ps --context
PID CONTEXT                                           COMMAND
1701 unconfined_u:unconfined_r:unconfined_t:s0-s0:c0.c1023 -bash
1818 unconfined_u:unconfined_r:unconfined_t:s0-s0:c0.c1023 ping 8.8.8.8
...
```

### Ví dụ 18: Hiển thị thông tin bảo mật chi tiết

```
[root@localhost ~]# ps -eo euser,ruser,suser,fuser,f,comm,label
EUSER RUSER SUSER FUSER F COMMAND LABEL
root  root  root  root  4 systemd system_u:system_r:init_t:s0
root  root  root  root  1 kthreadd system_u:system_r:kernel_t:s0
...
```

## 9. Giám sát thời gian thực

### Ví dụ 19: Sử dụng watch để giám sát real-time

```
# Giám sát top 10 tiến trình sử dụng bộ nhớ cao nhất, cập nhật mỗi giây
[root@localhost ~]# watch -n 1 'ps -eo pid,ppid,cmd,%mem,%cpu --sort=-%mem | head -10'

Every 1.0s: ps -eo pid,ppid,cmd,%mem,%cpu --sort=-%mem | head    Fri Jul 26 19:28:24 2019

PID PPID CMD                          %MEM %CPU
1033    1 /usr/bin/python -Es /usr/sb   1.9  0.0
1453    1 /usr/bin/python2 -Es /usr/s   1.0  0.0
991     1 /usr/lib/polkit-1/polkitd     0.8  0.0
...
```

## 10. Lệnh pstree - Hiển thị cây tiến trình

### Ví dụ 20: Sử dụng pstree cơ bản

```
[root@localhost ~]# pstree
systemd─┬─NetworkManager───2*[{NetworkManager}]
        ├─VGAuthService
        ├─agetty
        ├─anacron
        ├─auditd───{auditd}
        ├─crond
        ├─dbus-daemon───{dbus-daemon}
        ├─firewalld───{firewalld}
        ├─gssproxy───5*[{gssproxy}]
        ├─httpd───5*[httpd]
        ├─lvmetad
        ├─master─┬─pickup
        │        └─qmgr
        ├─polkitd───6*[{polkitd}]
        ├─rpcbind
        ├─rsyslogd───2*[{rsyslogd}]
        ├─sshd───sshd───bash─┬─4*[ping]
        │                    └─pstree
        ├─systemd-journal
        ├─systemd-logind
        ├─systemd-udevd
        ├─tuned───4*[{tuned}]
        └─vmtoolsd───{vmtoolsd}
```

### Ví dụ 21: pstree với arguments

```
[root@localhost ~]# pstree -a
systemd --switched-root --system --deserialize 22
├─NetworkManager --no-daemon
│   └─2*[{NetworkManager}]
├─VGAuthService -s
├─agetty --noclear tty1 linux
├─anacron -s
├─auditd
│   └─{auditd}
├─crond -n
├─dbus-daemon --system --address=systemd: --nofork
│   └─{dbus-daemon}
...
```

### Ví dụ 22: pstree với PID

```
[root@localhost ~]# pstree -p
systemd(1)─┬─NetworkManager(1069)─┬─{NetworkManager}(1080)
           │                      └─{NetworkManager}(1082)
           ├─VGAuthService(1013)
           ├─agetty(1030)
           ├─anacron(2195)
           ├─auditd(970)───{auditd}(971)
           ├─crond(1024)
           ├─dbus-daemon(997)───{dbus-daemon}(1011)
           ├─firewalld(1033)───{firewalld}(1241)
           ├─gssproxy(1004)─┬─{gssproxy}(1006)
           │                ├─{gssproxy}(1007)
           │                ├─{gssproxy}(1008)
           │                ├─{gssproxy}(1009)
           │                └─{gssproxy}(1010)
           ├─httpd(2069)─┬─httpd(2070)
           │             ├─httpd(2071)
           │             ├─httpd(2072)
           │             ├─httpd(2073)
           │             └─httpd(2074)
           ...
```

### Ví dụ 23: Làm nổi bật tiến trình cụ thể

```
# Làm nổi bật tiến trình httpd (PID 2070)
[root@localhost ~]# pstree -H 2070
systemd─┬─NetworkManager─┬─dhclient
        │                └─2*[{NetworkManager}]
        ├─VGAuthService
        ├─agetty
        ├─anacron
        ├─auditd───{auditd}
        ├─crond
        ├─dbus-daemon───{dbus-daemon}
        ├─firewalld───{firewalld}
        ├─gssproxy───5*[{gssproxy}]
        ├─httpd───5*[httpd]  # Tiến trình này sẽ được highlight
        ...
```

### Ví dụ 24: Hiển thị cây của tiến trình cụ thể

```
# Hiển thị cây từ tiến trình mẹ đến tiến trình con (PID 1853)
[root@localhost ~]# pstree -s 1853
systemd───sshd───sshd───bash───ping
```

### Ví dụ 25: Hiển thị Group ID

```
[root@localhost ~]# pstree -g
systemd(1)─┬─NetworkManager(1069)─┬─dhclient(2778)
           │                      ├─{NetworkManager}(1069)
           │                      ├─{NetworkManager}(1069)
           │                      └─{NetworkManager}(1069)
           ├─VGAuthService(1013)
           ├─agetty(1030)
           ├─anacron(2195)
           ├─auditd(970)───{auditd}(970)
           ...
```

## 11. Trạng thái tiến trình

### Bảng mã trạng thái tiến trình:

| Mã | Ý nghĩa |
|---|---|
| **D** | Uninterruptible sleep (thường là I/O) |
| **R** | Running hoặc runnable (trong hàng đợi chạy) |
| **S** | Interruptible sleep (chờ sự kiện hoàn thành) |
| **T** | Stopped (bởi job control signal hoặc đang được trace) |
| **W** | Paging (không hợp lệ từ kernel 2.6.xx) |
| **X** | Dead (không bao giờ thấy) |
| **Z** | Defunct ("zombie") - đã kết thúc nhưng chưa được parent thu hồi |

### Ký hiệu bổ sung:
| Ký hiệu | Ý nghĩa |
|---|---|
| **<** | High-priority (not nice to other users) |
| **N** | Low-priority (nice to other users) |
| **L** | Has pages locked into memory |
| **s** | Session leader |
| **l** | Multi-threaded |
| **+** | In foreground process group |

### Ví dụ 26: Tìm các tiến trình trong trạng thái chờ I/O

```
# Tìm các tiến trình ở trạng thái 'D' (Uninterruptible sleep)
[root@localhost ~]# ps aux | awk '$8 ~ /D/ { print $0 }'
root  473  0.1  0.0     0     0 ?   D   11:48   0:10 [kworker/u2:29]
root  563  0.0  0.0     0     0 ?   D   11:48   0:01 [xfsaild/dm-0]
root 5706  2.0  0.0 121136 1300 tty1 D+  14:29   0:01 cp -i -av FILE ISO/ /opt/
```

### Ví dụ 27: Tìm các tiến trình zombie

```
# Tìm các tiến trình zombie (trạng thái Z)
[root@localhost ~]# ps aux | awk '$8 ~ /Z/ { print $0 }'
```

## 12. Một số lệnh ps hữu ích khác

### Ví dụ 28: Hiển thị tất cả tiến trình đang chạy của người dùng hiện tại

```
[root@localhost ~]# ps -u $(whoami)
PID TTY      TIME CMD
1701 pts/0   00:00:00 bash
1818 pts/0   00:00:00 ping
1821 pts/0   00:00:00 ping
```

### Ví dụ 29: Tìm tiến trình parent của một tiến trình

```
# Tìm parent của tiến trình có PID 1818
[root@localhost ~]# ps -o ppid= -p 1818
1701
```

### Ví dụ 30: Hiển thị tiến trình sử dụng swap

```
# Hiển thị các tiến trình đang sử dụng swap
[root@localhost ~]# ps -eo pid,ppid,cmd,pmem,swap,psr --sort=-swap
```

### Ví dụ 31: Tìm tiến trình theo command line

```
# Tìm tất cả tiến trình có chứa "httpd" trong command line
[root@localhost ~]# ps aux | grep httpd | grep -v grep
apache  2070  0.0  0.0  43340  3564 ?  S    18:50   0:00 /usr/sbin/httpd -DFOREGROUND
apache  2071  0.0  0.0  43340  3564 ?  S    18:50   0:00 /usr/sbin/httpd -DFOREGROUND
```

### Ví dụ 32: Hiển thị tiến trình theo định dạng tùy chỉnh với header

```
[root@localhost ~]# ps -eo pid,ppid,pcpu,pmem,cmd --sort=-pcpu | head -10
PID  PPID %CPU %MEM CMD
1       0  0.1  0.3 /usr/lib/systemd/systemd --switched-root --system
2       0  0.0  0.0 [kthreadd]
3       2  0.0  0.0 [rcu_gp]
```

## 13. Tổng kết

Lệnh `ps` là một công cụ không thể thiếu trong quản trị hệ thống Linux. Với các tùy chọn đa dạng, nó cho phép:

- **Giám sát hiệu suất**: Theo dõi mức sử dụng CPU, RAM

- **Quản lý tiến trình**: Tìm kiếm, lọc tiến trình theo nhiều tiêu chí

- **Phân tích hệ thống**: Hiểu rõ mối quan hệ giữa các tiến trình

- **Troubleshooting**: Phát hiện các tiến trình có vấn đề

- **Bảo mật**: Kiểm tra các tiến trình đang chạy và quyền truy cập

**Các lệnh ps phổ biến nhất:**

- `ps aux` - Hiển thị tất cả tiến trình với thông tin chi tiết

- `ps -ef` - Hiển thị tất cả tiến trình với format đầy đủ

- `ps -eo pid,ppid,cmd,%mem,%cpu --sort=-%mem` - Sắp xếp theo sử dụng bộ nhớ

- `pstree` - Hiển thị cây tiến trình

- `ps -fC <command>` - Hiển thị tiến trình theo tên lệnh

Để sử dụng hiệu quả, hãy kết hợp `ps` với các lệnh khác như `grep`, `awk`, `sort`, và `head` để lọc và phân tích thông tin theo nhu cầu cụ thể.