# Hướng dẫn triển khai Cluster Web Server 
![cluster.png](http://sv1.upsieutoc.com/2017/11/02/cluster.png)
## Mô hình triển khai
Ở đây chúng ta triển khai mô hình gồm 2 cluster. 1 cluster Web và 1 cluster database.
Cluster Web:
- LBWP1: 192.168.4.151
- LBWP2: 192.168.4.152
- WP1: 192.168.4.51
- WP2: 192.168.4.52
- WP3: 192.168.4.53

Cluster Database:
- LBDB1: 192.168.4.101
- LBDB2: 192.168.4.102
- DB1: 192.168.4.201
- DB2: 192.168.4.202
- DB3: 192.168.4.203

## Cấu hình đồng bộ các node backend cluster database
Việc cài đặt và đồng bộ các database thực hiện trên 3 node DB1, DB2, DB3. Trong bài hướng dẫn này dùng Percona XtraDB để đồng bộ.

**1. Cài đặt MySQL Multi Cluster - Percona XtraDB**  
Nếu trước đây bạn đã từng cài đặt MySQL trên máy chủ, có thể có hồ sơ Apparmor sẽ các node Perodes XtraDB Cluster giao tiếp với nhau. Vì vậy nên gỡ bỏ gói apparmor để tránh xung đột:
```
sudo -y apt-get remove apparmor
```
Cài đặt các gói bổ trợ và update hệ điều hành
```
wget https://repo.percona.com/apt/percona-release_0.1-4.$(lsb_release -sc)_all.deb
sudo dpkg -i percona-release_0.1-4.$(lsb_release -sc)_all.deb
sudo apt-get update
```
Cài đặt Percona XtraDB
```
sudo apt-get install -y percona-xtradb-cluster-57
```
Trong khi quá trình cài đặt diễn ra, hệ thống sẽ yêu cầu bạn nhập password cho tài khoản root của MySQL.

**2. Cầu hình đồng bộ từng node**  
Chú ý: Sau khi cài đặt Percona thì dịch vụ Mysql tự động được bật, chúng ta cần tắt dịch vụ trên cả 3 node DB bằng lệnh:
```
/etc/init.d/mysql stop
```
**2.1. Trên Node DB1**: Cấu hình file **/etc/mysql/percona-xtradb-cluster.conf.d/wsrep.cnf** như sau
```
[mysqld]
datadir=/var/lib/mysql
user=mysql

# Path to Galera library
wsrep_provider=/usr/lib/galera3/libgalera_smm.so

# Cluster connection URL contains IPs of nodes DB1, DB2, DB3
#If no IP is found, this implies that a new cluster needs to be created,
#in order to do that you need to bootstrap this node
wsrep_cluster_address=gcomm://192.168.4.201,192.168.4.202,192.168.4.203

# In order for Galera to work correctly binlog format should be ROW
binlog_format=ROW

# MyISAM storage engine has only experimental support
default_storage_engine=InnoDB

# Slave thread to use
wsrep_slave_threads= 8
wsrep_log_conflicts

# This changes how InnoDB autoincrement locks are managed and is a requirement for Galera
innodb_autoinc_lock_mode=2

# Node DB1 IP address
wsrep_node_address=192.168.4.201

# Cluster name
wsrep_cluster_name=pxc-cluster

#If wsrep_node_name is not specified,  then system hostname will be used
wsrep_node_name=pxc-cluster-node-1

#pxc_strict_mode allowed values: DISABLED,PERMISSIVE,ENFORCING,MASTER
pxc_strict_mode=ENFORCING

# SST method
wsrep_sst_method=xtrabackup-v2

#Authentication for SST method
wsrep_sst_auth="vccloud:vccorp"
```
Khởi động Node đầu tiên với câu lệnh
```
/etc/init.d/mysql bootstrap-pxc
```
Lệnh này sẽ khởi động nút đầu tiên và boostrap để kết nối với các máy còn lại
Sau khi node đầu tiên được bật, ta có thể kiểm tra được trạng thái của cluster bằng lệnh sau:
```
show status like 'wsrep%';
```
Và chúng ta sẽ có kết quả giống như bên dưới:

|Variable_name|Value|
|-------------|------|
|wsrep_local_state_uuid|9358494f-bdf3-11e7-ba56-2bd5f2eb85be|
|...|...|
| wsrep_local_state| 4|
| wsrep_local_state_comment|Synced|
|...|...|
| wsrep_cluster_size|1|
| wsrep_cluster_status| Primary|
| wsrep_connected| ON|
|...|...|
| wsrep_ready| ON|

Trong MySQL, chúng ta tạo user mới và set quyền:
```
mysql> CREATE USER 'vccloud'@'localhost' IDENTIFIED BY 'vccorp';
mysql> GRANT PROCESS, RELOAD, LOCK TABLES, REPLICATION CLIENT ON *.* TO 'vccloud'@'localhost';
mysql> FLUSH PRIVILEGES;
mysql> exit;
```
**2.2. Trên Node DB2**:  
Cấu hình tương tự Node 1 trong file **/etc/mysql/percona-xtradb-cluster.conf.d/wsrep.cnf** và chỉ sửa dòng
```
wsrep_node_addres=192.168.4.202
```
Khởi động MySQL lên:
```
/etc/init.d/mysql start
```
Vào MySQL kiểm tra và chúng ta sẽ được kết quả

|Variable_name|Value|
|-------------|------|
|wsrep_local_state_uuid|9358494f-bdf3-11e7-ba56-2bd5f2eb85be|
|...|...|
| wsrep_local_state| 4|
| wsrep_local_state_comment|Synced|
|...|...|
| wsrep_cluster_size|2|
| wsrep_cluster_status| Primary|
| wsrep_connected| ON|
|...|...|
| wsrep_ready| ON|

**2.3. Trên Node DB3**:  
Cấu hình tương tự Node 1 trong file **/etc/mysql/percona-xtradb-cluster.conf.d/wsrep.cnf** và chỉ sửa dòng
```
wsrep_node_addres=192.168.4.203
```
Khởi động MySQL lên:
```
/etc/init.d/mysql start
```
Vào MySQL kiểm tra và chúng ta sẽ được kết quả

|Variable_name|Value|
|-------------|------|
|wsrep_local_state_uuid|9358494f-bdf3-11e7-ba56-2bd5f2eb85be|
|...|...|
| wsrep_local_state| 4|
| wsrep_local_state_comment|Synced|
|...|...|
| wsrep_cluster_size|3|
| wsrep_cluster_status| Primary|
| wsrep_connected| ON|
|...|...|
| wsrep_ready| ON|

**3. Test**  
Để kiểm tra replication, chúng ta sẽ thử tạo 1 cơ sở dữ liệu mới trên node DB2, 1 bảng cho CSDL mới đó trên node DB3 và thêm một số bản ghi trên node DB1.
Trên Node DB2: Ta tạo 1 CSDL mới
```
mysql> CREATE DATABASE percona;
```
Trên Node DB3: Ta tạo bảng cho CSDL đã tạo ở trên
```
mysql> USE percona;
mysql> CREATE TABLE example (node_id INT PRIMARY KEY, node_name VARCHAR(30));
```
Trên Node DB1: Ta thử tạo bản ghi với câu lệnh
```
mysql> INSERT INTO percona.example VALUES (1, 'percona1');
```
Chúng ta thử kiểm tra quá trình đồng bộ trên cả 3 node với câu lệnh
```
mysql> SELECT * FROM percona.example;
```
Và kết quả sẽ được như hình:

![replication.png](http://sv1.upsieutoc.com/2017/10/31/replication.png)

## Triển khai HAProxy, Keepalive trên 2 node LBDB1 và LBDB2

LBDB1: 192.168.4.101  
LBDB2: 192.168.4.102

**1. Chuẩn bị các node database (DB1, DB2, DB3)**  
Đầu tiên chúng ta sẽ tiến hành tạo user để connect với HA Proxy. Chúng ta cần tạo 2 user, 1 dùng cho HAProxy check status, 1 dùng để connect.
Chúng ta chỉ cần thực hiện tạo user và gán quyền trên 1 trong 3 node vì các node sẽ tự đồng bộ với nhau.

**Tạo User để HAProxy check status**
```
INSERT INTO mysql.user (Host,User,ssl_cipher,x509_issuer,x509_subject) values ('192.168.4.101','vccloudlb1','','','');
INSERT INTO mysql.user (Host,User,ssl_cipher,x509_issuer,x509_subject) values ('192.168.4.102','vccloudlb2','','','');
```
**Tạo User để HAProxy connect tới Database**
```
GRANT ALL PRIVILEGES ON *.* TO 'vccloud'@'192.168.4.101' IDENTIFIED BY 'vccorp' WITH GRANT OPTION;
GRANT ALL PRIVILEGES ON *.* TO 'vccloud'@'192.168.4.102' IDENTIFIED BY 'vccorp' WITH GRANT OPTION;
```
**2. Triển khai HAProxy trên 2 node (LBDB1, LBDB2)**  
HAProxy là phần mềm cân bằng tải TCP/HTTP và là giải pháp reverse proxy mã nguồn mở phổ biến. HAProxy thường dùng để cải thiện hiệu suất (performance) và sự tin cậy (reliability) của môi trường máy chủ bằng cách phân tán lưu lượng tải (workload) trên nhiều máy chủ (như web, application, database).
Để cài đặt HAProxy, trên cả 2 Node LBDB1 và LBDB2 ta dùng câu lệnh:
```
sudo apt-get install haproxy -y
```
Cho phép Haproxy khởi chạy khi khởi động
```
echo ENABLED=1 >> /etc/default/haproxy
```
Chúng ta thêm cấu hình HAProxy dưới đây vào file **/etc/haproxy/haproxy.cfg** trên cả 2 node LBDB1 và LBDB2
```
listen  mysql-cluster
        bind 0.0.0.0:3306
        mode tcp
        balance roundrobin
        server DB1 192.168.4.201:3306 check
        server DB2 192.168.4.202:3306 check
        server DB3 192.168.4.203:3306 check

listen  stats
        bind 0.0.0.0:9000
        mode http
        stats enable
        stats uri /monitor
        stats auth vccloud:vccorp #User Pass đăng nhập vào trang monitor
```
Bây giờ chúng ta restart lại HAProxy để load config
```
service haproxy restart
```
Bạn có thể thử thực hiện 1 vài câu lệnh truy vấn vào IP của HAProxy ví dụ như:
```
mysql -h 192.168.4.102 -u vccloud -p -e "SELECT * FROM percona.example;"
mysql -h 192.168.4.101 -u vccloud -p -e "SELECT * FROM percona.example;"
```
Bây giờ ta có thể truy cập vào trang http://192.168.4.101:9000/monitor & http://192.168.4.102:9000/monitor để kiểm tra việc load balance:
![LBDB.png](http://sv1.upsieutoc.com/2017/11/01/LBDB.png)

**3. Triển khai KeepAlived trên 2 node (LBDB1, LBDB2)**  
Chúng ta cài đặt keepalived với lệnh
```
sudo apt-get install keepalived -y
```
File config của keepalived nằm tại **/etc/keepalived/keepalived.conf**. Nếu chưa có bạn có thể tạo ra và cấu hình như sau:
Trên node LBDB1
```
! Configuration File for keepalived

global_defs{
        notification_email {
                root@vccloud.vn
        }
        notification_email_from LBDB1@vccloud.vn
        smtp_server localhost
        smtp_connect_timeout 30
}

vrrp_instance VI_1 {
        state MASTER
        interface ens33
        virtual_router_id 101
        priority 200
        advert_int 1
        authentication {
                auth_type PASS
                auth_pass vccloud
        }
        virtual_ipaddress {
                192.168.4.100
        }
}
```
Trên node LBDB2
```
! Configuration File for keepalived

global_defs{
        notification_email {
                root@vccloud.vn
        }
        notification_email_from LBDB2@vccloud.vn
        smtp_server localhost
        smtp_connect_timeout 30
}

vrrp_instance VI_1 {
        state BACKUP
        interface ens33
        virtual_router_id 101
        priority 100
        advert_int 1
        authentication {
                auth_type PASS
                auth_pass vccloud
        }
        virtual_ipaddress {
                192.168.4.100
        }
}
```
Chúng ta có thể restart keepalived để load config:
```
/etc/init.d/keepalived restart
```
Với cấu hình này, LBDB1 sẽ là master nắm giữ Virtual IP, LBDB2 sẽ là backup cho LBDB1. Trong card mạng ens33 của LBDB1 sẽ có thêm 1 IP nữa là Virtual IP, trên LBDB2 sẽ không có.
Card mạng ens33 của LBDB1  
![ens33LBDB1.png](http://sv1.upsieutoc.com/2017/11/01/ens33LBDB1.png)
Card mạng ens33 của LBDB2 sẽ không có Virtual IP  
![ens33LBDB2.png](http://sv1.upsieutoc.com/2017/11/01/ens33LBDB2.png)
Bạn có thể shutdown interface ens33 của LBDB1 để test failover
```
root@LBDB1:~# ifdown ens33
```
Bây giờ LBDB2 sẽ tiếp nhận Virtual IP
![keepalived.png](http://sv1.upsieutoc.com/2017/11/01/keepalived.png)
Nếu bạn đưa cổng ens33 của LBDB1 hoạt động trở lại, LBDB1 sẽ tiếp nhận lại Virtual IP và LBDB2 sẽ lại trở thành backup.

## Cấu hình node Cluster Web
**1. Tạo database cho wordpress**  
Chúng ta sẽ tạo database trên node DB1, DB2, DB3(chỉ cần tạo ở 1 trong 3, các node sẽ tự đồng bộ)
Đăng nhập vào mysql bằng lệnh:
```
mysql -u root -p
```
Bạn điền password tài khoản root của bạn để đăng nhập vào mysql. Sau đó chúng ta sẽ tạo cơ sở dữ liệu cho wordpress:
```
CREATE DATABASE wordpress;
```
Bạn có thể chuyển sang các node khác và xem database đã đồng bộ hay chưa

![db2.png](http://sv1.upsieutoc.com/2017/11/01/db2.png)
![db3.png](http://sv1.upsieutoc.com/2017/11/01/db3.png)

**2. Cấu hình đồng bộ content cho 3node WP1, WP2, WP3 bằng GlusterFS**  
Trên cả 3 node, cài đặt glusterfs bằng lệnh
```
sudo apt-get install software-properties-common -y
sudo add-apt-repository ppa:gluster/glusterfs-3.8
sudo apt-get update
sudo apt install -y glusterfs-*
```
Sau đó ta khởi động config rpcbind để GlusterFS-Client có thể được tự động mount port sau khi khởi động Gluster-server .
```
systemctl start rpcbind
systemctl enable rpcbind
```
Tạo thư mục trên các máy để làm mount point
```
mkdir -p /wordpress/www
```
Tạo ra thêm mount point là thư mục sẽ chứa wordpress
```
mkdir -p /var/www/html
```
Trên một node bất kỳ (VD Node1), ta bắt đầu triển khai GlusterFS để bắt đầu sync thư mục sẽ chứa web content
```
gluster peer probe 192.168.4.52
gluster peer probe 192.168.4.53
```
Sau đó trên cả 3 máy ta kiểm tra tình trạng kết nối bằng lệnh
```
gluster peer status
```
Tiếp theo tạo volume để tạo replica web content ta sử dụng lệnh
```
sudo gluster volume create wordpressvol replica 3 transport tcp WP1:/wordpress/www WP2:/wordpress/www WP3:/wordpress/www force
```
Khởi động volume bằng lệnh
```
sudo gluster volume start wordpressvol
```
Có thể kiểm tra tình trạng volume thông qua lệnh
```
gluster volume status
```
Trong bảng status này ta sẽ thấy cột Online chuyển sang Y như vậy có nghĩa là volume này đã hoạt động.
![gluster-volume.png](http://sv1.upsieutoc.com/2017/11/01/gluster-volume.png)

Bây giờ ta cần cấu hình lại mount point để sử dụng GlusterFS trong file /etc/fstab trên cả 3 node.
Đưa thêm dòng sau vào cuối file /etc/fstab
```
localhost:/wordpressvol	/var/www/html  glusterfs   defaults,_netdev      0 0
```
Sau đó dùng lệnh `mount -a` để bắt đầu sync web content.

**3. Cài đặt wordpress**  
**Cài đặt Apache và PHP**  
Ta cài apache2 và php trên cả 3 node với lệnh  
```
sudo apt-get update
sudo apt-get install -y apache2 php7.0 libapache2-mod-php7.0 php7.0-mcrypt php7.0-mysql mysql-client
```
Thay đổi user cho thư mục /var/www/html bằng câu lệnh
```
sudo chown -R www-data:www-data /var/www/html
```

**Tải wordpress**  
Trên 1 trong 3 node bất kỳ WP1, WP2, hoặc WP3 ta tải và cấu hình wordpress như sau (Các node khác sẽ tự đồng bộ):
```
cd /var/www/html
sudo wget http://wordpress.org/latest.tar.gz
sudo tar xzvf latest.tar.gz
```
**Cấu hình wordpress**  
Chúng ta sẽ dùng cấu hình trên file **wp-config.php** với user, password, database chúng ta đã tạo trên các node DB.
File này sẽ dựa trên file **wp-config-sample.php**
```
sudo cp /var/www/html/wordpress/wp-config-sample.php /var/www/html/wordpress/wp-config.php
```
Mở file wp-config.php và sửa như sau:
```
// ** MySQL settings - You can get this info from your web host ** //
/** The name of the database for WordPress */
define('DB_NAME', 'wordpress');

/** MySQL database username */
define('DB_USER', 'vccloud');

/** MySQL database password */
define('DB_PASSWORD', 'vccorp');

/** MySQL hostname */
define('DB_HOST', '192.168.4.100');
```
Taị phần DB_USER và DB_PASSWORD, chúng ta dùng user và pass mà đã tạo dùng để HAProxy connect tới database ở trên.  
Tại phần DB_HOST, chúng ta sẽ trỏ đến Virtual IP đã cấu hình ở đến để load balace và keepalived  
Bây giờ ta có thể truy cập cả 3 thư node cùng 1 wordpress.
![wordpress1.png](http://sv1.upsieutoc.com/2017/11/02/wordpress1.png)
![wordpress2.png](http://sv1.upsieutoc.com/2017/11/02/wordpress2.png)
![wordpress3.png](http://sv1.upsieutoc.com/2017/11/02/wordpress3.png)

## Triển khai nginx để load balance và KeepAlived trên 2 Node LBWP1 và LBWP2
Cài đặt nginx và keepalived trên cả 2 node
```
sudo apt-get update
sudo apt-get install nginx
sudo apt-get install keepalived
```
**Cấu hình keepalived**  
Trong file **/etc/keepalived/keepalived.conf** ta cấu hình như sau:
Trên node LBWP1
```
! Configuration File for keepalived

global_defs{
        notification_email {
                root@vccloud.vn
        }
        notification_email_from LBWP1@vccloud.vn
        smtp_server localhost
        smtp_connect_timeout 30
}

vrrp_instance VI_1 {
        state MASTER
        interface ens33
        virtual_router_id 101
        priority 200
        advert_int 1
        authentication {
                auth_type PASS
                auth_pass vccloud
        }
        virtual_ipaddress {
                192.168.4.150
        }
}
```
Trên node LBWP2
```
! Configuration File for keepalived

global_defs{
        notification_email {
                root@vccloud.vn
        }
        notification_email_from LBWP2@vccloud.vn
        smtp_server localhost
        smtp_connect_timeout 30
}

vrrp_instance VI_1 {
        state BACKUP
        interface ens33
        virtual_router_id 101
        priority 100
        advert_int 1
        authentication {
                auth_type PASS
                auth_pass vccloud
        }
        virtual_ipaddress {
                192.168.4.150
        }
}
```

**Cấu hình load balance bằng nginx**  
Trong file **/etc/nginx/sites-enable/default** ta cấu hình như sau trên cả 2 node
```
upstream lbwp {
        server 192.168.4.51;
        server 192.168.4.52;
        server 192.168.4.53;
}

server {
        listen 80;

        location / {
                proxy_pass http://lbwp;
        }
}
```
Sau đó bạn restart nginx để load cấu hình
```
/etc/init.d/nginx restart
```
Chúng ta có thể test cấu hình bằng cách shutdown node LBWP1, ngay lập tức LBWP2 sẽ tiếp nhận VIP giúp client vẫn có thể truy cập website bình thường bằng VIP đó.  
LBWP2 tiếp nhận VIP  
![vip.png](http://sv1.upsieutoc.com/2017/11/02/vip.png)

Website vẫn truy cập bình thường bằng VIP  
![VIP.png](http://sv1.upsieutoc.com/2017/11/02/VIP.png)

Trên đây là bài hướng dẫn triển khai Cluster Web đáp ứng được High Availability & Fail Over.

