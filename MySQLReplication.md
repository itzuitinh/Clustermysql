# Các bước cài đặt cho khách
Tạo VIP

## Cài MySQL Replication
Clone 2 server. Đặt lại file host.  

```
# k9-yax-zd6-1
127.0.0.1 localhost
127.0.1.1  k9-yax-zd6-1     localhost
10.6.39.176	k9-yax-zd6-2
10.6.39.177	k9-yax-zd6-1
```
```
# k9-yax-zd6-2
127.0.0.1 localhost
127.0.1.1  k9-yax-zd6-2     localhost
10.6.39.176     k9-yax-zd6-2
10.6.39.177     k9-yax-zd6-1
```
Chỉnh sửa file `vim /etc/mysql/mysql.conf.d/mysqld.cnf` trên Server1
```
bind-address            = 10.6.39.177
server_id               = 1
log_bin                 = /var/log/mysql/mysql-bin.log
log_bin_index           = /var/log/mysql/mysql-bin.log.index
relay_log               = /var/log/mysql/mysql-relay-bin
relay_log_index         = /var/log/mysql/mysql-relay-bin.index
expire_logs_days        = 10
max_binlog_size         = 100M
log_slave_updates       = 1
auto-increment-increment = 2    
auto-increment-offset   = 1
```
Chỉnh sửa file `vim /etc/mysql/mysql.conf.d/mysqld.cnf` trên Server2
```
bind-address            = 10.6.39.176
server_id               = 2
log_bin                 = /var/log/mysql/mysql-bin.log
log_bin_index           = /var/log/mysql/mysql-bin.log.index
relay_log               = /var/log/mysql/mysql-relay-bin
relay_log_index         = /var/log/mysql/mysql-relay-bin.index
expire_logs_days        = 10    
max_binlog_size         = 100M
log_slave_updates       = 1     
auto-increment-increment = 2
auto-increment-offset   = 2
```
Restart service mysql (2 node)
```
service mysql restart
```
Allow port 3306 (2 node)
```
ufw allow 3306/tcp
ufw allow 3306
service ufw restart
```
Truy cập mysql (2 node)
```
mysql -u debian-sys-maint -p
lyZIlmDUNjq9vIMO
```
Tạo user replication (2node)
```
# Node 1
GRANT REPLICATION SLAVE ON *.* TO 'replication'@'IP_Node_2' IDENTIFIED BY 'yBneW9DdUofe3VZ';

# Node 2
GRANT REPLICATION SLAVE ON *.* TO 'replication'@'IP_Node_1' IDENTIFIED BY 'yBneW9DdUofe3VZ';
```
Show master status (Node 1)
```
SHOW MASTER STATUS;
+------------------+----------+--------------+------------------+-------------------+
| File             | Position | Binlog_Do_DB | Binlog_Ignore_DB | Executed_Gtid_Set |
+------------------+----------+--------------+------------------+-------------------+
| mysql-bin.000001 |      788 |              |                  |                   |
+------------------+----------+--------------+------------------+-------------------+
```
Start Slave (Node 2)
```
STOP SLAVE;
CHANGE MASTER TO master_host='IP_Node_1', master_port=3306, master_user='replication', master_password='yBneW9DdUofe3VZ', master_log_file='mysql-bin.000001', master_log_pos=[Position];
START SLAVE;
```
Show master status (Node 2)
```
mysql> SHOW MASTER STATUS;
+------------------+----------+--------------+------------------+-------------------+
| File             | Position | Binlog_Do_DB | Binlog_Ignore_DB | Executed_Gtid_Set |
+------------------+----------+--------------+------------------+-------------------+
| mysql-bin.000001 |      788 |              |                  |                   |
+------------------+----------+--------------+------------------+-------------------+
```
Start Slave (Node 1)
```
STOP SLAVE;
CHANGE MASTER TO master_host='IP_Node_2', master_port=3306, master_user='replication', master_password='yBneW9DdUofe3VZ', master_log_file='mysql-bin.000001', master_log_pos=[Position];
START SLAVE;
```
Remove Server UUID do clone bị trùng
```
rm /var/lib/mysql/auto.cnf
```
Restart service mysql
```
service mysql restart
```

