# Linux Text Processing Commands

## 1. LỆNH TR (Translate/Transform)

### Cú pháp cơ bản

```
tr [options] SET1 [SET2]
```

### 1.1 Chuyển đổi ký tự

**Chuyển chữ thường thành chữ hoa:**

```
echo "hello world" | tr 'a-z' 'A-Z'
# Output: HELLO WORLD

# Hoặc sử dụng character classes
echo "hello world" | tr '[:lower:]' '[:upper:]'
# Output: HELLO WORLD
```

**Chuyển chữ hoa thành chữ thường:**

```
echo "HELLO WORLD" | tr 'A-Z' 'a-z'
# Output: hello world

# Đọc từ file
tr '[:upper:]' '[:lower:]' < /etc/hostname
```

### 1.2 Thay thế ký tự cụ thể

**Thay thế dấu cách bằng dấu gạch dưới:**

```
echo "hello world test" | tr ' ' '_'
# Output: hello_world_test

# Ứng dụng thực tế: Đổi tên file
filename="My Important Document.txt"
new_filename=$(echo "$filename" | tr ' ' '_')
echo $new_filename
# Output: My_Important_Document.txt
```

**Thay thế nhiều ký tự:**

```
echo "hello,world;test:data" | tr ',;:' '___'
# Output: hello_world_test_data

# Thay thế bằng các ký tự khác nhau
echo "a1b2c3" | tr '123' 'xyz'
# Output: axbycz
```

### 1.3 Xóa ký tự (-d option)

**Xóa số:**

```
echo "abc123def456" | tr -d '0-9'
# Output: abcdef

# Xóa ký tự đặc biệt
echo "hello@world#test!" | tr -d '@#!'
# Output: helloworldtest
```

**Xóa dấu cách:**

```
echo "h e l l o   w o r l d" | tr -d ' '
# Output: helloworld
```

### 1.4 Nén ký tự lặp (-s option)

**Nén dấu cách:**

```
echo "hello    world     test" | tr -s ' '
# Output: hello world test

# Nén newlines
cat file.txt | tr -s '\n'
```

**Nén và thay thế:**

```
echo "hello,,,world;;;test" | tr -s ',;' '_'
# Output: hello_world_test
```

### 1.5 Ví dụ thực tế với tr

**Làm sạch số điện thoại:**

```
phone="(084) 123-456-789"
clean_phone=$(echo "$phone" | tr -d '()- ')
echo $clean_phone
# Output: 084123456789
```

**Tạo password từ string:**

```
echo "hello world" | tr 'a-z' 'A-Z' | tr -d ' ' | tr 'AEIOU' '12345'
# Output: H2LL4W4RLD
```

**Chuyển đổi Windows line endings:**

```
tr -d '\r' < windows_file.txt > unix_file.txt
```

## 2. LỆNH TEE (T-pipe)

### Cú pháp cơ bản

```
command | tee [options] file
```

### 2.1 Ghi vào file và hiển thị

**Ghi kết quả vào file:**

```
ls -la | tee directory_listing.txt
# Hiển thị trên màn hình VÀ ghi vào file

# Ghi vào nhiều file
echo "Hello World" | tee file1.txt file2.txt file3.txt
```

**Append vào file (-a option):**

```
date | tee -a log.txt
# Thêm vào cuối file thay vì ghi đè

# Log với timestamp
echo "Application started" | tee -a /var/log/myapp.log
```

### 2.2 Ví dụ thực tế với tee

**Logging trong script:**

```
#!/bin/bash
echo "Starting backup process..." | tee -a backup.log
rsync -av /home/user/ /backup/ | tee -a backup.log
echo "Backup completed at $(date)" | tee -a backup.log
```

**Debugging pipeline:**

```
cat large_file.txt | 
  grep "error" | 
  tee debug_grep.txt | 
  cut -d' ' -f1 | 
  tee debug_cut.txt | 
  sort | 
  tee debug_sort.txt
```

**Ghi với sudo:**

```
# Ghi vào file cần quyền root
echo "new content" | sudo tee /etc/config.conf

# Append với sudo
echo "additional line" | sudo tee -a /etc/config.conf
```

## 3. LỆNH WC (Word Count)

### Cú pháp cơ bản

```
wc [options] [file...]
```

### 3.1 Đếm cơ bản

**Đếm lines, words, characters:**

