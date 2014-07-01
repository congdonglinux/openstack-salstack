openstack-salstack
==================
##1. Giới thiệu

Đây là 1 script nhỏ để giúp cho việc cài đặt Openstack Multi-Node đơn giản hơn, ít nhàm chán hơn, quản lý tập trung dễ dàng hơn... và còn nhiều cái hơn nữa. Ý tưởng là sử dụng [saltstack](http://www.saltstack.com/) để quản lý cấu hình tập trung. Chúng ta chỉ chỉnh sửa cấu hình tại Saltstack master và áp cấu hình lên tất cả các Node trong mô hình của mình. Mỗi khi thay đổi cấu hình ta cũng chỉ cần sửa trên Master và lại áp cấu hình mới này lên các Node theo ý muốn sau đó restart lại các service tương ứng, mọi việc sẽ được thực hiện trong vòng chưa đầy 1 nốt nhạc thay vì phải SSH lên từng Node và sửa.

##2. Mô hình Lab 
Mô hình sử dụng Neutron để cung cấp dịch vụ mạng cho máy ảo (Sử dụng ML2 plugin và GRE tunnel)
 **2.1 Controller Node:**
- Openstack Controller
- Saltmaster
- Saltminion

Hostname | controller
-------- | ----------
eth0 | 10.0.0.11
eth1 | Internet Access
OS | Ubuntu 14.04 amd64

 **2.2 Compute nodes**
- Openstack Compute & Network
- Saltminion

Hostname | compute1
-------- | --------
eth0 | 10.0.0.21
eth1 | 10.0.1.21
eth2 | Internet Access
/dev/sdb | cinder-volume

Hostname | compute2
-------- | --------
eth0 | 10.0.0.31
eth1 | 10.0.1.31
eth2 | Internet Access
/dev/sdb | cinder-volume

##3. Các bước cài đặt
Cách cài đặt salt-master và salt-minion [tham khảo tại đây](https://github.com/d0m0reg00dthing/saltstack)

**3.1 Cấu hình Saltstack**

Controller Node:
**/etc/salt/master**
```shell
interface: 0.0.0.0

file_roots:
  base:
	    - /srv/salt
```

**/etc/salt/minion**
```shell
master: 10.0.0.11
```
Restart lại các dịch vụ
```shell
service salt-master restart
service salt-minion restart
```

Compute Nodes
**/etc/salt/minion**
```shell
master: 10.0.0.11
```
Restart lại dịch vụ
```shell
service salt-minion restart
```
Trên Controller Node 
```shell
salt-key -L

salt-key -A
(Chọn Y cho tất cả câu trả lời)
```
Kiểm tra xem master và minion thông nhau chưa
```shell
salt '*' test.ping
```
Cài đặt Openstack

Tải file salt.openstack.tar.bz2 đặt tại thư mục /srv (Phải đặt đúng thư mục này)
```shell
 cd /srv
 tar -xjvf salt.openstack.tar.bz2
```

Cách 1: Cài đặt & cấu hình từng dịch vụ
Cài đặt Controller Node
Trên Controller Node
```shell
# Thêm 1 record vào file /etc/hosts
salt 'controller' state.sls host -l debug
# Cài đặt ntp và sử dụng ntp.ubuntu.com làm update server
salt 'controller' state.sls ntp -l debug

# Cài đặt rabbitmq-server và đổi mật khẩu guest
salt 'controller' state.sls rabbit -l debug

# Cài đặt mysql-server và tạo database/user/grant quyền tương tứng
salt 'controller' state.sls mysql -l debug

# Cài đặt Openstack Identity Service
salt 'controller' state.sls keystone -l debug

# Cài đặt Openstack Images Service
salt 'controller' state.sls glance -l debug

# Tạo user/service/tenant/endpoint cho các dịch vụ
salt 'controller' state.sls keystone.create-user-tenant -l debug

# Cài đặt và cấu hình nova-api/nova-cert/....
salt 'controller' state.sls nova.nova-api -l debug

# Cài đặt cấu hình cinder-api
salt 'controller' state.sls cinder.api -l debug

# Cài đặt Dashboard
salt 'controller' state.sls horizon -l debug

# Cài đặt/cấu hình neutron-server
salt 'controller' state.sls neutron.api -l debug
```

Cài đặt cho các compute node
Trên Controller Node
```shell
# Tạo record trong /etc/hosts
salt 'compute*' state.sls hosts -l debug

# Cài đặt ntp. Sử dụng controller làm update server
salt 'compute*' state.sls ntp -l debug

# Cài đặt python-mysqldb
salt 'compute*' state.sls python-mysql -l debug

# Cài đặt/cấu hình nova-compute
salt 'compute*' state.sls nova.nova-compute -l debug

# Cài đặt cinder volume.Tạo pv và vg sử dụng /dev/sdb
salt 'compute*' state.sls cinder.volume -l debug

# Cài đặt/cấu hình neutron-l3-agent, neutron-plugin-ml2, neutron-dhcp-agent...
salt 'compute*' state.sls neutron.network -l debug
```

Cách 2: Sử dụng 1 command cài đặt tất cả các Node
Trên Controller Node
```shell
salt '*' state.highstate -l debug
```

Kiểm tra sau khi cài đặt
Trên Controller Node
```shell
source /root/openrc

# Kiểm tra neutron
neutron agent-list

# Kiểm tra nova
nova-manage service list

# Kiểm tra glance
glance image-list

# Kiểm tra keystone
keystone endpoint-list
keystone service-list
keystone user-list

# Kiểm tra nova
nova image-list

# Kiểm tra cinder
cinder list
```

Kết thúc
Lần đầu viết script cho Saltstack, còn nhiều chỗ trình bày chưa thật sự đẹp nhưng quan trọng là script chạy được (ít ra là những lần mình test trên 2 Node/3 Node đều chạy). Hạn chế của script còn rất nhiều, tương lai mình sẽ fix và thêm 1 vài chức năng nữa. 
Mục đích mình shared script này không phải để mọi người kéo về và gõ cái đống command trên mà mình muốn mọi người sẽ tìm hiểu về nó (saltstack) đê chỉnh sửa/viết thêm những script phù hợp với công việc cuả từng người để việc triên khai Openstack ít nhàm chán hơn. Trong quá trình cài đặt nếu có lỗi gì mọi người có thể liên lạc với mình qua FB: https://www.facebook.com/cucxabong
