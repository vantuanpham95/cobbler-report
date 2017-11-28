# cobbler-report

## Table of Contents

 * Cấu trúc và các thành phần chính 
 * Cài đặt và cấu hình Cobbler 
 * Cài đặt và cấu hình webadmin cho Cobbler 
 * Cài đặt và cấu hình dhcp 
 * Thao tác quản lý cobbler 
 * import file iso (create distro & profile) 
 * Chỉnh sửa Profile
 * Tạo system 
 * Thao tác list, xóa, edit, rename, show 
 * Quản lý power on, power off, reboot machine (chức năng này chưa test)
 * Lưu ý quan trọng

## 1. Cấu trúc và các thành phần chính

Cobbler kết nối và tự động hóa nhiều công đoạn khác nhau trong quá trình cài đặt Linux, giúp cho người quản trị dễ dàng hơn trong việc cài đặt số lượng lớn hệ điều hành Linux với những cấu hình khác nhau. Cobbler quản lý những thành phần chính sau :

 * Kickstart file : đây là thành phần quan trọng nhất, kickstart file là file chứa tất cả những câu trả lời cần thiết cho việc cài đặt các distro Linux nhờ vào file trả lời này mà toàn bộ quá trình cài đặt sẽ được tự hóa hoàn toàn.
 * TFTP, Web, FTP, rsync server : đây là các giao thức mà Cobbler sử dụng để truyển tải các file cài đặt từ server đến các máy trạm để cài linux.
 * DHCP server : để phục vụ cho việc cài đặt qua mạng, máy trạm phải kết nối được đến server và được cấp 1 địa chỉ IP. Quá trình cấp địa chỉ này được thực hiện bởi DHCP server
 * DNS server : để tiện cho việc quản lý, ta có thể gán địa chỉ IP với 1 tên miền dịch vụ này không phải yêu cầu bắt buộc như DHCP.
 * Web server : Cobbler cung cấp giao diện web cho phép người quản trị thông qua đó, quản lý các profile cũng như các máy trạm được cài đặt.

**Các khái niệm cơ bản cần biết của Cobbler**

 * Distribution : chứa các thông tin về kernel và initrd nào được sử dụng, bao gồm cả các dữ liệu dùng để cài đặt. Hiểu 1 cách đơn giản. Đấy chính là đĩa cài đặt của Linux của ta.
 * Profile = Distribution + kickstart file + các gói cài đặt thêm (nếu cần)
 * System = Profile + MAC addr (IP addr, hostname)
 * Repo : là nơi chứa các gói cài đặt thêm.

Như vậy, có thể thấy, trình tự cấu hình Cobbler sẽ như sau :

 * Tạo distribution
 * Tạo profle
 * Tạo repo.
 * Add system.
 * Boot máy trạm và chờ kết quả
 
## 2. Cài đặt và cấu hình Cobbler
```
root@cobble#apt-get install cobbler cobbler-common
```
Setting một các thông số bên dưới trong file cobbler settings :
```
root@cobbler#vi /etc/cobbler/settings

next_server: 172.30.1.100 #chỉ định tftp server.
Server: 172.30.1.100 # chỉ định cobbler server
manage_dhcp: 1 # set 1 - nếu muốn cobbler quản lý dịch vụ dhcp
```
Restart cobbler service:
```
/etc/init.d/cobbler restart
```
Check cấu hình:
```
root@cobbler#cobbler check
```
Đồng bộ cấu hình (apply các thay đổi):
```
root@cobbler#cobbler sync
```
Khi thực hiện lệnh này, cobbler sẽ update nội dung toàn bộ file trong thư viện tftp, các file cấu hình
boot pxe, update kernel boot của toàn bộ các profile được tạo, update cấu hình dhcp (nếu manage_dhcp
được set bằng 1 trong cobbler settings) dựa trên systems được set cho từng node và đồng thời restart
dịch vụ dhcp).

## 3. Cài đặt và cấu hình webadmin cho Cobbler

Cobbler cung cấp 1 giao diện web, có thể thao tác qua giao diện web này mà không cần gõ
commandline,
```
root@cobble#apt-get install cobbler-web
```
Sau lệnh cài đặt cobbler-web, system sẽ tạo ra các file cấu hình apache (đặt tại /etc/apache2/conf.d/)

