## Xây dựng kịch bản down node DB
Giả sử 1 trong các node DB hỏng, chúng ta sẽ phải tạo node mới để thay thế. Node mới có tên là DB4: 192.168.4.204  
Trước hết chúng ta sẽ vào mysql trên 2 node và thử gõ câu lệnh
```
show status like 'wsrep%';
```
Nhìn vào dòng **wsrep_cluster_size** chúng ta sẽ thấy nó có giá trị là **2**, vì đã có 1 node bị hỏng.  
Bây giờ chúng ta sẽ cài đặt Percona để đồng bộ node mới  
**1. Cài đặt MySQL Multi Cluster - Percona XtraDB**
```
sudo apt-get remove -y apparmor
wget https://repo.percona.com/apt/percona-release_0.1-4.$(lsb_release -sc)_all.deb
sudo dpkg -i percona-release_0.1-4.$(lsb_release -sc)_all.deb
sudo apt-get update
sudo apt-get install -y percona-xtradb-cluster-57
```
Trong khi quá trình cài đặt diễn ra, hệ thống sẽ yêu cầu bạn nhập password cho tài khoản root của MySQL.

**2. Cầu hình đồng bộ các node**
Trước hết tắt dịch vụ MySQL trên node DB4
```
/etc/init.d/mysql stop
```
Trên node DB2, DB3: Thay đổi dòng sau trong file **/etc/mysql/percona-xtradb-cluster.conf.d/wsrep.conf**:
```
wsrep_cluster_address=gcomm://192.168.4.202,192.168.4.203,192.168.4.204
```
Trên node DB4: Cấu hình file **/etc/mysql/percona-xtradb-cluster.conf.d/wsrep.conf** như sau:
```
[mysqld]
datadir=/var/lib/mysql
user=mysql
# Path to Galera library
wsrep_provider=/usr/lib/galera3/libgalera_smm.so

# Cluster connection URL contains IPs of nodes
#If no IP is found, this implies that a new cluster needs to be created,
#in order to do that you need to bootstrap this node
wsrep_cluster_address=gcomm://192.168.4.202,192.168.4.203,192.168.4.204

# In order for Galera to work correctly binlog format should be ROW
binlog_format=ROW

# MyISAM storage engine has only experimental support
default_storage_engine=InnoDB

# Slave thread to use
wsrep_slave_threads= 8

wsrep_log_conflicts

# This changes how InnoDB autoincrement locks are managed and is a requirement for Galera
innodb_autoinc_lock_mode=2

# Node IP address
wsrep_node_address=192.168.4.204
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
Khởi động node DB4
```
/etc/init.d/mysql start
```
Trên tất cả các node thử kiểm tra trạng thái của cluster bằng lệnh:
```
show status like 'wsrep%';
```
Nếu dòng **wsrep_cluster_size** có giá trị là **3** tức là chúng ta đã thêm và đồng bộ thành công trên node mới. Thử kiểm tra trên node DB4 bằng lệnh
```
show databases;
```
Ta thấy node DB4 đã tự động đồng bộ các database  
![db4.png](http://sv1.upsieutoc.com/2017/11/02/db4.png)

Trên 2 node LBDB1 và LBDB2 trong file **/etc/haproxy/haproxy.cfg** tại dòng
```
server DB1 192.168.4.201:3306 check
```
sửa thành
```
server DB4 192.168.4.204:3306 check
```
