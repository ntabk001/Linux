## Red Hat Certified System Administrator (RHEL-9 Version)

### User Account Management

* Commands
 
```
  • useradd

  • groupadd

  • userdel

  • groupdel

  • usermod
```

* Files

```
  • /etc/passwd
  
  • /etc/group
  
  • /etc/shadow
```

Example:

```
groupadd ams.service
useradd -g ams.service -s /bin/bash -c "User description" -m -d /home/ams.service ams.service
```

- Lệnh useradd được sử dụng để tạo một tài khoản người dùng mới trên hệ thống Linux. Dưới đây là giải thích chi tiết cho từng tham số trong lệnh bạn đã cung cấp:

  * `useradd`: Lệnh để tạo tài khoản người dùng mới.
  * `-g ams.service`: Chỉ định nhóm chính mà người dùng sẽ thuộc về. Trong trường hợp này, người dùng mới sẽ thuộc nhóm có tên là ams.service.
  * `-s /bin/bash`: Xác định shell mặc định cho người dùng. Ở đây, shell được chỉ định là `/bin/bash`, có nghĩa là người dùng sẽ sử dụng Bash shell khi đăng nhập.
  * `-c "User description"`: Thêm một mô tả cho tài khoản người dùng. Mô tả này thường được sử dụng để cung cấp thông tin thêm về người dùng.
  * `-m`: Tạo một thư mục chính cho người dùng mới. Thư mục này sẽ được tạo tại vị trí được chỉ định trong tham số -d.
  * `-d /home/ams.service`: Xác định đường dẫn đến thư mục chính của người dùng mới. Trong trường hợp này, thư mục chính sẽ là `/home/ams.service`.
  * `ams.service`: Tên của tài khoản người dùng được tạo.

* Lệnh chage được sử dụng để quản lý chính sách mật khẩu cho người dùng trên hệ thống Linux. Dưới đây là một số tính năng chính của lệnh này:

```
chage [-m mindays] [-M maxdays] [-d lastday] [-I inactive] [-E expiredate] [-W warndays] user
```

```
-d = 3. Last password change (lastchanged) : Days since Jan 1, 1970 that password was last changed
-m = 4. Minimum : The minimum number of days required between password changes i.e. the number of days left
before the user is allowed to change his/her password
-M = 5. Maximum : The maximum number of days the password is valid (after that user is forced to change his/her
password)
-W = 6. Warn : The number of days before password is to expire that user is warned that his/her password must be changed
-I = 7. Inactive : The number of days after password expires that account is disabled
-E = 8. Expire : days since Jan 1, 1970 that account is disabled i.e. an absolute date specifying when the login may no longer be used.
```

  * -M: Xác định số ngày tối đa giữa các lần thay đổi mật khẩu.
  * -m: Xác định số ngày tối thiểu trước khi người dùng có thể thay đổi mật khẩu.
  * -W: Xác định số ngày trước khi hết hạn mà người dùng sẽ nhận được thông báo.
  
* File = `/etc/login.def`
```
  • PASS_MAX_DAYS 99999

  • PASS_MIN_DAYS 0

  • PASS_MIN_LEN 5

  • PASS_WARN_AGE 7  
```  
  
* `nmcli` là công cụ dòng lệnh để quản lý `NetworkManager` trên hệ thống Linux. Dưới đây là tổng hợp các lệnh và chức năng chính của `nmcli`
  
  * List connections  `nmcli connection show`
  * Create a new connection `nmcli connection add type <type> con-name <connection-name> ifname <interface-name>`
  * Delete a connection `nmcli connection delete <connection-name>`
  * Edit a connection `nmcli connection edit <connection-name>`
  * List network devices `nmcli device`
  * Activate a device `nmcli device connect <device-name>`
  * Check NetworkManager status `nmcli general status`
  * Restart NetworkManager `nmcli networking off && nmcli networking on`
  
Here's an example of how to use nmcli to configure a static IP address for a network connection on a Linux system
  
```
nmcli connection show
nmcli connection modify ens160 ipv4.addresses 192.168.30.101/24
nmcli connection modify ens160 ipv4.gateway 192.168.0.2
nmcli connection modify ens160 ipv4.dns 8.8.8.8,8.8.4.4
nmcli connection modify ens160 ipv4.method manual

# After configuring the static IP, bring the connection up:
nmcli connection up MyConnection
nmcli connection reload

OR

sudo nmcli connection down eth0
sudo nmcli connection up eth0
```

```
cat /etc/NetworkManager/system-connections/ens160.nmconnection
[connection]
id=ens160
uuid=05b308e2-260e-3013-a4f1-b4f71b9bcac5
type=ethernet
autoconnect-priority=-999
interface-name=ens160
timestamp=1746929979

[ethernet]

[ipv4]
address1=192.168.30.101/24,192.168.30.2
dns=8.8.8.8;8.8.4.4;
method=manual

[ipv6]
addr-gen-mode=eui64
method=auto

[proxy]
```

