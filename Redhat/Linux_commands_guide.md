## Hướng dẫn sử dụng các lệnh Linux

## 1. Lệnh `sort` - Sắp xếp dữ liệu

### Cú pháp cơ bản:

```
sort [options] [file...]
```

### Các ví dụ sử dụng:

**Sắp xếp file theo thứ tự alphabet:**

```
sort names.txt
```

**Sắp xếp theo số (numeric sort):**

```
sort -n numbers.txt
```

**Sắp xếp ngược (reverse):**

```
sort -r names.txt
```

**Sắp xếp theo cột thứ 2:**

```
sort -k2 data.txt
```

**Sắp xếp theo số và loại bỏ trùng lặp:**

```
sort -nu numbers.txt
```

**Ví dụ thực tế:**

```
# Sắp xếp danh sách IP theo thứ tự
sort -t. -k1,1n -k2,2n -k3,3n -k4,4n ip_list.txt

# Sắp xếp file CSV theo cột thứ 3
sort -t, -k3 employees.csv
```

## 2. Lệnh `uniq` - Loại bỏ/đếm dòng trùng lặp

### Cú pháp cơ bản:

```
uniq [options] [input] [output]
```

### Các ví dụ sử dụng:

**Loại bỏ dòng trùng lặp liên tiếp:**

```
uniq data.txt
```

**Đếm số lần xuất hiện:**

```
uniq -c data.txt
```

**Chỉ hiển thị dòng trùng lặp:**

```
uniq -d data.txt
```

**Chỉ hiển thị dòng không trùng lặp:**

```
uniq -u data.txt
```

**Ví dụ thực tế:**

```
# Đếm số lần truy cập theo IP từ log file
cat access.log | cut -d' ' -f1 | sort | uniq -c | sort -nr

# Tìm email trùng lặp
sort emails.txt | uniq -d
```

## 3. Lệnh `paste` - Ghép dòng từ nhiều file

### Cú pháp cơ bản:

```
paste [options] file1 file2 ...
```

### Các ví dụ sử dụng:

**Ghép 2 file theo dòng:**

```
paste file1.txt file2.txt
```

**Sử dụng delimiter tùy chỉnh:**

```
paste -d',' file1.txt file2.txt
```

**Ghép tất cả dòng của 1 file thành 1 dòng:**

```
paste -s -d' ' words.txt
```

**Ví dụ thực tế:**

```
# Tạo file CSV từ 3 cột riêng biệt
paste -d',' names.txt ages.txt cities.txt > people.csv

# Ghép số thứ tự với nội dung file
seq 1 10 | paste -d': ' - data.txt
```

## 4. Lệnh `join` - Kết hợp file dựa trên cột chung

### Cú pháp cơ bản:

```
join [options] file1 file2
```

### Các ví dụ sử dụng:

**Join 2 file theo cột đầu tiên:**

```
join file1.txt file2.txt
```

**Join theo cột khác:**

```
join -1 2 -2 1 file1.txt file2.txt
```

**Sử dụng delimiter tùy chỉnh:**

```
join -t',' file1.csv file2.csv
```

**Ví dụ thực tế:**

```
# Kết hợp thông tin nhân viên với phòng ban
join -t',' employees.csv departments.csv

# Left join (hiển thị tất cả dòng từ file1)
join -a1 -t',' users.csv orders.csv
```

## 5. Lệnh `split` - Chia file thành các phần nhỏ

### Cú pháp cơ bản:

```
split [options] [input] [prefix]
```

### Các ví dụ sử dụng:

**Chia file theo số dòng:**

```
split -l 1000 large_file.txt part_
```

**Chia file theo kích thước:**

```
split -b 1M large_file.txt chunk_
```

**Chia file theo kích thước với suffix số:**

```
split -b 1M -d large_file.txt part_
```

**Ví dụ thực tế:**

```
# Chia log file lớn thành các file 10MB
split -b 10M access.log log_part_

# Chia CSV file thành các file 5000 dòng
split -l 5000 --additional-suffix=.csv data.csv batch_
```

## 6. Lệnh `head` - Hiển thị đầu file

