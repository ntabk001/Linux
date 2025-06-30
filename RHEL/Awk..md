## Tổng hợp kiến thức về lệnh `awk`

`awk` là một công cụ xử lý văn bản mạnh mẽ trong Linux/Unix, có thể thực hiện nhiều thao tác phức tạp trên dữ liệu dạng text hoặc file.

### 1. Cú pháp cơ bản

```
awk 'pattern {action}' file
command | awk 'pattern {action}'
```

### 2. Các biến đặc biệt trong awk

|Biến | Ý nghĩa |
|-----|---------|
|$0 | Toàn bộ dòng hiện tại|
|$1, $2,..., $n | Trường thứ 1, 2,..., n|
|NR | Số dòng hiện tại (Number of Records)|
|NF | Số trường trong dòng hiện tại (Number of Fields)|
|FS | Dấu phân cách trường (Field Separator), mặc định là khoảng trắng|
|OFS | Dấu phân cách trường đầu ra (Output Field Separator)|
|RS | Dấu phân cách dòng (Record Separator)|
|ORS | Dấu phân cách dòng đầu ra|

### 3. Các trường hợp sử dụng phổ biến

### a. In nội dung file

```
awk '{print}' file.txt        # In toàn bộ file
awk '{print $0}' file.txt     # Tương tự trên
awk 'BEGIN { action; } /search/ { action; } END { action; }' filename
awk '/search-pattern/ { action-to-take-on-matches; another-action; }' file-to-parse
```

* Chú thích tham số lệnh awk:
  * `/search-patterm/`: Là nội dung mẫu tìm kiếm.
  * `action-to-take-on-matches`: Biểu thức xử lí nội dung.
  * `;`: Dấu kết thúc biểu thức xử lí nội dung.
  * `another-action`: Biểu thức xử lí nội dung khác.
  * `file-to-parse`: Là tên tệp hoăc đường dẫn đến tệp.

### b. In các trường cụ thể

```
awk '{print $1, $3}' file.txt          # In trường 1 và 3
awk '{print $NF}' file.txt             # In trường cuối cùng
awk '{print $(NF-1)}' file.txt         # In trường áp cuối
```

### c. Lọc dữ liệu theo điều kiện

```
awk '/pattern/ {print}' file.txt       # In dòng chứa "pattern"
awk 'NR==5 {print}' file.txt           # In dòng thứ 5
awk 'NR>=10 && NR<=20' file.txt        # In từ dòng 10 đến 20
awk '$3 > 100 {print}' file.txt        # In dòng có trường 3 > 100
```

### d. Thay đổi dấu phân cách

```
awk -F: '{print $1}' /etc/passwd       # Phân tách bằng dấu ":"
awk 'BEGIN {FS=":"; OFS="---"} {print $1,$6}' /etc/passwd
```

### e. Tính toán

```
awk '{sum += $1} END {print sum}' file.txt    # Tính tổng cột 1
awk '{avg = sum/NR} END {print avg}' file.txt # Tính trung bình
```

### f. Xử lý theo dòng

```
awk 'NR % 2 == 0 {print}' file.txt    # In các dòng chẵn
awk 'length($0) > 80' file.txt        # In dòng dài hơn 80 ký tự
```

### 4. Ví dụ thực tế

### a. Phân tích log file

```
# Đếm số lần xuất hiện của IP trong access log
awk '{print $1}' access.log | sort | uniq -c | sort -nr

# Lấy top 5 IP truy cập nhiều nhất
awk '{print $1}' access.log | sort | uniq -c | sort -nr | head -5
```

### b. Xử lý file CSV

```
# Tính tổng cột 3 trong file CSV
awk -F, '{sum += $3} END {print sum}' data.csv

# Lọc dòng có giá trị cột 2 > 50
awk -F, '$2 > 50' data.csv
```

### c. Quản lý hệ thống

