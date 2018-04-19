## Xây dựng kịch bản down node WP
Giả sử 1 trong các node WP hỏng, chúng ta sẽ phải tạo node mới để thay thế. Node mới có tên là WP4: 192.168.4.54  
Trước hết trên các node còn sống, kiểm tra các kết nối bằng lệnh
```
gluster peer status
```
Ta thấy node WP1 đã disconect  
![wp1fail.png](http://sv1.upsieutoc.com/2017/11/02/wp1fail.png)  
Trên node WP4, cài đặt glusterfs để sync thư mục web
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
Trên 1 trong 2 node WP2, WP3 cũ, ta thực hiện kết nối node WP4 vào
```
gluster peer probe 192.168.4.54
```
Kiểm tra lại tình trạng kết nối
```
gluster peer status
```
![wp4on.png](http://sv1.upsieutoc.com/2017/11/02/wp4on.png)  
Ta thấy node WP4 đã được kết nối vào gluster  
Tiếp theo ta đưa thêm node WP4 vào volume **wordpressvol** đã tạo trước đó (Thực hiện trên node WP2 hoặc WP3)
```
gluster volume add-brick wordpressvol replica 4 WP4:/wordpress/www force
```
Và xóa node WP1 ra khỏi volume
```
gluster volume remove-brick wordpressvol replica 3 WP1:/wordpress/www force
```
Kiểm tra info volume
```
gluster volume info
```
![volume.png](https://i.imgur.com/AGXfbeQ.png)  
Đưa thêm dòng sau vào cuối file /etc/fstab để mount
```
localhost:/wordpressvol /var/www/html  glusterfs   defaults,_netdev      0 0
```
Dùng lệnh `mount -a` để sync thư mục  
Trên node WP4, cài đặt apache và PHP
```
sudo apt-get update
sudo apt-get install -y apache2 php7.0 libapache2-mod-php7.0 php7.0-mcrypt php7.0-mysql mysql-client
```
Thay đổi user cho thư mục /var/www/html bằng câu lệnh
```
sudo chown -R www-data:www-data /var/www/html
```
Restart apache2 để load cấu hình
```
service apache2 restart
```
Kiểm tra xem node WP4 đã được sync chưa  
![wp4sync.png](http://sv1.upsieutoc.com/2017/11/02/wp4sync.png)  
Truy cập WP4 thành công với cùng wordpress
![wp4done.png](http://sv1.upsieutoc.com/2017/11/02/wp4done.png)
