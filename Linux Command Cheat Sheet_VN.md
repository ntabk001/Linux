<h1 id="_cac-lenh-ve-system-0">CÁC LỆNH VỀ SYSTEM</h1>
<table>
<thead>
<tr>
<th>Command</th>
<th>Description</th>
</tr>
</thead>
<tbody>
<tr>
<td><code>uname -a</code></td>
<td>Hiển thị thông tin về hệ thống Linux</td>
</tr>
<tr>
<td><code>uname -r</code></td>
<td>Hiển thị thông tin về  kernel</td>
</tr>
<tr>
<td><code>cat /etc/os-release</code></td>
<td>Hiển thị thông tin hệ điều hành như tên và phiên bản</td>
</tr>
<tr>
<td><code>uptime</code></td>
<td>Hiển thị thời gian hệ thống đã chạy</td>
</tr>
<tr>
<td><code>hostname</code></td>
<td>Hiển thị tên máy chủ</td>
</tr>
<tr>
<td><code>hostname -I</code></td>
<td>Hiển thị tất cả các địa chỉ IP trên local</td>
</tr>
<tr>
<td><code>last reboot</code></td>
<td>Hiển thị lịch sử reboot</td>
</tr>
<tr>
<td><code>date</code></td>
<td>Hiển thị ngày giờ hiện tại</td>
</tr>
<tr>
<td><code>cal</code></td>
<td>Hiển thị lịch tháng hiện tại</td>
</tr>
<tr>
<td><code>whoami</code></td>
<td>Hiển thị tên đăng nhập hiện tại của bạn</td>
</tr>
<tr>
<td><code>history</code></td>
<td>Hiển thị danh sách tất cả các lệnh đã sử dụng</td>
</tr>
<tr>
<td><code>clear</code></td>
<td>Xóa terminal</td>
</tr>
<tr>
<td><code>shutdown -h now</code></td>
<td>Tắt hệ thống</td>
</tr>
<tr>
<td><code>reboot</code></td>
<td>Khởi động lại hệ thống</td>
</tr>
</tbody>
</table>
<h1 id="_cac-lenh-ve-hardware-1">CÁC LỆNH VỀ HARDWARE</h1>
<table>
<thead>
<tr>
<th>Command</th>
<th>Description</th>
</tr>
</thead>
<tbody>
<tr>
<td><code>cat /proc/cpuinfo</code></td>
<td>Hiển thị thông tin CPU</td>
</tr>
<tr>
<td><code>cat /proc/meminfo</code></td>
<td>Hiển thị thông tin memory</td>
</tr>
<tr>
<td><code>free -h</code></td>
<td>Hiển thị bộ nhớ còn trống và đã sử dụng [ -h (human), -m (MB), -g (GB)  ]</td>
</tr>
<tr>
<td><code>lspci -tv</code></td>
<td>Hiển thị các  thiết bị PCI</td>
</tr>
<tr>
<td><code>lsusb -tv</code></td>
<td>Hiển thị các thiết bị USB</td>
</tr>
<tr>
<td><code>lsblk</code></td>
<td>Hiển thị thông tin về  block devices</td>
</tr>
<tr>
<td><code>dmidecode</code></td>
<td>Hiển thị DMI / SMBIOS (thông tin phần cứng) từ BIOS</td>
</tr>
<tr>
<td><code>hdparm -i /dev/sda</code></td>
<td>Hiển thị thông tin về đĩa sda</td>
</tr>
<tr>
<td><code>hdparm -tT /dev/sda</code></td>
<td>Thực hiện kiểm tra tốc độ đọc trên đĩa sda</td>
</tr>
<tr>
<td><code>badblocks -s /dev/sda</code></td>
<td>Kiểm tra các khối không đọc được trên đĩa sda</td>
</tr>
</tbody>
</table>
<h1 id="_cac-lenh-ve-monitoring-2">CÁC LỆNH VỀ MONITORING</h1>
<table>
<thead>
<tr>
<th>Command</th>
<th>Description</th>
</tr>
</thead>
<tbody>
<tr>
<td><code>mpstat 1</code></td>
<td>Hiển thị thống kê liên quan đến bộ xử lý</td>
</tr>
<tr>
<td><code>vmstat 1</code></td>
<td>Hiển thị thống kê bộ nhớ ảo</td>
</tr>
<tr>
<td><code>iostat 1</code></td>
<td>Hiển thị thống kê I/O</td>
</tr>
<tr>
<td><code>tail -100 /var/log/messages</code></td>
<td>Hiển thị 100 log gần nhất của hệ thống</td>
</tr>
<tr>
<td><code>tcpdump -i eth0</code></td>
<td>Capture  và hiển thị tất cả các packets trên interface eth0</td>
</tr>
<tr>
<td><code>tcpdump -i eth0 'port 80'</code></td>
<td>Monitor tất cả lưu lượng truy cập trên port 80 ( HTTP )</td>
</tr>
<tr>
<td><code>lsof</code></td>
<td>Liệt kê tất cả các tệp đang mở trên hệ thống</td>
</tr>
<tr>
<td><code>lsof -u user</code></td>
<td>Liệt kê các tệp được mở bởi người dùng</td>
</tr>
<tr>
<td><code>watch df -h</code></td>
<td>Thực thi và "<strong>watch</strong>" câu lệnh <code>df -h</code></td>
</tr>
</tbody>
</table>
<h1 id="_cac-lenh-ve-user-3">CÁC LỆNH VỀ USER</h1>
<table>
<thead>
<tr>
<th>Command</th>
<th>Description</th>
</tr>
</thead>
<tbody>
<tr>
<td><code>id</code></td>
<td>Hiển thị id người dùng và nhóm hiện tại của người dùng</td>
</tr>
<tr>
<td><code>last</code></td>
<td>Hiển thị người dùng gần nhất đã đăng nhập vào hệ thống</td>
</tr>
<tr>
<td><code>who</code></td>
<td>Hiển thị ai đã đăng nhập vào hệ thống</td>
</tr>
<tr>
<td><code>w</code></td>
<td>Hiển thị ai đã đăng nhập và họ đang làm gì</td>
</tr>
<tr>
<td><code>groupadd test</code></td>
<td>Tạo một nhóm có tên là "test"</td>
</tr>
<tr>
<td><code>groupdel test</code></td>
<td>Xóa một nhóm có tên là "test"</td>
</tr>
<tr>
<td><code>useradd -c "This is NhaX" -m nhax</code></td>
<td>Tạo tài khoản có tên nhax, với comment "This is NhaX" và tạo thư mục chính cho người dùng</td>
</tr>
<tr>
<td><code>userdel nhax</code></td>
<td>Xóa tài khoản nhax</td>
</tr>
<tr>
<td><code>usermod -aG sales nhax</code></td>
<td>Thêm tài khoản nhax vào nhóm sales</td>
</tr>
<tr>
<td><code>usermod [option] username</code></td>
<td>Thay đổi thông tin tài khoản người dùng bao gồm: nhóm, thư mục chính, shell, ngày hết hạn</td>
</tr>
</tbody>
</table>
<h1 id="_cac-lenh-ve-file-directory-4">CÁC LỆNH VỀ FILE, DIRECTORY</h1>
<table>
<thead>
<tr>
<th>Command</th>
<th>Description</th>
</tr>
</thead>
<tbody>
<tr>
<td><code>pwd</code></td>
<td>Hiển thị folder làm việc hiện tại</td>
</tr>
<tr>
<td><code>cd</code></td>
<td>Thay đổi folder hiện tại thành folder chính</td>
</tr>
<tr>
<td><code>cd foldername</code></td>
<td>Thay đổi folder thành folder có tên "foldername"</td>
</tr>
<tr>
<td><code>cd ..</code></td>
<td>Thay đổi folder lên một cấp</td>
</tr>
<tr>
<td><code>ls</code></td>
<td>Liệt kê tất cả các file và folder trong folder hiện tại</td>
</tr>
<tr>
<td><code>ls -al</code></td>
<td>Liệt kê tất cả các file và folder bao gồm: tệp ẩn và các thông tin khác như quyền, kích thước và chủ sở hữu</td>
</tr>
<tr>
<td><code>mkdir directory</code></td>
<td>Tạo folder tên "directory"</td>
</tr>
<tr>
<td><code>rm file</code></td>
<td>Xóa file</td>
</tr>
<tr>
<td><code>rm -r directory</code></td>
<td>Xóa folder và nội dung của nó theo cách đệ quy</td>
</tr>
<tr>
<td><code>rm -f file</code></td>
<td>Buộc xóa file mà không cần xác nhận</td>
</tr>
<tr>
<td><code>rm -rf directory</code></td>
<td>Buộc xóa folder theo cách đệ quy</td>
</tr>
<tr>
<td><code>cp file1 file2</code></td>
<td>Copy file1 sang file2</td>
</tr>
<tr>
<td><code>cp -r source_directory destination</code></td>
<td>Sao chép folder source_directory đến folder destination. Nếu source_directory tồn tại, sao chép source_directory vào destination, nếu không thì tạo destination với nội dung của source_directory.</td>
</tr>
<tr>
<td><code>mv file1 file2</code></td>
<td>Đổi tên hoặc di chuyển file1 thành file2</td>
</tr>
<tr>
<td><code>ln -s /path/to/file linkname</code></td>
<td>Tạo symbolic link đến linkname</td>
</tr>
<tr>
<td><code>touch file</code></td>
<td>Tạo một file trống (hoặc cập nhật thời gian truy cập và sửa đổi file)</td>
</tr>
<tr>
<td><code>cat file</code></td>
<td>Xem nội dung của file</td>
</tr>
<tr>
<td><code>cat file1 file2 &gt; file3</code></td>
<td>Kết hợp hai file có tên là file1 và file2 và lưu trữ đầu ra trong file3</td>
</tr>
<tr>
<td><code>less file</code></td>
<td>Duyệt qua một file</td>
</tr>
<tr>
<td><code>head file</code></td>
<td>Hiển thị 10 dòng đầu tiên của file</td>
</tr>
<tr>
<td><code>tail file</code></td>
<td>Hiển thị 10 dòng cuối cùng của file</td>
</tr>
<tr>
<td><code>tail -f file</code></td>
<td>Hiển thị 10 dòng cuối cùng của file và "watch" file khi tệp thay đổi.</td>
</tr>
</tbody>
</table>
<h1 id="_cac-lenh-ve-process-5">CÁC LỆNH VỀ PROCESS</h1>
<table>
<thead>
<tr>
<th>Command</th>
<th>Description</th>
</tr>
</thead>
<tbody>
<tr>
<td><code>ps</code></td>
<td>Hiển thị các process hiện đang chạy</td>
</tr>
<tr>
<td><code>pstree</code></td>
<td>Hiển thị các process trong sơ đồ dạng cây</td>
</tr>
<tr>
<td><code>ps -ef</code></td>
<td>Hiển thị tất cả các process hiện đang chạy trên hệ thống.</td>
</tr>
<tr>
<td><code>ps -ef | grep processname</code></td>
<td>Hiển thị process có tên "processname"</td>
</tr>
<tr>
<td><code>top</code></td>
<td>Hiển thị các top process</td>
</tr>
<tr>
<td><code>htop</code></td>
<td>Tương tự <code>top</code>, nhưng trực quan hơn</td>
</tr>
<tr>
<td><code>kill pid</code></td>
<td>Kill process có process ID là pid</td>
</tr>
<tr>
<td><code>killall processname</code></td>
<td>Kill tất cả các process có tên "processname"</td>
</tr>
<tr>
<td><code>program &amp;</code></td>
<td>Chạy program ở background</td>
</tr>
<tr>
<td><code>bg</code></td>
<td>Hiển thị  các jobs đã dừng hoặc chạy ngầm</td>
</tr>
<tr>
<td><code>fg</code></td>
<td>Đưa các background jobs  gần nhất lên foreground</td>
</tr>
<tr>
<td><code>fg n</code></td>
<td>Đưa jobs n lên foreground</td>
</tr>
<tr>
<td><code>lsof</code></td>
<td>Liệt kê tất cả các tệp được mở bằng các process đang chạy</td>
</tr>
<tr>
<td><code>pidof processname</code></td>
<td>Lấy PID của bất kỳ process nào</td>
</tr>
</tbody>
</table>
<h1 id="_cac-lenh-ve-file-permissions-6">CÁC LỆNH VỀ FILE PERMISSIONS</h1>
<img src="/img/per.PNG"
<em>(Execute = 1, Write = 2, Read = 4)</em></p>
<table>
<thead>
<tr>
<th>Command</th>
<th>Description</th>
</tr>
</thead>
<tbody>
<tr>
<td><code>ls -l filename</code></td>
<td>Kiểm tra permission hiện tại của file</td>
</tr>
<tr>
<td><code>chmod 777 filename</code></td>
<td>Gán quyền đầy đủ (đọc, viết và thực thi) cho mọi người</td>
</tr>
<tr>
<td><code>chmod -R 777 dirname</code></td>
<td>Gán quyền đầy đủ cho folder và tất cả các folder con</td>
</tr>
<tr>
<td><code>chmod -x filename</code></td>
<td>Xóa quyền thực thi của bất file nào</td>
</tr>
<tr>
<td><code>chown username filename</code></td>
<td>Thay đổi quyền sở hữu của một file</td>
</tr>
<tr>
<td><code>chown user:group filename</code></td>
<td>Thay đổi chủ sở hữu và quyền sở hữu nhóm của file</td>
</tr>
<tr>
<td><code>chown -R user:group dirname</code></td>
<td>Thay đổi chủ sở hữu và quyền sở hữu nhóm của folder và tất cả các folder con</td>
</tr>
</tbody>
</table>
<h1 id="_cac-lenh-ve-networking-7">CÁC LỆNH VỀ NETWORKING</h1>
<table>
<thead>
<tr>
<th>Command</th>
<th>Description</th>
</tr>
</thead>
<tbody>
<tr>
<td><code>ip a</code></td>
<td>Hiển thị network interfaces và địa chỉ IP</td>
</tr>
<tr>
<td><code>ip addr show dev eth0</code></td>
<td>Hiển thị địa chỉ eth0 và thông tin chi tiết</td>
</tr>
<tr>
<td><code>ip addr add IP-Address dev eth1</code></td>
<td>Thêm địa chỉ IP tạm thời vào interface eth1</td>
</tr>
<tr>
<td><code>ethtool eth0</code></td>
<td>Truy vấn hoặc kiểm soát cài đặt phần cứng và network driver</td>
</tr>
<tr>
<td><code>ping host</code></td>
<td>Gửi ICMP echo request tới host</td>
</tr>
<tr>
<td><code>whois domain</code></td>
<td>Hiển thị thông tin whois cho domain</td>
</tr>
<tr>
<td><code>dig domain</code></td>
<td>Hiển thị thông tin DNS cho domain</td>
</tr>
<tr>
<td><code>dig -x IP_ADDRESS</code></td>
<td>Tra cứu ngược IP_ADDRESS</td>
</tr>
<tr>
<td><code>host domain</code></td>
<td>Hiển thị địa chỉ IP DNS cho domain</td>
</tr>
<tr>
<td><code>hostname -i</code></td>
<td>Hiển thị địa chỉ mạng của hostname.</td>
</tr>
<tr>
<td><code>hostname -I</code></td>
<td>Hiển thị tất cả các địa chỉ IP cục bộ của máy chủ.</td>
</tr>
<tr>
<td><code>wget http://domain.com/file</code></td>
<td>Tải xuống <a href="http://domain.com/file" target="_blank">http://domain.com/file</a></td>
</tr>
<tr>
<td><code>netstat -nutlp</code></td>
<td>Hiển thị các cổng tcp và udp đang nghe và các chương trình tương ứng</td>
</tr>
</tbody>
</table>
<h1 id="_cac-lenh-ve-archives-tar-files-8">CÁC LỆNH VỀ ARCHIVES (TAR FILES)</h1>
<table>
<thead>
<tr>
<th>Command</th>
<th>Description</th>
</tr>
</thead>
<tbody>
<tr>
<td><code>tar -cvf filename.tar filename</code></td>
<td>Nén file thành Tar</td>
</tr>
<tr>
<td><code>tar -xvf filename.tar</code></td>
<td>Giải nén file Tar</td>
</tr>
<tr>
<td><code>tar -tvf filename.tar</code></td>
<td>Liệt kê nội dung của file Tar</td>
</tr>
<tr>
<td><code>tar -xvf filename.tar file1.txt</code></td>
<td>Gỡ bỏ một file duy nhất khỏi file Tar</td>
</tr>
<tr>
<td><code>tar -rvf filename.tar file2.txt</code></td>
<td>Thêm file vào Tar</td>
</tr>
<tr>
<td><code>zip filename.zip filename</code></td>
<td>Nén một file thành một file zip</td>
</tr>
<tr>
<td><code>zip filename.zip file1.txt file2.txt file3.txt</code></td>
<td>Nén nhiều file vào một zip</td>
</tr>
<tr>
<td><code>zip -u filename.zip file4.txt</code></td>
<td>Thêm file vào zip</td>
</tr>
<tr>
<td><code>zip -d filename.zip file4.txt</code></td>
<td>Xóa file khỏi zip</td>
</tr>
<tr>
<td><code>unzip -l filename.zip</code></td>
<td>Hiển thị nội dung của file lưu trữ zip</td>
</tr>
<tr>
<td><code>unzip filename.zip</code></td>
<td>Giải nén</td>
</tr>
<tr>
<td><code>unzip filename.zip -d /dirname</code></td>
<td>Giải nén file vào một thư mục cụ thể</td>
</tr>
</tbody>
</table>
<h1 id="_cac-lenh-ve-installing-packages-9">CÁC LỆNH VỀ INSTALLING PACKAGES</h1>
<table>
<thead>
<tr>
<th>Command</th>
<th>Description</th>
</tr>
</thead>
<tbody>
<tr>
<td><code>apt-get install packagename</code></td>
<td>Cài đặt gói trên các bản phân phối dựa trên Debian</td>
</tr>
<tr>
<td><code>apt-get remove packagename</code></td>
<td>Xóa gói trên các bản phân phối dựa trên Debian</td>
</tr>
<tr>
<td><code>dpkg -l | grep -i installed</code></td>
<td>Nhận danh sách tất cả các gói trên bản phân phối dựa trên Debian</td>
</tr>
<tr>
<td><code>dpkg -i packagename.deb</code></td>
<td>Cài đặt gói .deb</td>
</tr>
<tr>
<td><code>apt-get update</code></td>
<td>Cập nhật kho lưu trữ trên các bản phân phối dựa trên Debian</td>
</tr>
<tr>
<td><code>apt-get upgrade packagename</code></td>
<td>Nâng cấp một gói cụ thể trên các bản phân phối dựa trên Debian</td>
</tr>
<tr>
<td><code>apt-get autoremove</code></td>
<td>Xóa tất cả các gói không mong muốn trên các bản phân phối dựa trên Debian</td>
</tr>
<tr>
<td><code>yum install packagename</code></td>
<td>Cài đặt gói trên các bản phân phối dựa trên RPM</td>
</tr>
<tr>
<td><code>yum remove packagename</code></td>
<td>Xóa gói trên các bản phân phối dựa trên RPM</td>
</tr>
<tr>
<td><code>yum update</code></td>
<td>Cập nhật tất cả các gói hệ thống lên phiên bản mới nhất trên các bản phân phối dựa trên RPM</td>
</tr>
<tr>
<td><code>yum list --installed</code></td>
<td>Liệt kê tất cả các gói đã cài đặt trên các bản phân phối dựa trên RPM</td>
</tr>
<tr>
<td><code>yum list --available</code></td>
<td>Liệt kê tất cả các gói có sẵn trên các bản phân phối dựa trên RPM</td>
</tr>
</tbody>
</table>
<h1 id="_cac-lenh-ve-search-10">CÁC LỆNH VỀ SEARCH</h1>
<table>
<thead>
<tr>
<th>Command</th>
<th>Description</th>
</tr>
</thead>
<tbody>
<tr>
<td><code>grep pattern file</code></td>
<td>Tìm kiếm "pattern" trong file</td>
</tr>
<tr>
<td><code>grep -r pattern directory</code></td>
<td>Tìm kiếm đệ quy "pattern" trong thư mục</td>
</tr>
<tr>
<td><code>locate name</code></td>
<td>Tìm tệp và thư mục theo tên</td>
</tr>
<tr>
<td><code>find /home/john -name 'prefix*'</code></td>
<td>Tìm tệp trong /home/john bắt đầu bằng "prefix".</td>
</tr>
<tr>
<td><code>find /home -size +100M</code></td>
<td>Tìm tệp lớn hơn 100MB trong /home</td>
</tr>
</tbody>
</table>
<h1 id="_cac-lenh-ve-ssh-logins-11">CÁC LỆNH VỀ SSH LOGINS</h1>
<table>
<thead>
<tr>
<th>Command</th>
<th>Description</th>
</tr>
</thead>
<tbody>
<tr>
<td><code>ssh host</code></td>
<td>Kết nối với máy chủ dưới dạng tên người dùng cục bộ của bạn</td>
</tr>
<tr>
<td><code>ssh user@host</code></td>
<td>Kết nối với máy chủ với tư cách người dùng</td>
</tr>
<tr>
<td><code>ssh -p port user@host</code></td>
<td>Kết nối với máy chủ bằng cổng</td>
</tr>
</tbody>
</table>
<h1 id="_cac-lenh-ve-file-transfers-12">CÁC LỆNH VỀ FILE TRANSFERS</h1>
<table>
<thead>
<tr>
<th>Command</th>
<th>Description</th>
</tr>
</thead>
<tbody>
<tr>
<td><code>scp file.txt server:/tmp</code></td>
<td>Sao chép an toàn file.txt vào thư mục /tmp trên máy chủ</td>
</tr>
<tr>
<td><code>scp server:/var/www/*.html /tmp</code></td>
<td>Sao chép các tệp *.html từ máy chủ vào thư mục /tmp cục bộ.</td>
</tr>
<tr>
<td><code>scp -r server:/var/www /tmp</code></td>
<td>Sao chép đệ quy tất cả các tệp và thư mục từ máy chủ vào thư mục /tmp của hệ thống hiện tại.</td>
</tr>
<tr>
<td><code>rsync -a /home /backups/</code></td>
<td>Đồng bộ hóa /home với /backups/home</td>
</tr>
<tr>
<td><code>rsync -avz /home server:/backups/</code></td>
<td>Đồng bộ hóa các tệp/thư mục giữa hệ thống cục bộ và từ xa với chức năng nén</td>
</tr>
</tbody>
</table>
<h1 id="_cac-lenh-ve-disk-usage-13">CÁC LỆNH VỀ DISK USAGE</h1>
<table>
<thead>
<tr>
<th>Command</th>
<th>Description</th>
</tr>
</thead>
<tbody>
<tr>
<td><code>df -h</code></td>
<td>Hiển thị dung lượng trống và đã sử dụng trên mounted filesystems</td>
</tr>
<tr>
<td><code>df -i</code></td>
<td>Hiển thị các inode trống và đã sử dụng trên mounted filesystems</td>
</tr>
<tr>
<td><code>fdisk -l</code></td>
<td>Hiển thị kích thước và loại phân vùng đĩa</td>
</tr>
<tr>
<td><code>du -ah</code></td>
<td>Hiển thị mức sử dụng đĩa cho tất cả các tệp và thư mục ở định dạng con người có thể đọc được</td>
</tr>
<tr>
<td><code>du -sh</code></td>
<td>Hiển thị tổng mức sử dụng đĩa ngoài thư mục hiện tại</td>
</tr>
</tbody>
</table>
<h1 id="_cac-lenh-ve-security-14">CÁC LỆNH VỀ SECURITY</h1>
<table>
<thead>
<tr>
<th>Command</th>
<th>Description</th>
</tr>
</thead>
<tbody>
<tr>
<td><code>passwd</code></td>
<td>Thay đổi mật khẩu của người dùng hiện tại.</td>
</tr>
<tr>
<td><code>sudo -i</code></td>
<td>Chuyển sang tài khoản root với môi trường của root.</td>
</tr>
<tr>
<td><code>sudo -s</code></td>
<td>Thực thi shell hiện tại của bạn với quyền root</td>
</tr>
<tr>
<td><code>sudo -l</code></td>
<td>Liệt kê các đặc quyền sudo cho người dùng hiện tại.</td>
</tr>
<tr>
<td><code>visudo</code></td>
<td>Chỉnh sửa tệp cấu hình sudoers.</td>
</tr>
<tr>
<td><code>getenforce</code></td>
<td>Hiển thị chế độ SELinux hiện tại.</td>
</tr>
<tr>
<td><code>sestatus</code></td>
<td>Hiển thị chi tiết SELinux như chế độ SELinux hiện tại, chế độ được định cấu hình và chính sách đã tải.</td>
</tr>
<tr>
<td><code>setenforce 0</code></td>
<td>Thay đổi chế độ SELinux hiện tại thành Permissive. (Không tồn tại khi khởi động lại.)</td>
</tr>
<tr>
<td><code>setenforce 1</code></td>
<td>Thay đổi chế độ SELinux hiện tại thành Enforcing. (Không tồn tại khi khởi động lại.)</td>
</tr>
<tr>
<td><code>SELINUX=enforcing</code></td>
<td>Đặt chế độ SELinux để thực thi khi khởi động bằng cách sử dụng cài đặt này trong tệp /etc/selinux/config.</td>
</tr>
<tr>
<td><code>SELINUX=permissive</code></td>
<td>Đặt chế độ SELinux thành cho phép khi khởi động bằng cách sử dụng cài đặt này trong tệp /etc/selinux/config.</td>
</tr>
<tr>
<td><code>SELINUX=disabled</code></td>
<td>Đặt chế độ SELinux thành tắt khi khởi động bằng cách sử dụng cài đặt này trong tệp /etc/selinux/config.</td>
</tr>
</tbody>
</table>
<h1 id="_cac-lenh-ve-logging-and-auditing-15">CÁC LỆNH VỀ LOGGING AND AUDITING</h1>
<table>
<thead>
<tr>
<th>Command</th>
<th>Description</th>
</tr>
</thead>
<tbody>
<tr>
<td><code>dmesg</code></td>
<td>Hiển thị message trong kernel ring buffer</td>
</tr>
<tr>
<td><code>journalctl</code></td>
<td>Hiển thị nhật ký được lưu trữ trong systemd journal.</td>
</tr>
<tr>
<td><code>journalctl -u servicename</code></td>
<td>Hiển thị nhật ký cho một đơn vị servicename cụ thể.</td>
</tr>
</tbody>
</table>