```
wc file.txt
# Output: 10 50 300 file.txt (lines words chars)

# Chỉ đếm lines
wc -l file.txt
# Output: 10 file.txt

# Chỉ đếm words
wc -w file.txt
# Output: 50 file.txt

# Chỉ đếm characters
wc -c file.txt
# Output: 300 file.txt
```

### 3.2 Ví dụ thực tế với wc

**Đếm files trong directory:**

```
ls | wc -l
# Đếm số file/folder trong thư mục hiện tại

find /home/user -type f | wc -l
# Đếm tổng số file trong /home/user
```

**Đếm processes:**

```
ps aux | wc -l
# Đếm số processes đang chạy

ps aux | grep apache | wc -l
# Đếm số processes Apache
```

**Đếm connections:**

```
netstat -an | grep :80 | wc -l
# Đếm số kết nối đến port 80

ss -tuln | wc -l
# Đếm số socket đang lắng nghe
```

**Thống kê log file:**

```
# Đếm số lần xuất hiện lỗi
grep -i "error" /var/log/messages | wc -l

# Đếm số IP unique trong access log
cut -d' ' -f1 /var/log/apache2/access.log | sort | uniq | wc -l
```

## 4. LỆNH CUT (Extract columns)

### Cú pháp cơ bản

```
cut [options] [file...]
```

### 4.1 Cắt theo delimiter

**Cắt theo dấu phẩy:**

```
echo "name,age,city" | cut -d',' -f1
# Output: name

echo "john,25,hanoi" | cut -d',' -f2
# Output: 25

# Lấy nhiều fields
echo "john,25,hanoi,vietnam" | cut -d',' -f1,3
# Output: john,hanoi

# Lấy từ field 2 trở đi
echo "john,25,hanoi,vietnam" | cut -d',' -f2-
# Output: 25,hanoi,vietnam
```

**Cắt theo dấu cách:**

```
echo "user daemon 1234 /usr/sbin/nginx" | cut -d' ' -f1,3
# Output: user 1234

# Xử lý nhiều dấu cách
ps aux | tr -s ' ' | cut -d' ' -f1,2,11
```

### 4.2 Cắt theo vị trí ký tự

**Cắt theo position:**

```
echo "hello world" | cut -c1-5
# Output: hello

echo "hello world" | cut -c7-
# Output: world

# Lấy ký tự cụ thể
echo "hello world" | cut -c1,3,5
# Output: hlo
```

### 4.3 Ví dụ thực tế với cut

**Lấy thông tin từ /etc/passwd:**

```
# Lấy username
cut -d':' -f1 /etc/passwd

# Lấy username và home directory
cut -d':' -f1,6 /etc/passwd

# Lấy shell của user cụ thể
grep "^$USER:" /etc/passwd | cut -d':' -f7
```

**Xử lý CSV file:**

```
# Giả sử có file employees.csv: name,age,department,salary
cut -d',' -f1,4 employees.csv | head -10
# Lấy name và salary của 10 nhân viên đầu

# Lấy department unique
cut -d',' -f3 employees.csv | sort | uniq
```

**Lấy thông tin từ log:**

```
# Lấy IP từ access log
cut -d' ' -f1 /var/log/apache2/access.log | head -10

# Lấy HTTP status code
cut -d' ' -f9 /var/log/apache2/access.log | sort | uniq -c
```

**Xử lý output của lệnh hệ thống:**

```
# Lấy tên process
ps aux | cut -d' ' -f11- | head -10

# Lấy filesystem usage
df -h | cut -d' ' -f1,5 | grep -v "Use%"
```

## 5. LỆNH SED (Stream Editor)

### Cú pháp cơ bản

```
sed [options] 'command' [file...]
```

### 5.1 Thay thế text (substitute)

**Thay thế cơ bản:**

```
echo "hello world" | sed 's/world/universe/'
# Output: hello universe

# Thay thế tất cả occurrences
echo "hello world world" | sed 's/world/universe/g'
# Output: hello universe universe

# Thay thế case-insensitive
echo "Hello World" | sed 's/hello/hi/i'
# Output: hi World
```

**Thay thế trong file:**

```
# Thay thế và hiển thị
sed 's/old_text/new_text/g' file.txt

# Thay thế và lưu vào file mới
sed 's/old_text/new_text/g' file.txt > new_file.txt

# Thay thế trực tiếp trong file (-i option)
sed -i 's/old_text/new_text/g' file.txt

# Backup trước khi thay thế
sed -i.bak 's/old_text/new_text/g' file.txt
```