* Summary of the Commands
  * Set static IP: ipv4.addresses
  * Set gateway: ipv4.gateway
  * Set DNS servers: ipv4.dns
  * Set method to manual: ipv4.method manual

```
sudo nmcli connection down eth0
sudo nmcli connection up eth0
or
sudo systemctl restart NetworkManager
nmcli device status
journalctl -u NetworkManager
sudo dhclient -r <interface-name>
sudo dhclient <interface-name>
```

* Remove Existing Connection and Recreate

```
sudo nmcli connection delete eth0
sudo nmcli connection add type ethernet con-name eth0 ifname eth0 ipv4.addresses 192.168.1.100/24 ipv4.gateway 192.168.1.1 ipv4.dns 8.8.8.8,8.8.4.4 ipv4.method manual
cat /etc/sysconfig/network-scripts/ifcfg-eth0
sudo systemctl stop network
sudo systemctl disable network
```

* `tcpdump` là một công cụ dòng lệnh mạnh mẽ để phân tích và chẩn đoán lưu lượng mạng. Dưới đây là tổng hợp các tính năng và cách sử dụng cơ bản của `tcpdump`.

  * Capture all traffic on the eth0 interface `sudo tcpdump -i eth0`
  * Capture traffic from a specific IP address `sudo tcpdump -i eth0 src 192.168.1.10`
  * Save packets to a file for later analysis `sudo tcpdump -i eth0 -w output.pcap`
  * To read a saved file `tcpdump -r output.pcap`
  * To display more detailed information about packets `sudo tcpdump -i eth0 -vv`

### Manage Local Users and Groups  
  
* Switch Users and sudo Access
```
  • su – username
  • sudo command
  • visudo
```
* File

  • `/etc/sudoers` 
  
### Control Access to Files

* There are 3 type of permissions
```  
  • r - read
  • w - write
  • x - exeawke = running a program
``` 
• Each permission (rwx) can be controlled at three levels:
```  
  • u - user = yourself
  • g - group = can be people in the same project
  • o - other = everyone on the system
```  
• File or Directory permission can be displayed by running `ls –l command`
```  
  • -rwxrwxrwx
```
### Maintaining Accurate Time


• Command to show system `time/date`

	* date

• Command for `time/date` and NTP setting

	* timedatectl

• To get help

	* timedatectl --help

• To view list of time zones

	* timedatectl list-timezones

• To set a time zone

	* timedatectl set-timezone America/New_York

