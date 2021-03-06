#Tìm hiểu NFS (Network File System)
##Mục lục:

[1 Giới thiệu](#1)

[2 NFS server](#2)
- [2.1 File cấu hình, dịch vụ](#2.1)
- [2.2 Các dịch vụ có liên quan](#2.2)
- [2.3 Xác minh các dịch vụ của NFS đang chạy](#2.3)
- [2.4 Cập nhật thay đổi cho /etc/exports](#2.4)
- [2.5 Chức năng mount tự động khi boot](#2.5)
- [2.6 Các trường hợp dùng NFS](#2.6)

[3 Demo NFS](#)

<a name="1"></a>
###1 Giới thiệu

<ul>
<li>NFS là hệ thống cung cấp dịch vụ chia sẻ file phổ biến trong hệ thống mạng Linux và Unix.</li>
<li>NFS cho phép các máy tính kết nối tới 1 phân vùng đĩa trên 1 máy từ xa giống như là local disk. Cho phép việc truyền file qua mạng được nhanh và trơn tru hơn.</li>
<li>NFS sử dụng mô hình Client/Server. Trên server có các disk chứa các file hệ thống được chia sẻ và một số dịnh vụ chạy ngầm (daemon) phục vụ cho việc chia sẻ với Client.</li>
<li>Cung cấp chức năng bảo mật file và quản lý lưu lượng sử dụng (file system quota).</li>
<li>Các Client muốn sử dụng các file system được chia sẻ thì sử dụng giao thức NFS để mount các file đó về.</li>
<li>Khi triển khai hệ thống lớn hoặc chuyên biệt cần áp dụng 3NFS, còn người dùng ngẫu nhiên hoặc nhỏ lẻ thì áp dụng 2NFS, 4NFS.</li>
<li>Với NFSv4, yêu cầu hệ thống phải có kernel phiên bản từ 2.6 trở lên
<li>Xử lý được những file lớn hơn 2GB,  đòi hỏi hệ thống phải có phiên bản kernel lớn hơn hoặc bằng 2.4x và glibc từ 2.2.x trở lên
<li>Client từ phiên bản kernel 2.2.18 trở đi đều hỗ trợ NFS trên nền TCP
</ul>

<a name="1"></a>
###2 NFS server

<a name="2.1"></a>
####2.1 File cấu hình, dịch vụ

Có ba tập tin cấu hình chính, bạn sẽ cần phải chỉnh sửa để thiết lập một máy chủ NFS: **/etc/exports, /etc/hosts.allow và /etc/hosts.deny**

**2.1.1 File /etc/export**

`dir host1(options) host2(options) hostN(options) …`

<ul>
<li>dir : thư mục hoặc file system muốn chia sẻ. </li>
<li>host : một hoặc nhiều host được cho phép mount dir. Có thể được định nghĩa là một tên, một nhóm sử dụng ký tự , * hoặc một nhóm sử dụng 1 dải địa chỉ mạng/subnetmask... </li>
<li>options : định nghĩa 1 hoặc nhiều options khi mount. </li>
</ul>

Các options:
<ul>
<li>ro: thư mục được chia sẻ chỉ đọc được; client không thể ghi lên nó.</li>
<li>rw: client có thể đọc và ghi trên thư mục.</li>
<li>no_root_squash: Mặc định, bất kỳ file truy vấn được tạo bởi người dùng (root) máy trạm đều được xử lý tương tự nếu nó được tạo bởi user nobody. Nếu no_root_squash được chọn, user root trên client sẽ giống như root trên server. </li>
<li>no_subtree_check: Nếu chỉ 1 phần của ổ đĩa được chia sẻ, 1 đoạn chương trình gọi là “thẩm tra lại việc kiểm tra cây con” được yêu cầu từ phía client (nó là 1 file n m trong phân vùng được chia sẻ). Nếu toàn bộ ổ đĩa được chia sẻ, việc vô hiệu hóa sự kiểm tra này sẽ tăng tốc độ truyền tải.</li>
<li>sync: thông báo cho client biết 1 file đã được ghi xong- tức là nó đã được ghi để lưu trữ an toàn-khi mà NFS hoàn thành việc kiểm soát ghi lên các file hệ thống. cách xử lí này có thể là nguyên nhân làm sai lệch dữ liệu nếu server khởi động lại.</li>
</ul>

Ví dụ 1 file cấu hình mẫu 

```sh
/usr/local *.123.vn(ro) 
/home 192.168.1.0/255.255.255.0(rw) 
/var/tmp 192.168.1.1(rw) 192.168.1.3(rw)
```

<ul>
<li>Dòng thứ nhất : Cho phép tất cả các host với tên miền định dạng “somehost”.123.vn được mount thư mục /usr/local với quyền chỉ đọc. </li>
<li>Dòng thứ hai : Cho phép bất kỳ host nào có địa chỉ IP thuộc subnet 192.168.1.0/24 được mount thư mục /home với quyền đọc và ghi. </li>
<li>Dòng thứ ba : Cho phép 2 host được mount thư mục /var/tmp với quyền đọc và ghi.</li>
</ul>

**2.1.2 File /etc/hosts.allow và /etc/hosts.deny**

Hai tập tin đặc biệt này giúp xác định các máy tính trên mạng có thể sử dụng các dịch vụ trên máy của bạn. Mỗi dòng trong nội dung file chứa duy nhất 1 danh sách gồm 1 dịch vụ và 1 nhóm các máy tính. Khi server nhận được yêu cầu từ client, các công việc sau sẽ được thực thi:

<ul>
<li>Kiểm tra file host.allow – nếu client phù hợp với 1 quy tắc được liệt kê tại đây thì nó có quyền truy cập.</li>
<li>Nếu client không phù hợp với 1 mục trong host.allow server chuyển sang kiểm tra trong host.deny để xem thử client có phù hợp với 1 quy tắc được liệt kê trong đó hay không (host.deny). Nếu phù hợp thì client bị từ chối truy cập.</li>
<li>Nếu client phù hợp với các quy tắc không được liệt kê trong cả 2 file thì nó sẽ được quyền truy cập.</li>
</ul>

Ví dụ:
Muốn chặn hoặc cho phép một host hoặc network thì thêm vào file deny hoặc allow.

`portmap: 10.10.10.5, 10.10.10.0/24`

<a name="2.2"></a>
####2.2 Các dịch vụ có liên quan

Để sử dụng dịch vụ NFS, cần có các daemon (dịch vụ chạy ngầm trên hệ thống) sau:

<ul>
<li>**Portmap**: Quản lý các kết nối, dịch vụ chạy trên port 2049 và 111 ở cả server và client.</li>
<li>**NFS**: Khởi động các tiến trình RPC (Remote Procedure Call) khi được yêu cầu để phục vụ cho chia sẻ file, dịch vụ chỉ chạy trên server.</li>
<li>**NFS lock**: Sử dụng cho client khóa các file trên NFS server thông qua RPC.</li>
</ul>

**2.2.1 Khởi động portmapper**

NFS phụ thuộc vào tiến trình ngầm quản lý các kết nối (portmap hoặc rpc.portmap), chúng cần phải được khởi động trước. 

Nó nên được đặt tại /sbin nhưng đôi khi trong /usr/sbin. Hầu hết các bản phân phối linux gần đây đều khởi động dịch vụ này trong „kịch bản khởi động‟ (boot scripts –tự khởi động khi server khởi động) nhưng vẩn phải đảm bảo nó được khởi động đầu tiên trước khi bạn làm việc với NFS (chỉ cần gõ lệnh netstat -anp |grep portmap để kiểm tra).

**2.2.2 Các tiến trình ngầm**

Dịch vụ NFS được hỗ trợ bởi 5 tiến trình ngầm:

<ul>
<li>**rpc.nfsd**: thực hiện hầu hết mọi công việc.</li>
<li>**rpc.lockd and rpc.statd**: quản lý việc khóa các file.</li>
<li>**rpc.mountd**: quản lý các yêu cầu gắn kết lúc ban đầu.</li>
<li>**rpc.rquotad**: quản lý các hạn mức truy cập file của người sử dụng trên server được truy xuất.</li>
<li>**lockd**: được gọi theo yêu cầu của nfsd. Vì thế bạn cũng không cần quan tâm lắm tới việc khởi động nó.</li>
<li>**statd**: thì cần phải được khởi động riêng.</li>
</ul>

Tuy nhiên trong các bản phân phối linux gần đây đều có kịch bản khởi động cho các tiến trình trên.
Tất cả các tiến trình này đều nằm trong gói nfs-utils, nó có thể được lưu giữ trong /sbin hoặc /usr/sbin.
Nếu bản phân phối của bạn không tích hợp chúng trong kịch bản khởi động, thì bạn nên tự thêm chúng vào, cấu hình theo thứ tự sau đây:
```sh
rpc.portmap
rpc.mountd 
rpc.nfsd
```

<a name="2.3"></a>
####2.3 Xác minh các dịch vụ của NFS đang chạy

Để làm điều này, ta truy vấn portmapper với lệnh **rpcinfo* để tìm ra dịch vụ nào đang được cung cấp.

<img src=http://i.imgur.com/z2Ksqb1.jpg>

<a name="2.4"></a>
####2.4 Cập nhật thay đổi cho /etc/exports

<ul>
<li>Nếu thay đổi trong /etc/exports, các thay đổi đó có thể chưa có hiệu lực ngay lập tức, bạn phải thực thi lệnh **exportfs -ra** để bắt nfst cập nhật lại nội dung file /etx/exports .</li>
<li>Nếu không tìm thấy lệnh exportfs thì bạn có thể kết thúc nfsd với lệnh HUD.</li>
<li>Nếu các việc đó không hoạt động, đừng quên kiểm tra lại hosts.allow để đảm bảo rằng bạn không quên việc khai báo danh sách các máy con trong đấy. Ngoài ra cũng nên kiểm tra danh sách các máy chủ trên bất kỳ hệ thống tường lửa nào mà bạn đã thiết lập.</li>
</ul>

<a name="2.5"></a>
####2.5 Chức năng mount tự động khi boot

Chỉnh sửa file **fstab** bên phía client. Để đảm bảo cho hoạt động nên dùng **hard,intr** cho nfs.

<img src=http://i.imgur.com/UrcQsIS.png>

<a name="2.6"></a>
####2.6 Các trường hợp dùng NFS

<ul>
<li>Ứng dụng hỗ trợ: VDI, Oracle, VMware ESXi, SAS Grid, SAP HANA, TIBCO, OpenStack, Docker, etc</li>
<li>Các khách hàng lớn.</li>
<li>Đơn giản, dễ quản lý.</li>
<li>Không cần client OS file system.</li>
<li>Dễ dàng mở rộng, thu hồi.</li>
<li>Dễ dàng di chuyển các storage.</li>
<li>Chạy trên Ethernet.</li>
<li>Hiệu suất lớn, độ trễ thấp. Hiệu suất tốt hơn iSCSl trong vài trường hợp.</li>
</ul>

####Chú ý

- Có thể cấu hình tùy chọn kích thước gói dữ liệu để tối ưu khi truyền với options **rsize** và **wsize** 

link tham khảo: http://blogit.edu.vn/toi-uu-he-thong-nfs/

- Không đặt rsize và wsize lớn hơn MTU của mạng truyền.
- Dùng **nfsstat** để có thêm thông tin về hoạt động của NFS.

- Thay đổi port cho statd: `statd -p 32765 -o 32766`
- Thay đổi port mountd: `mountd -p 32767`
				
- Dùng ssh để đổi port hoạt động	
```sh
# ssh root@10.10.10.40 -L 250:localhost:2049  -f sleep 60m
# ssh root@10.10.10.40 -L 251:localhost:32767 -f sleep 60m	
fstab: 10.10.10.40:/home  /mnt/home  nfs  rw,hard,intr,port=250,mountport=251  0 0	
```

NFS có 2 chế độ mount:
```sh
Mount cứng là ghi trực tiếp vào file /etc/fstab
Mount mềm là mount bằng lệnh thông thường và bị mất khi máy tính được khởi động lại
```
Tối 
<a name="3"></a>
###3 Demo NFS

PC1: Ubuntu server 14.04- NFS server IP: 10.10.10.40

PC2: Ubuntu server 14.04- NFS client IP: 10.10.10.41

PC1: 

- Cài đặt NFS server
`apt-get -y install nfs-kernel-server`
- Chỉnh sửa file **exportfs** ví dụ cho phép mount thư mục **/mnt/nfs** tới dải 10.10.10.0/24
`/mnt/nfs        10.10.10.0/24(rw,no_root_squash)`
- Khởi động lại NFS
`/etc/init.d/nfs-kernel-server restart`

PC2:

- Cài đặt NFS client
`apt-get  -y install nfs-common`
- Tạo thư mục /mnt/a và mount /mnt/nfs về thư mục vừa tạo
`mount 10.10.10.40:/mnt/nfs /mnt/a`
- Dùng lệnh **df -h** để kiểm tra

<img src=http://i.imgur.com/4cVUve4.jpg>


Tham khảo:

[1]- http://nfs.sourceforge.net/nfs-howto/