Setting password quản trị web:
```
root@cobbler#htdigest /etc/cobbler/users.digest "FullName" username
```
**Cài đặt và cấu hình dhcp**

Cài đặt dhcp:
```
root@cobble#apt-get instal isc-dhcp-server
```

Chỉ định interface mà dhcp listenting:
```
#vi /etc/default/isc-dhcp-server
INTERFACES="enp0s8"
```

Chỉnh sửa dhcp template trong cobbler:
```
root@cobble#vi /etc/cobbler/dhcp.template
```

Cobbler sẽ dùng template này để tạo cấu hình cho dhcp, các thông số MACADDRESS, IP, gateway chỉ
định trong cobbler'system khi “cobbler sync” cũng được import vào file cấu hình dhcp.

## 4. Thao tác quản lý cobbler
### a. Import file iso (create distro & profile)

Cobbler sẽ import nội dung file iso, vì vậy cần bung nén hoặc mount trước khi import

Nội dung của file sẽ được sync vào thư mục web: /var/www/cobbler/ks_mirror

Khi import, Cobbler sẽ nhận dạng:
 
 * Kernel boot của iso vừa import (Initrd, Kernel)
 * Kiểu Architecture, OS version (vd là precise hay raring),
 
và tạo distro, profile theo name của iso, (distro có thể hiểu như image (iso))

Profile trong cobbler được tạo dựa trên distro
```
root@cobbler#mkdir /mnt/iso
root@cobbler#mount -o loop ubuntu-13.04-server-amd64.iso /mnt/iso
root@cobbler#cobbler import --name=ubuntu-raring –path=/mnt/iso
```
Sau lệnh import trên, name của distro, profile sẽ có cùng tên là raring-x86_64

Trong trường hợp file iso import, Cobbler không nhận dạn được các thông số trên 

==> Cobbler sẽ ko tạo tự động distro và profile, và ta sẽ phải tạo distro, profile bằng tay. Khi tạo distro sẽ cần setting các thông số về kernel boot (initrd, kernel)

### b. Chỉnh sửa Profile

Trong cobbler, gọi các file preseed, kickstart chung là kickstart file, đặt tại /var/lib/cobbler/kickstarts/

=> cần phải cấu hình file kickstart cho mỗi profile,

Mặc định profile ubuntu sẽ dùng file kickstart “sample.seed”, ta cần tạo mới 1 file kickstart khác và trỏ profile được tạo tự động vừa rồi vào file kickstart này bằng thao tác edit profile:
```
cobbler profile edit --name=raring-x86_64 --kickstart=/var/lib/cobbler/kickstarts/raring.preseed
```
### c. Tạo system

Mục đích để tự động assign 1 machine cài đặt profile cụ thể thông qua MACADDRESS của machine đó. Đồng thời có thể cấu hình hostname, IP, gateway,,.. để khi cài sẽ fix sẵn thông tin (nếu muốn).

Nếu ko set system, khi boot,ta sẽ phải chọn trong profile trong menu profile.
```
root@cobbler#cobbler system add --name=machine_01 --profile= raring-x86_64 \
--mac=00:00:00:00:00:00 --ip-address=172.30.1.10 \
--hostname=machine01.vccloud --name-servers=8.8.8.8 \
--gateway= 172.30.1.10 –static=1
cobbler system add --name=softraid_02 --profile=ubuntu-trusty-x86_64 –
mac=08:00:27:47:65:44 --ip-address=172.30.1.103 --hostname=softraid_02 --nameservers=8.8.8.8
--gateway= 172.30.1.78 --static=1
```
### d. Thao tác list, xóa, edit, rename, show

Đối với mỗi distro, profile, system, ngoài thao tác add, có thể remove , show details, rename, list,

ví dụ:

Liệt kê profile:
```
root@cobbler#cobbler profile list
```
Show details:
```
root@cobbler#cobbler profile report --name= raring-x86_64
```
Remove:
```
root@cobbler#cobbler profile remove --name= raring-x86_64
```
### e. Quản lý power on, power off, reboot machine (chức năng này chưa test)
Đối với mỗi system được tạo , có thể thao tác power on, power off, reboot machine.

Cobbler thực hiện poweron qua wake on LAN.

ví dụ: 
```
#cobbler system poweron --name=machine_01
```
chi tiết thêm lệnh xem tại 
```
#cobbler system --help
```
