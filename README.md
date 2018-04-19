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

Node Cluster Database sử dụng HAProxy và Keepalived để đảm bảo Loadbalance và Failover  
Cluster Web sử dụng Nginx và Keepalived để loadbalance và đảm bảo FailOver   

Đọc file Cluster.md để xem hướng dẫn thực hiện  
Đọc file DownNodeDB.md để xem hướng dẫn khi có Node DB bị down  
Đọc file DownNodeWP.md để xem hướng dẫn khi có Node WP bị down