• To set time or to set time and date

	* timedatectl set-time HH:MM:SS

	* timedatectl set-time ‘2021-08-18 20:15:50`

* To enable NTP synchronization

	* timedatectl set-ntp true

### Controlling access to files with ACL

* What is ACL?

  * Access control list (ACL) provides an additional, more flexible permission mechanism for file systems. It is designed to assist with UNIX file permissions. ACL allows you to give permissions for any user or group to any disc resource.

* Use of ACL :

  * Think of a scenario in which a particular user is not a member of group created by you but still you want to give some read or write access, how can you do it without making user a member of group, here comes in picture Access Control Lists, ACL helps us to do this trick

  * Basically, ACLs are used to make a flexible permission mechanism in Linux.

  * From Linux man pages, ACLs are used to define more fine-grained discretionary access rights for files and directories.

  * Commands to assign and remove ACL permissions are: `setfacl` and `getfacl`


### Access Control List (ACL)

* List of commands for setting up ACL 

1) To add permission for user

```
setfacl -m u:user:rwx /path/to/file
```

2) To add permissions for a group

```
setfacl -m g:group:rw /path/to/file
```

3) To allow all files or directories to inherit ACL entries from the directory it is within

```
setfacl -dm "entry" /path/to/dir
```

4) To remove a specific entry

```
setfacl -x u:user /path/to/file (For a specific user)
```

5) To remove all entries

```
setfacl -b path/to/file (For all users)
```

Note:

  • As you assign the ACL permission to a file/directory it adds + sign at the end of the permission
  
  • Setting w permission with ACL does not allow to remove a file

```
firewall-cmd --list-all
firewall-cmd --list-services
```

### Introducing SELinux

SELinux implements additional layer of security to protect system resources by implementing MAC –Mandatory Access Control.

What is Mandatory Access Control ?

Before we talk about this, we will understand well known access control on Linux Systems known as DAC- Discretionary Access Control.

Discretionary Access Control (DAC) defines the basic access controls for objects (file, directory, device, etc.) in a filesystem. This type of access is provided by owner of Objects to different users or groups based on their identity. Such access is generally at the discretion of the owner of the object (file, directory, device, etc.), so the name Discretionary Access Control.

Now about Mandatory Access Control , MAC is based on the security features of the entity and implements access control mainly between different processes and system resources (filesystem entities). For example , An Apache server process httpd will be allowed access to files or directories on path `/var/www/html/`

### How is this achieved ?

To implement MAC, Every process and system resource is assigned a special security label called an SELinux context. So, a process with specific SELinux context will be allowed access to specific system resources on the basis on SELinux context set on them and this access is defined by SELinux policy which gets loaded on system boot.

SELinux contexts have four fields: user, role, type, and security level. The SELinux type information is the most important when it comes to the SELinux policy, as the most common policy rule which defines the allowed interactions between processes and system resources uses SELinux types and not the full SELinux context. SELinux type ends with _t .

For example, the type name for the web server is httpd_t. The type context for files and directories found in `/var/www/html/` is `httpd_sys_content_t` .

There is a policy rule that permits Apache (the web server process running with context type httpd_t) to access files and directories found in `/var/www/html/` and other web server directories with a context (`httpd_sys_content_t`).

So, if Apache webserver gets compromised, Access to other system resources will not be possible .Access will be restricted to only web server directories .

<img src="/img/selinux-intro-apache-mariadb.png">

To list SELinux Context ,execute:

```
[root@rhcsa ~]# ls -ldZ /var/www/html/
drwxr-xr-x. 2 root root system_u:object_r:httpd_sys_content_t:s0 6 Aug 12  2024 /var/www/html/
```

```
[root@rhcsa ~]# ls -lZ anaconda-ks.cfg
-rw-------. 1 root root system_u:object_r:admin_home_t:s0 938 May 10 18:41 anaconda-ks.cfg
```

```
[root@rhcsa ~]# ps -efZ | grep httpd
system_u:system_r:httpd_t:s0    root        2044       1  9 08:12 ?        00:00:00 /usr/sbin/httpd -DFOREGROUND
system_u:system_r:httpd_t:s0    apache      2051    2044  0 08:12 ?        00:00:00 /usr/sbin/httpd -DFOREGROUND
system_u:system_r:httpd_t:s0    apache      2052    2044  0 08:12 ?        00:00:00 /usr/sbin/httpd -DFOREGROUND
system_u:system_r:httpd_t:s0    apache      2053    2044 13 08:12 ?        00:00:00 /usr/sbin/httpd -DFOREGROUND
system_u:system_r:httpd_t:s0    apache      2054    2044 14 08:12 ?        00:00:00 /usr/sbin/httpd -DFOREGROUND
unconfined_u:unconfined_r:unconfined_t:s0-s0:c0.c1023 root 2237 1041  0 08:12 pts/0 00:00:00 grep --color=auto httpd
```

So, SELinux context has 4 fields: user, role, type, and security level but as already said type is most interest of us.

### SELinux modes

* SELinux can run in one of three modes: enforcing, permissive, or disabled.

  • Enforcing mode is the default, and recommended, mode of operation; in enforcing mode SELinux operates normally, enforcing the loaded security policy on the entire system.
  
  • In permissive mode, the system acts as if SELinux is enforcing the loaded security policy, including labeling objects and emitting access denial entries in the logs, but it does not actually deny any operations. While not recommended for production systems, permissive mode can be helpful for SELinux policy development and debugging.
  
  • Disabled mode is strongly discouraged; not only does the system avoid enforcing the SELinux policy, but it also avoids labeling any persistent objects such as files, making it difficult to enable SELinux in the future.

To check state or mode of SELinux ; Execute command `getenforce` or `sestatus`

To change the SELinux mode to enforcing (Runtime), Execute , `setenforce 1`

To change the SELinux mode to permissive(Runtime), Execute, `setenforce 0`

But these changes do not persist after reboot, so this command should only be used for testing purposes. For persistent changes , required mode is set in SELinux config file (`/etc/selinux/config`)  
  
```
semanage port -l
semanage boolean -l
sesearch --all
seinfo --type
sudo chown -R test:test /etc/redis
sudo chmod -R u+rw /etc/redis
sudo semanage fcontext -a -t redis_config_t "/etc/redis(/.*)?"
sudo restorecon -R /etc/redis
```

### Create repo offline with DVD for Rhel 9

```
Mount the RHEL Binary DVD ISO to a directory such as /mnt, e.g.:

mkdir -p  /mnt
mount -o loop rhel-baseos-9.0-x86_64-dvd.iso /mnt

mount /dev/sr0  /mnt
* Note: The Warning mount: /mnt/disc: WARNING: source write-protected, mounted read-only. is expected.

cat <<\EOF > /etc/yum.repos.d/rhel9dvd.repo 
[BaseOS]
name=BaseOS Packages Red Hat Enterprise Linux 9
metadata_expire=-1
gpgcheck=1
enabled=1
baseurl=file:///mnt/BaseOS/
gpgkey=file:///etc/pki/rpm-gpg/RPM-GPG-KEY-redhat-release

[AppStream]
name=AppStream Packages Red Hat Enterprise Linux 9
metadata_expire=-1
gpgcheck=1
enabled=1
baseurl=file:///mnt/AppStream/
gpgkey=file:///etc/pki/rpm-gpg/RPM-GPG-KEY-redhat-release
EOF

- Clear the cache and check whether you are able to get the packages from this DVD repository:

yum clean all
yum repolist

```