```
# Liệt kê các process sử dụng nhiều CPU
ps aux | awk '$3 > 5.0 {print $0}'

# Tính tổng kích thước thư mục
du -sk * | awk '{sum += $1} END {print sum/1024 " MB"}'
```

### d. Biến đổi dữ liệu

```
# Đảo ngược thứ tự các cột
awk '{for (i=NF; i>0; i--) printf "%s ", $i; print ""}' file.txt

# Thêm số dòng vào file
awk '{print NR, $0}' file.txt
```

### e. Xử lý file cấu hình

```
# Lấy tất cả user từ /etc/passwd
awk -F: '{print $1}' /etc/passwd

# Lấy user có shell là /bin/bash
awk -F: '$7 == "/bin/bash" {print $1}' /etc/passwd
```

### 5. Awk script file

Khi cần thực hiện các thao tác phức tạp, có thể viết script riêng:

```
# script.awk
BEGIN {
    print "Bắt đầu xử lý file"
    count = 0
}
{
    if ($3 > 50) {
        print $0
        count++
    }
}
END {
    print "Tổng số bản ghi thỏa mãn:", count
}
```

Chạy script:

```
awk -f script.awk data.txt
```

### 6. Mẹo và thủ thuật

* In số dòng: `awk 'END{print NR}' file.txt`

* Đếm từ: `awk '{total += NF} END {print total}' file.txt`

* Tìm `max/min`: `awk 'NR==1{max=$1;next} $1>max{max=$1} END{print max}' file.txt`

* In dòng dài nhất: `awk 'length > max_length {max_length = length; longest_line = $0} END {print longest_line}' file.txt`

* Thay thế text: `awk '{gsub(/old/, "new"); print}' file.txt`

* `NF`: là một biến tích hợp có chứa số lượng các trường trong bản ghi hiện tại. Vì vậy `$NF` đưa ra trường cuối cùng và `$(NF-1)` sẽ đưa ra trường cuối cùng thứ hai.

Ví dụ: Chúng ta hãy in thông tin thanh toán hóa đơn ở phần trước chúng ta tạo như sau:

```
awk 'BEGIN {print "HOA DON THANH TOAN";} {s+=$5;} END {print "Tong tien thanh toan la: " s;}' text.txt

Kết quả
HOA DON THANH TOAN
Tong tien thanh toan la: 930000
```

* Chuyển đổi dữ liệu tệp thành bảng, bây giờ chúng ta hãy chuyển đổi dữ liệu tệp `passwd` tại `/etc/passwd` thành bảng bằng câu lệnh sau:

```
awk 'BEGIN { FS=":"; print "User\t\tUID\t\tGID\t\tHome\t\tShell\n"; } {print $1,"\t\t",$3,"\t\t",$4,"\t\t",$6,"\t\t",$7;} END { print "---------\nChuyen doi tep thanh cong" }' /etc/passwd
Kết quả
User            UID             GID             Home            Shell

root             0               0               /root           /bin/bash
daemon           1               1               /usr/sbin       /usr/sbin/nologin
bin              2               2               /bin            /usr/sbin/nologin
sys              3               3               /dev            /usr/sbin/nologin
sync             4               65534           /bin            /bin/sync
games            5               60              /usr/games      /usr/sbin/nologin
man              6               12              /var/cache/man  /usr/sbin/nologin
lp               7               7               /var/spool/lpd  /usr/sbin/nologin
mail             8               8               /var/mail       /usr/sb
---------
Chuyen doi tep thanh cong
```

* Sử dụng awk để tìm kiếm tên loại trái cây chỉ khớp nội dung ở đầu cột thứ hai bằng cách sử dụng lệnh sau:

```
awk '$2 ~ /^K/' text.txt
```

* Chúng ta cũng có thể dễ dàng tìm kiếm những thứ không khớp với từ khóa tìm kiếm bằng cách thêm ký tự ! trước dấu ngã `(~)`. Lệnh này sẽ trả về tất cả các dòng không có tên loại quả bắt đầu bằng `C` như sau:

```
awk '$2 !~ /^C/' text.txt
Kết quả
STT     Fruit   Amount  Price   Total
2       Xoài    6       30000   180000
3       Bưởi    7       40000   280000
4       Kiwi    4       60000   240000
5       Mận     10      20000   200000
```

* Ngoài ra, chúng ta cũng có thể sử dụng biểu thức điều kiện để xử lí tệp đầu vào bằng cách:

```
awk '$2 !~ /^C/ && $1 < 3' text.txt
Kết quả
2       Xoài    6       30000   180000
```

* Another use of NR built-in variables (Display Line From 3 to 6)  

```
awk 'NR==3, NR==6 {print NR,$0}' text.txt
```

* Để in mục đầu tiên cùng với số hàng (NR) được phân tách bằng dấu `-` từ mỗi dòng trong text.txt

```
awk '{print NR "- " $1 }' geeksforgeeks.txt 
1 - A
2 - Tarun
3 – Manav    
4 - Praveen
```

* In các dòng có hơn 10 ký tự

```
awk 'length($0) > 10' geeksforgeeks.txt 
```

* Để tìm/kiểm tra bất kỳ chuỗi nào trong bất kỳ cột cụ thể nào

```
awk 'BEGIN { for(i=1;i<=6;i++) print "square of", i, "is",i*i; }' 
```

###  Chỉ định tách trường đầu vào

Sử dụng tùy chọn `-F` để chỉnh định tách trường đầu vào

```
echo 'foo:123:bar:456' | awk -F: '{print $2}'
123

echo 'foo:123:bar:456' | awk -F: '{print $NF}'
456

echo 'foo:123:bar:456' | awk -F: '{print $1, $NF}'
foo 456

echo 'foo:123:bar:456' | awk -F: '{print $(NF-1)}'
bar

echo 'one;two;three;four' | awk -F';' '{print $2}'
two
```

### Phép so sánh

* Sử dụng lệnh awk thực hiện như sau `awk '$1 > 200' file1.txt`. Nếu `$1` lớn hơn `200` thì chương trình sẽ thực hiện in nội dung của `file1.txt`.

```
cat file1.txt
500  Sanjay  Sysadmin   Technology  $7,000
300  Nisha   Manager    Marketing   $9,500
400  Randy   DBA        Technology  $6,000

awk '$1 > 200' file1.txt
500  Sanjay  Sysadmin   Technology  $7,000
300  Nisha   Manager    Marketing   $9,500
400  Randy   DBA        Technology  $6,000
```

### Thay thế

```
echo  ' 1-2-3-4-5 '  | awk ' {sub ("-", ":")} 1 '
1:2-3-4-5
 
echo  ' 1-2-3-4-5 '  | awk ' {gsub ("-", ":")} 1 '
1:2:3:4:5
 
echo '1-2-3-4-5' | awk '{n=gsub("-", ":"); print n} 1'
4

1:2:3:4:5

echo '1-2-3-4-5' | awk '{gsub(/[^-]+/, "abc")} 1'
abc-abc-abc-abc-abc

echo 'one;two;three;four' | awk -F';' '{gsub("e", "E", $3)} 1'
one two thrEE four
```

### Tính tổng giá trị

Lệnh `awk` thực hiện tính tổng dựa trên cú pháp sau:

```
awk '{s+=$(cột cần tính)} END {print s}' {{filename}}
```

```cat file1.txt
500  Sanjay  Sysadmin   Technology  $7,000
300  Nisha   Manager    Marketing   $9,500
400  Randy   DBA        Technology  $6,000

awk '{s+=$1} END {print s}' file1.txt
1200

awk '{s+=$1; print $(cột cần tính)} END {print "--------"; print s}' {{filename}}

cat file1.txt
500  Sanjay  Sysadmin   Technology  $7,000
300  Nisha   Manager    Marketing   $9,500
400  Randy   DBA        Technology  $6,000

awk '{s+=$1; print $1} END {print "--------"; print s}' file1.txt
500
300
400
--------
1200
```