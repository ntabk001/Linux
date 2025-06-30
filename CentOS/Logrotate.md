## Linux Log Logrotate

`LogRotate` là một công cụ trên hệ điều hành Linux dùng để quản lý file log. Nó giúp xử lý và lưu trữ file log một cách tự động, giúp tránh file log quá lớn và gây tắc nghẽn hệ thống. `LogRotate` cung cấp một số tùy chọn cấu hình để quản lý file log như xóa các file log cũ, nén các file log đã quá hạn, xác định tần suất xử lý file log và kích thước tối đa của file log trước khi xử lý.

* LogRotate có thể:

  * Xóa các file log cũ và tạo file log mới.
  * Nén các file log đã quá hạn để giảm dung lượng lưu trữ.
  * Xác định tần suất xử lý file log (hàng ngày, hàng tuần, hàng tháng).
  * Xác định kích thước tối đa của file log trước khi xử lý.

* Install logrotate

```
yum install logrotate -y
```

### 1. Các thông số thường gặp trong các tệp tin chính của logrotate  

| Thông số | Chức năng |  
|----------|-----------|
|daily | Mỗi ngày |
|weekly | Mỗi tuần |
|monthly | Mỗi tháng |
|yearly | Mỗi năm |
|missingok | Nếu file log bị mất hoặc không có tồn tại *.log thì logrotate sẽ di chuyển tới phần cấu hình log của file log khác mà không phải xuất ra thông báo lỗi |
|nomissingok | Ngược lại so với cấu hình missingok |
|notifempty | Không rotate log nếu file log này trống |
|rotate | số lượng file log cũ đã được giữ lại sau khi rotate |
|compress | Logrotate sẽ nén tất cả các file log lại sau khi đã được rotate mặc định bằng gzip |
|compresscmd | Khi sử dụng chương trình nén như bzip2, xz hoặc zip |
|delaycompress | Được sử dụng khi không muốn file log cũ phải nén ngay sau khi vừa được rotate |
|nocompress | Không sử dụng tính năng nén đối với file log cũ |
|create | Phân quyền cho file log mới sau khi rotate |
|copytruncate |	File log cũ được sao chép vào một tệp lưu trữ, và sau đó nó xóa các dòng log cũ |
|postrotate [command] endscript | Để chạy lệnh sau khi quá trình rotate kết thúc, chúng ta đặt lệnh thực thi nằm giữa postrotate và endscript |
|prerotate [command] endscript |Để chạy lệnh trước khi quá trình rotate bắt đầu, chúng ta đặt lệnh thực thi nằm giữa prerotate và endscript |
  
* Các tệp tin chính của logrotate:

  * Tập lệnh shell thực thi các lệnh logrotate hàng ngày được lưu tại `/etc/cron.daily/logrotate` 
  
```
cat /etc/cron.daily/logrotate
#!/bin/sh

/usr/sbin/logrotate -s /var/lib/logrotate/logrotate.status /etc/logrotate.conf
EXITVALUE=$?
if [ $EXITVALUE != 0 ]; then
    /usr/bin/logger -t logrotate "ALERT exited abnormally with [$EXITVALUE]"
fi
exit 0
```  
  
* Cấu hình Logrotate được lưu tại `/etc/logrotate.conf` có chứa các thông tin thiết lập toàn bộ file log mà `Logrotate` quản lý, bao gồm chu kì lặp, dung lượng file log, nén file,...

```
cat /etc/logrotate.conf
# see "man logrotate" for details
# rotate log files weekly
weekly

# keep 4 weeks worth of backlogs
rotate 4

# create new (empty) log files after rotating old ones
create

# use date as a suffix of the rotated file
dateext

# uncomment this if you want your log files compressed
#compress

# RPM packages drop log rotation information into this directory
include /etc/logrotate.d

# no packages own wtmp and btmp -- we'll rotate them here
/var/log/wtmp {
    monthly
    create 0664 root utmp
    minsize 1M
    rotate 1
}

/var/log/btmp {
    missingok
    monthly
    create 0600 root utmp
    rotate 1
}

# system-specific logs may be also be configured here.  
```  
  
