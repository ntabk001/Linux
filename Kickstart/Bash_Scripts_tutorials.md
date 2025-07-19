# Hướng dẫn Bash Shell Scripting

## Phần 1: Câu lệnh If trong Bash Shell Scripting

### Câu lệnh điều kiện (Conditional Statements)

Câu lệnh điều kiện cho phép chúng ta quyết định có thực hiện một hành động hay không, quyết định này được đưa ra bằng cách đánh giá một biểu thức. Dạng cơ bản nhất là:

```
if [ expression ];

then

      statements

elif [ expression ];

then

      statements

else

      statements

fi
```

- Các phần `elif` (else if) và `else` là tùy chọn
- Đặt khoảng trắng sau `[` và trước `]`, và xung quanh các toán tử và toán hạng.

### Biểu thức (Expressions)

Một biểu thức có thể là: So sánh chuỗi, So sánh số, Toán tử tệp và Toán tử logic và nó được biểu diễn bởi `[biểu_thức]`:

#### So sánh số:

- `-eq` - bằng - `if [ "$a" -eq "$b" ]`
- `-ne` - không bằng - `if [ "$a" -ne "$b" ]`
- `-gt` - lớn hơn - `if [ "$a" -gt "$b" ]`
- `-ge` - lớn hơn hoặc bằng - `if [ "$a" -ge "$b" ]`
- `-lt` - nhỏ hơn - `if [ "$a" -lt "$b" ]`
- `-le` - nhỏ hơn hoặc bằng - `if [ "$a" -le "$b" ]`
- `<` - nhỏ hơn - `(("$a" < "$b"))`
- `<=` - nhỏ hơn hoặc bằng - `(("$a" <= "$b"))`
- `>` - lớn hơn - `(("$a" > "$b"))`
- `>=` - lớn hơn hoặc bằng - `(("$a" >= "$b"))`

#### Ví dụ:

- `[ n1 -eq n2 ]` (đúng nếu n1 giống n2, ngược lại sai)
- `[ n1 -ge n2 ]` (đúng nếu n1 lớn hơn hoặc bằng n2, ngược lại sai)
- `[ n1 -le n2 ]` (đúng nếu n1 nhỏ hơn hoặc bằng n2, ngược lại sai)
- `[ n1 -ne n2 ]` (đúng nếu n1 không giống n2, ngược lại sai)
- `[ n1 -gt n2 ]` (đúng nếu n1 lớn hơn n2, ngược lại sai)
- `[ n1 -lt n2 ]` (đúng nếu n1 nhỏ hơn n2, ngược lại sai)

#### So sánh chuỗi:

- `=` - bằng - `if [ "$a" = "$b" ]`
- `==` - bằng - `if [ "$a" == "$b" ]`
- `!=` - không bằng - `if [ "$a" != "$b" ]`
- `<` - nhỏ hơn, theo thứ tự bảng chữ cái ASCII - `if [[ "$a" < "$b" ]]`
- `>` - lớn hơn, theo thứ tự bảng chữ cái ASCII - `if [[ "$a" > "$b" ]]`
- `-z` - chuỗi rỗng, tức là có độ dài bằng không

#### Ví dụ:

- `[ s1 = s2 ]` (đúng nếu s1 giống s2, ngược lại sai)
- `[ s1 != s2 ]` (đúng nếu s1 không giống s2, ngược lại sai)
- `[ s1 ]` (đúng nếu s1 không rỗng, ngược lại sai)
- `[ -n s1 ]` (đúng nếu s1 có độ dài lớn hơn 0, ngược lại sai)
- `[ -z s2 ]` (đúng nếu s2 có độ dài bằng 0, ngược lại sai)

### Ví dụ Script

#### Script number.sh:

```
#!/bin/bash

echo -n “Enter a number 1 < x < 10: "

read num

if [ “$num” -lt 10 ]; then

      if [ “$num” -gt 1 ]; then

            echo “$num*$num=$(($num*$num))”

      else

            echo “Wrong insertion !”

      fi

else

      echo “Wrong insertion !”

fi
```

#### Script string.sh:

```
#! /bin/bash

word=a

if  [[ $word == "b" ]]
then
  echo "condition b is true"
elif [[ $word == "a" ]]
then 
  echo "condition a is true" 
else
  echo "condition is false"    
fi
```

---

## Phần 2: Kiểm tra tệp có tồn tại hay không