### 5.2 Xóa dòng (delete)

**Xóa dòng theo pattern:**

```
# Xóa dòng chứa "delete_this"
sed '/delete_this/d' file.txt

# Xóa dòng trống
sed '/^$/d' file.txt

# Xóa dòng comment (bắt đầu bằng #)
sed '/^#/d' file.txt
```

**Xóa dòng theo số:**

```
# Xóa dòng 3
sed '3d' file.txt

# Xóa dòng 2 đến 5
sed '2,5d' file.txt

# Xóa dòng cuối
sed '$d' file.txt
```

### 5.3 Thêm text (append/insert)

**Thêm text:**

```
# Thêm dòng sau dòng chứa "pattern"
sed '/pattern/a\New line added' file.txt

# Thêm dòng trước dòng chứa "pattern"
sed '/pattern/i\New line inserted' file.txt

# Thêm vào cuối file
sed '$a\End of file' file.txt
```

### 5.4 Ví dụ thực tế với sed

**Thay đổi cấu hình:**

```
# Thay đổi port trong config
sed -i 's/^port=.*/port=8080/' app.conf

# Uncomment dòng
sed -i 's/^#\(.*setting.*\)/\1/' config.conf

# Comment dòng
sed -i 's/^debug=true/#debug=true/' config.conf
```

**Xử lý log file:**

```
# Lọc log theo thời gian
sed -n '/2024-01-01/,/2024-01-02/p' app.log

# Xóa log cũ
sed -i '/2023-/d' app.log

# Thay thế sensitive data
sed -i 's/password=[^[:space:]]*/password=****/g' app.log
```

**Xử lý code:**

```
# Thay đổi variable name
sed -i 's/\bold_var\b/new_var/g' *.py

# Thêm header vào file
sed -i '1i\#!/usr/bin/env python3' script.py

# Xóa trailing whitespace
sed -i 's/[[:space:]]*$//' file.txt
```

## 6. LỆNH GREP (Global Regular Expression Print)

### Cú pháp cơ bản

```
grep [options] pattern [file...]
```

### 6.1 Tìm kiếm cơ bản

**Tìm kiếm text:**

```
grep "search_term" file.txt

# Case-insensitive
grep -i "search_term" file.txt

# Hiển thị số dòng
grep -n "search_term" file.txt

# Hiển thị tên file
grep -H "search_term" file.txt

# Đếm số lần xuất hiện
grep -c "search_term" file.txt
```

**Tìm kiếm recursive:**

```
# Tìm trong tất cả file trong directory
grep -r "search_term" /path/to/directory

# Tìm chỉ trong file .txt
grep -r --include="*.txt" "search_term" /path/to/directory

# Loại trừ file .log
grep -r --exclude="*.log" "search_term" /path/to/directory
```

### 6.2 Tìm kiếm với regex

**Regex patterns:**

```
# Tìm dòng bắt đầu bằng "Error"
grep "^Error" log.txt

# Tìm dòng kết thúc bằng "failed"
grep "failed$" log.txt

# Tìm IP address
grep -E '[0-9]{1,3}\.[0-9]{1,3}\.[0-9]{1,3}\.[0-9]{1,3}' log.txt

# Tìm email
grep -E '[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\.[a-zA-Z]{2,}' file.txt
```

### 6.3 Tìm kiếm ngược và context

**Tìm kiếm ngược:**

```
# Tìm dòng KHÔNG chứa pattern
grep -v "exclude_this" file.txt

# Tìm file KHÔNG chứa pattern
grep -L "pattern" *.txt
```

**Hiển thị context:**

```
# Hiển thị 3 dòng trước và sau
grep -C 3 "pattern" file.txt

# Hiển thị 5 dòng trước
grep -B 5 "pattern" file.txt

# Hiển thị 5 dòng sau
grep -A 5 "pattern" file.txt
```

### 6.4 Ví dụ thực tế với grep

**Tìm kiếm trong log:**

```
# Tìm lỗi trong log
grep -i "error\|warning\|fail" /var/log/messages

# Tìm failed login
grep "Failed password" /var/log/auth.log

# Tìm IP có nhiều kết nối
grep -o '[0-9]\{1,3\}\.[0-9]\{1,3\}\.[0-9]\{1,3\}\.[0-9]\{1,3\}' /var/log/apache2/access.log | sort | uniq -c | sort -nr
```

**Tìm kiếm trong code:**