Nội dung file `/etc/logrotate.conf` thể hiện file log được `rotate` hàng tuần, dữ liệu của file log được lưu trữ trong vòng 4 file, file log mới sẽ được tạo sau khi rotate file cũ.

Thông tin cấu hình của file log các ứng dụng được lưu trữ tại `/etc/logrotate.d`  
  
* Xem thông tin cấu hình của package `bootlog`: 

```
/etc/logrotate.d/bootlog
/var/log/boot.log
{
    missingok
    daily
    copytruncate
    rotate 7
    notifempty
}  
``` 
  * `missingok`: Nếu file log bị mất hoặc không tồn tại *.log thì logrotate sẽ tự động di chuyển tới phần cấu hình log của file log khác mà không cần phải xuất ra thông báo lỗi
  * `daily`: Được rotate mỗi ngày
  * `copytruncate`: File bootlog cũ được sao chép vào một tệp lưu trữ, và sau đó nó xóa các dòng log cũ
  * `rotate 7`: Giữ lại 7 file log cũ sau khi rotate
  * `notifempty`: Không rotate log nếu file bootlog này trống  
  
* Xem thông tin cấu hình của package `syslog`

```
cat /etc/logrotate.d/syslog
/var/log/cron
/var/log/maillog
/var/log/messages
/var/log/secure
/var/log/spooler
{
    missingok
    notifempty
    sharedscripts
    postrotate
    /bin/kill -HUP `cat /var/run/syslogd.pid 2> /dev/null` 2> /dev/null || true
    endscript
}  
```  
  
Khi `logrotate` chạy, nó sẽ kiểm tra bất kỳ tệp nào ở `/var/log/cron`, `/var/log/maillog`, `/var/log/messages`, `/var/log/secure`, `/var/log/spooler` và `rotate` chúng, nếu chúng không trống. Nếu nó kiểm tra thư mục `cron`, `maillog`, `messages`, `secure`, `spooler` và không tìm thấy bất kỳ tệp nhật ký nào, nó sẽ không phát sinh lỗi. Sau đó, nó chạy lệnh trong `postrotate/endscript` khối (trong trường hợp này là lệnh yêu cầu gán `/var/run/syslogd.pid` vào `/dev/null`), nhưng chỉ thực hiện sau khi nó xử lý tất cả các bản ghi đã chỉ định.

Trong tệp cấu hình `logrotate` với cấu trúc đơn giản gồm đường dẫn file log và thiết lập các cấu hình trong dấu {}.  
  
* Rotate theo dung lượng file log  
  
Chúng ta có thể quy định tiến trình rotate dựa vào dung lượng file. Với `/var/log/wtmp` khi `size 1M` sẽ tạo tệp mới:

```
cat logrotate.conf
/var/log/wtmp {
	size 1M
    create 0664 root utmp
    rotate 1
}
```

Ví dụ trên các tùy chọn có ý nghĩa như sau:

  * `size 1M`: Logrotate chỉ chạy nếu kích thước tệp bằng (hoặc lớn hơn) kích thước này.
  * `create`: Rotate tệp gốc và tạo tệp mới với sự cho phép người dùng và nhóm được chỉ định.
  * `rotate`: Giới hạn số vòng quay của fle log. Vì vậy, điều này sẽ chỉ giữ lại 1 file log được rotate gần nhất.  
  * `minsize`: Có nghĩa là kích thước nhật ký ít nhất phải là minsize để việc xoay tần số xảy ra. Tần suất hàng ngày (được gọi là hàng ngày từ cron) sẽ không có tác dụng gì nếu kích thước nhỏ hơn minsize.
  * `maxsize`: Có nghĩa là, ngoài tần suất chạy, nếu kích thước vượt quá maxsize, một vòng quay có thể xảy ra. Nếu cấu hình có tần suất hàng tuần được gọi là hàng ngày và nếu kích thước lớn hơn kích thước tối đa, thì việc xoay vòng có thể xảy ra.  
  
### 2. Chạy Logrotate thủ công

Khi chúng ta muốn chạy ngay `Logrotate`, hãy dùng lệnh bên dưới:

```
logrotate -vf /etc/logrotate.d/
```

Trong đó các tuỳ chọn là:

  * -v: Hiển thị thêm thông tin, có ích khi bạn muốn dò lỗi `logrotate`

  * -f: Bắt buộc `rotate` ngay lập tức  
  
### 3. Tùy chọn sao chép logrotate

Tiếp tục ghi thông tin nhật ký vào tệp vừa tạo sau khi xoay tệp nhật ký cũ.

```
cat logrotate.conf
/var/log/wtmp {
	size 1M
    copytruncate 
    rotate 1
}
```

Tùy chọn `copytruncate` hướng dẫn `logrotate` tạo bản sao của tệp gốc và cắt tệp gốc có kích thước 0 byte. Điều này giúp dịch vụ tương ứng thuộc về file log đó có thể ghi vào tệp thích hợp.  
  
### 4. Tuỳ chọn nén

`Logrotate` sẽ nén tất cả các file log khi được rotate và mặc định sẽ sử dụng chương trình nén bằng `gzip` sử dụng tuỳ chọn sau: `compress`

Chúng ta cũng có thể sử dụng các chương trình nén khác như `xz`, `bzip2`, `zip` bằng các sử dụng tùy chọn: `compresscmd [Chương trình nén]`.

Ví dụ: Nén các tệp đã được rotate với gzip chúng ta thực hiện như sau:

```
cat logrotate.conf
/var/log/wtmp {
	size 1M
    create 0664 root utmp
    rotate 1
    compress
}  
```

### 5. Tùy chọn logextate dateext

`Rotate` file log cũ có ngày trong tên file

```
cat logrotate.conf
/var/log/wtmp {
	size 1M
    create 0664 root utmp
    dateext 
    rotate 1
    compress
}
```

Cú pháp này sẽ chỉ làm việc một lần trong một ngày. Bởi vì khi nó `rotate` lần sau vào cùng ngày, tệp được xoay trước đó sẽ có cùng tên tệp. Vì vậy, `logrotate` sẽ không thành công sau lần chạy đầu tiên cùng ngày.
 
### 6. Tùy chọn đăng nhập hàng tháng, hàng ngày, hàng tuần

Rotate file log `hàng tuần`, `hàng ngày`, `hàng tháng`

* Để thực hiện hàng tháng một lần:

```
cat logrotate.conf
/var/log/wtmp {
	monthly
    copytruncate
    rotate 1
    compress
}  
```

* Để thực hiện hàng tuần thực hiện như sau:

```
cat logrotate.conf
/var/log/wtmp {
	weekly
    copytruncate
    rotate 1
    compress
}
```

* Để thực hiện hàng ngày thực hiện như sau:

```
cat logrotate.conf
/var/log/wtmp {
	daily
    copytruncate
    rotate 1
    compress
}
```

### 7. Tùy chọn kết thúc postrotate postrotate:

`Logrotate` cho phép bạn chạy các tập lệnh shell tùy chỉnh của riêng bạn sau khi hoàn thành rotate file log. Cấu hình sau chỉ ra rằng nó sẽ thực thi `myscript.sh` sau khi `logrotation`.

```
cat logrotate.conf
/var/log/wtmp {
	size 1M
    copytruncate 
    rotate 1
    compress
    postrotate
           /home/root/myscript.sh
    endscript
}
```

### 8. Tùy chọn maxage Logrotate

Logrotate tự động loại bỏ `rotate` file sau một số ngày cụ thể.

* Ví dụ: rotate file sẽ bị xóa sau 20 ngày.

```
cat logrotate.conf
/var/log/wtmp {
	size 1M
    copytruncate 
    rotate 1
    compress
    maxage 20
}
```

### 9. Tùy chọn Logrotate missingok

Chúng ta có thể bỏ qua thông báo lỗi khi tệp thực tế không có sẵn bằng cách sử dụng tùy chọn như bên dưới.

```
cat logrotate.conf
/var/log/wtmp {
	size 1M
    copytruncate 
    rotate 1
    compress
    missingok
}
```  