Trong phần này, chúng ta sẽ xem cách viết một bash shell script để kiểm tra xem tệp có tồn tại hay không.

### Cách thực hiện

Điều này có thể được thực hiện bằng mã shell script sau. Mã này sử dụng câu lệnh `if..else..fi` cho phép chúng ta đưa ra quyết định dựa trên thành công hay thất bại của một lệnh.

```
#! /bin/bash

echo -e "Enter the name of the file : \c"
read file_name

if [ -f $file_name ]
then
 else   
  echo "$file_name not exists"
fi
```

Trong shell script trên, chúng ta đã sử dụng Toán tử kiểm tra tệp. Biểu thức `-f $file_name` trong mã trên trả về True nếu tệp tồn tại và là một tệp thông thường.

### Danh sách các cờ khác trong Toán tử kiểm tra tệp

- `-a file` - Đúng nếu tệp tồn tại
- `-b file` - Đúng nếu tệp tồn tại và là một tệp đặc biệt khối
- `-c file` - Đúng nếu tệp tồn tại và là một tệp đặc biệt ký tự
- `-d file` - Đúng nếu tệp tồn tại và là một thư mục
- `-e file` - Đúng nếu tệp tồn tại
- `-f file` - Đúng nếu tệp tồn tại và là một tệp thông thường
- `-g file` - Đúng nếu tệp tồn tại và bit set-group-id của nó được đặt
- `-h file` - Đúng nếu tệp tồn tại và là một liên kết tượng trưng
- `-k file` - Đúng nếu tệp tồn tại và bit "sticky" của nó được đặt
- `-p file` - Đúng nếu tệp tồn tại và là một named pipe (FIFO)
- `-r file` - Đúng nếu tệp tồn tại và có thể đọc
- `-s file` - Đúng nếu tệp tồn tại và có kích thước lớn hơn không
- `-t fd` - Đúng nếu file descriptor fd đang mở và tham chiếu đến một terminal
- `-u file` - Đúng nếu tệp tồn tại và bit set-user-id của nó được đặt
- `-w file` - Đúng nếu tệp tồn tại và có thể ghi
- `-x file` - Đúng nếu tệp tồn tại và có thể thực thi
- `-G file` - Đúng nếu tệp tồn tại và thuộc sở hữu của effective group id
- `-L file` - Đúng nếu tệp tồn tại và là một liên kết tượng trưng
- `-N file` - Đúng nếu tệp tồn tại và đã được sửa đổi kể từ lần đọc cuối cùng
- `-O file` - Đúng nếu tệp tồn tại và thuộc sở hữu của effective user id
- `-S file` - Đúng nếu tệp tồn tại và là một socket

### So sánh tệp

- `file1 -ef file2` - Đúng nếu file1 và file2 tham chiếu đến cùng một device và inode numbers
- `file1 -nt file2` - Đúng nếu file1 mới hơn file2 (theo ngày sửa đổi), hoặc nếu file1 tồn tại và file2 không
- `file1 -ot file2` - Đúng nếu file1 cũ hơn file2, hoặc nếu file2 tồn tại và file1 không

### Kiểm tra biến và chuỗi

- `-o optname` - Đúng nếu tùy chọn shell optname được bật
- `-v varname` - Đúng nếu biến shell varname được đặt (đã được gán giá trị)
- `-R varname` - Đúng nếu biến shell varname được đặt và là một name reference
- `-z string` - Đúng nếu độ dài của chuỗi là không
- `-n string` - Đúng nếu độ dài của chuỗi là khác không
- `string1 == string2` hoặc `string1 = string2` - Đúng nếu các chuỗi bằng nhau
- `string1 != string2` - Đúng nếu các chuỗi không bằng nhau
- `string1 < string2` - Đúng nếu string1 sắp xếp trước string2 theo thứ tự từ điển
- `string1 > string2` - Đúng nếu string1 sắp xếp sau string2 theo thứ tự từ điển

### Toán tử số học

- `arg1 OP arg2` - Trong đó OP là một trong các toán tử: `-eq`, `-ne`, `-lt`, `-le`, `-gt`, hoặc `-ge`
- Các toán tử nhị phân số học này trả về true nếu arg1 bằng, không bằng, nhỏ hơn, nhỏ hơn hoặc bằng, lớn hơn, hoặc lớn hơn hoặc bằng arg2 tương ứng
- Arg1 và arg2 có thể là các số nguyên dương hoặc âm