## Cài GlusterFS
rsync thư mục chứa code web (Node 1)
```
mkdir -p /home/ab
rsync -avhP /home/michal /home/ab
```
Cài GlusterFS (2 Node)
```
apt-get install software-properties-common -y
add-apt-repository ppa:gluster/glusterfs-3.8
apt-get update
apt install -y glusterfs-******
```
khởi động config rpcbind (2 Node)
```
systemctl start rpcbind
systemctl enable rpcbind
```
Tạo thư mục chung (2 Node)
```
mkdir -p /mnt/glusterfs
```
Allow port 24007
```
ufw allow 24007/tcp
ufw allow 24007
service ufw restart
```
Node 1 Peer đến node 2 (Node 1)
```
gluster peer probe IP_Node_2
```
Xem status
```
gluster peer status
```
Tạo volume (Node 1)
```
gluster volume create glusterfs replica 2 transport tcp k9-yax-zd6-1:/mnt/glusterfs k9-yax-zd6-2:/mnt/glusterfs force
```
Start volume (Node 1)
```
gluster volume start glusterfs
```
Xem status volume (2Node)
```
gluster volume status
Status of volume: glusterfs
Gluster process                             TCP Port  RDMA Port  Online  Pid
------------------------------------------------------------------------------
Brick k9-yax-zd6-1:/mnt/glusterfs           49152     0          Y       3216 
Brick k9-yax-zd6-2:/mnt/glusterfs           49152     0          Y       32592
NFS Server on localhost                     2049      0          Y       32619
Self-heal Daemon on localhost               N/A       N/A        Y       32624
NFS Server on k9-yax-zd6-1                  2049      0          Y       3237 
Self-heal Daemon on k9-yax-zd6-1            N/A       N/A        Y       3242 
```
Allow các port tcp ở trên
```
ufw allow 49152/tcp
ufw allow 49152
ufw allow 2049/tcp
ufw allow 2049
service ufw restart
```
Thêm dòng sau vào file `/etc/fstab` (2Node)
```
localhost:/glusterfs /home/michal/  glusterfs   defaults,_netdev      0 0
```
Dùng lệnh `mount -a` để mount lại.  
Rsync lại thư mục web
```
rsync -avhP ab michal/
#cd /home/ab
#rsync -avh .bash_history /home/michal/
#rsync -avh .c* /home/michal/
#rsync -avh .bash_history /home/michal/
#rsync -avh .gitconfig /home/michal/
#rsync -avh .htaccess /home/michal/
#rsync -avh .lesshst /home/michal/
#rsync -avh .local/ /home/michal/
#rsync -avh .mysql_history /home/michal/
#rsync -avh .nano /home/michal/
#rsync -avh .nano /home/michal/
#rsync -avh .v8flags.4.5.103.35.michal.json /home/michal/
```
Bật lại apache2
```
service apache2 start
```

## Cài Keepalived
Cài đặt
```
apt-get install keepalived -y
```
Trong file `/etc/keepalived/keepalived.conf`. Điền VIP vào (Node 1)
```
! Configuration File for keepalived

global_defs{
        notification_email {
                plachym.it@gmail.com
        }
        notification_email_from plachym.it@gmail.com
        smtp_server localhost
        smtp_connect_timeout 30
}

vrrp_instance VI_1 {
        state MASTER
        interface eth1
        virtual_router_id 101
        priority 200
        advert_int 1
        authentication {
                auth_type PASS
                auth_pass yBneW9DdUofe3VZ
        }
        virtual_ipaddress {
                VIP
        }
}
```
Trong file `/etc/keepalived/keepalived.conf`. Điền VIP vào (Node 2)
```
! Configuration File for keepalived

global_defs{
        notification_email {
                plachym.it@gmail.com
        }
        notification_email_from plachym.it@gmail.com
        smtp_server localhost
        smtp_connect_timeout 30
}

vrrp_instance VI_1 {
        state BACKUP
        interface eth1
        virtual_router_id 101
        priority 100
        advert_int 1
        authentication {
                auth_type PASS
                auth_pass yBneW9DdUofe3VZ
        }
        virtual_ipaddress {
                VIP
        }
}
```
Restart keepalived
```
/etc/init.d/keepalived restart
```