### Cú pháp cơ bản:

```
head [options] [file...]
```

### Các ví dụ sử dụng:

**Hiển thị 10 dòng đầu (mặc định):**

```
head file.txt
```

**Hiển thị n dòng đầu:**

```
head -n 5 file.txt
head -5 file.txt
```

**Hiển thị n bytes đầu:**

```
head -c 100 file.txt
```

**Ví dụ thực tế:**

```
# Xem header của CSV file
head -1 data.csv

# Xem 20 dòng đầu của log file
head -20 /var/log/syslog

# Theo dõi file mới (kết hợp với tail)
head -f -n 0 logfile.txt
```

## 7. Lệnh `tail` - Hiển thị cuối file

### Cú pháp cơ bản:

```
tail [options] [file...]
```

### Các ví dụ sử dụng:

**Hiển thị 10 dòng cuối (mặc định):**

```
tail file.txt
```

**Hiển thị n dòng cuối:**

```
tail -n 5 file.txt
tail -5 file.txt
```

**Theo dõi file real-time:**

```
tail -f logfile.txt
```

**Bắt đầu từ dòng thứ n:**

```
tail -n +10 file.txt
```

**Ví dụ thực tế:**

```
# Theo dõi log file real-time
tail -f /var/log/apache2/access.log

# Xem 50 dòng cuối của file lớn
tail -50 large_file.txt

# Theo dõi nhiều file cùng lúc
tail -f file1.log file2.log
```

## 8. Lệnh `less` - Xem file theo trang (nâng cao)

### Cú pháp cơ bản:

```
less [options] [file...]
```

### Các phím tắt trong less:

- `Space` hoặc `f`: Trang tiếp theo
- `b`: Trang trước
- `/pattern`: Tìm kiếm xuôi
- `?pattern`: Tìm kiếm ngược
- `n`: Tìm tiếp theo
- `N`: Tìm trước đó
- `q`: Thoát

### Các ví dụ sử dụng:

**Xem file cơ bản:**

```
less file.txt
```

**Xem với số dòng:**

```
less -N file.txt
```

**Tìm kiếm case-insensitive:**

```
less -i file.txt
```

**Ví dụ thực tế:**

```
# Xem log file lớn
less /var/log/syslog

# Xem output của command
ps aux | less

# Xem file với highlight syntax
less -R colored_output.txt
```

## 9. Lệnh `more` - Xem file theo trang (cơ bản)

### Cú pháp cơ bản:

```
more [options] [file...]
```

### Các phím tắt trong more:

- `Space`: Trang tiếp theo
- `Enter`: Dòng tiếp theo
- `q`: Thoát
- `/pattern`: Tìm kiếm

### Các ví dụ sử dụng:

**Xem file cơ bản:**

```
more file.txt
```

**Xem với số dòng trên mỗi trang:**

```
more -10 file.txt
```

**Ví dụ thực tế:**

```
# Xem manual page
man ls | more

# Xem output dài
ls -la /usr/bin | more
```

## Kết hợp các lệnh

### Ví dụ pipeline phổ biến:

```
# Sắp xếp và loại bỏ trùng lặp
sort file.txt | uniq

# Đếm từ xuất hiện nhiều nhất
cat text.txt | tr ' ' '\n' | sort | uniq -c | sort -nr | head -10

# Xử lý log file
tail -f access.log | grep "ERROR" | head -20

# Tạo backup file được chia nhỏ
split -b 1G large_database.sql backup_part_ && 
ls backup_part_* | head -5
```

### Tips sử dụng hiệu quả:

1. **Luôn sắp xếp trước khi dùng uniq** - uniq chỉ loại bỏ dòng trùng lặp liên tiếp

2. **Sử dụng less thay vì more** - less có nhiều tính năng hơn

3. **Kết hợp với grep, awk, sed** để xử lý text mạnh mẽ hơn

4. **Sử dụng pipe để kết hợp các lệnh** tạo workflow xử lý dữ liệu

5. **Kiểm tra man page** để biết thêm options: `man sort`, `man uniq`, etc.