```
# Tìm function definition
grep -n "def function_name" *.py

# Tìm TODO comments
grep -r "TODO\|FIXME\|BUG" src/

# Tìm hardcoded passwords
grep -r -i "password\s*=" . --include="*.py"
```

**System monitoring:**

```
# Kiểm tra processes
ps aux | grep apache

# Kiểm tra network connections
netstat -an | grep :80

# Kiểm tra memory usage
free -h | grep Mem

# Kiểm tra disk usage
df -h | grep -v "tmpfs"
```

## 7. KẾT HỢP CÁC LỆNH (Pipeline Examples)

### 7.1 Phân tích log file

```
# Top 10 IP addresses trong access log
cut -d' ' -f1 /var/log/apache2/access.log | sort | uniq -c | sort -nr | head -10

# Top 10 requested URLs
cut -d' ' -f7 /var/log/apache2/access.log | sort | uniq -c | sort -nr | head -10

# Thống kê HTTP status codes
cut -d' ' -f9 /var/log/apache2/access.log | sort | uniq -c | sort -nr

# Lọc và đếm lỗi 404
grep " 404 " /var/log/apache2/access.log | cut -d' ' -f1 | sort | uniq -c | sort -nr
```

### 7.2 Xử lý CSV data

```
# Tạo file CSV mẫu
cat > employees.csv << 'EOF'
name,age,department,salary
john,25,IT,50000
jane,30,HR,45000
bob,35,IT,60000
alice,28,Finance,55000
EOF

# Lấy nhân viên IT
grep ",IT," employees.csv | cut -d',' -f1,4

# Tính tổng lương IT
grep ",IT," employees.csv | cut -d',' -f4 | tr -d '$' | awk '{sum+=$1} END {print sum}'

# Thay thế department
sed 's/,IT,/,Technology,/g' employees.csv | tee employees_updated.csv
```

### 7.3 System administration

```
# Cleanup log files
find /var/log -name "*.log" -type f -exec wc -l {} + | sort -nr | head -10

# Find large files
find /home -type f -exec wc -c {} + | sort -nr | head -10 | cut -d' ' -f2- | tr ' ' '\t'

# User analysis
cut -d':' -f1,3,6 /etc/passwd | grep -E ":[0-9]{4,}:" | cut -d':' -f1,3 | sort -t':' -k2 -n

# Network monitoring
netstat -an | grep ESTABLISHED | cut -d' ' -f5 | cut -d':' -f1 | sort | uniq -c | sort -nr
```

### 7.4 Text processing pipeline

```
# Clean và format text
cat messy_file.txt | 
  tr '[:upper:]' '[:lower:]' |          # Chuyển thành lowercase
  sed 's/[[:punct:]]//g' |             # Xóa punctuation
  tr -s ' ' |                          # Nén spaces
  sed 's/^ *//;s/ *$//' |              # Trim leading/trailing spaces
  grep -v '^$' |                       # Xóa dòng trống
  sort |                               # Sort
  uniq -c |                            # Count unique
  sort -nr |                           # Sort by count
  head -20 |                           # Top 20
  tee word_frequency.txt               # Save và display
```

## 8. TIPS VÀ TRICKS

### 8.1 Performance tips

```
# Sử dụng grep thay vì cat | grep
# Chậm:
cat large_file.txt | grep "pattern"

# Nhanh:
grep "pattern" large_file.txt

# Sử dụng cut thay vì awk cho simple field extraction
# Chậm:
awk -F',' '{print $1}' file.csv

# Nhanh:
cut -d',' -f1 file.csv
```

### 8.2 Error handling

```
# Kiểm tra file tồn tại
if [ -f "file.txt" ]; then
    wc -l file.txt
else
    echo "File not found"
fi

# Ignore errors
grep "pattern" file.txt 2>/dev/null || echo "Pattern not found"

# Redirect errors
sed 's/old/new/g' file.txt 2>errors.log 1>output.txt
```

### 8.3 Debugging pipelines

```
# Sử dụng tee để debug
cat file.txt | 
  tee debug1.txt | 
  grep "pattern" | 
  tee debug2.txt | 
  cut -d' ' -f1 | 
  tee debug3.txt | 
  sort

# Verbose output
set -x  # Enable debugging
grep -v "exclude" file.txt | cut -d',' -f1 | sort
set +x  # Disable debugging
```

Các lệnh này là foundation của text processing trong Linux và có thể kết hợp với nhau để tạo ra những pipeline mạnh mẽ!