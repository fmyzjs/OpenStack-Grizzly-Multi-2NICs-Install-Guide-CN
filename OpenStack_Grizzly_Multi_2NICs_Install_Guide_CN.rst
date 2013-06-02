==========================================================
  OpenStack Grizzly 双网卡安装指南
==========================================================

:Version: 1.0
:Source: https://github.com/fmyzjs/OpenStack-Grizzly-2NICs-Install-Guide-CN
:Keywords: 多点OpenStack安装, Grizzly, Quantum, Nova, Keystone, Glance, Horizon, Cinder, OpenVSwitch, KVM, Ubuntu Server 12.04 (64 bits).

作者
==========

`fmyzjs <http://www.idev.pw>`_ <fmyzjs@gmail.com>

本指南fork自

`ist0ne <https://github.com/ist0ne/OpenStack-Grizzly-Install-Guide-CN>`_ 
的git仓库。向第二作者致敬！
`Bilel Msekni <https://github.com/mseknibilel/OpenStack-Grizzly-Install-Guide>`_ 
的git仓库。向第一作者致敬！
感谢 天涯风情 QQ::519737673 的指导

内容列表
=================

::

  0. 简介
  1. 环境搭建
  2. 控制节点
  3. 网络节点
  4. 计算节点
  5. OpenStack使用 
  6. 参考文档


0. 简介
==============

OpenStack Grizzly 双网卡安装指南旨在让你轻松创建自己的OpenStack云平台。

状态: Stable


1. 环境搭建
====================

:节点角色: NICs
:控制节点: eth0 (192.168.8.51), eth1 (211.68.39.91)
:网络节点: eth0 (192.168.8.52), eth1 (211.68.39.92)
:计算节点: eth0 (192.168.8.53), eth1 (211.68.39.93)

**注意1:** 你总是可以使用dpkg -s <packagename>确认你使用的是grizzly软件包(版本: 2013.1)

**注意2:** 这个是当前网络架构

.. image:: http://i.imgur.com/NUC8Jdi.png

2. 控制节点
===============

2.1. 准备Ubuntu
-----------------

* 安装好Ubuntu 12.04 Server 64bits后, 进入sudo模式直到完成本指南::

   sudo su -

* 添加Grizzly仓库::

   apt-get install ubuntu-cloud-keyring python-software-properties software-properties-common python-keyring
   echo deb http://ubuntu-cloud.archive.canonical.com/ubuntu precise-updates/grizzly main >> /etc/apt/sources.list.d/grizzly.list
   #sed -i 's/us.archive.ubuntu.com/mirrors.4.tuna.tsinghua.edu.cn/g' /etc/apt/sources.list

* 升级系统::

   apt-get update

2.2.设置网络
------------

* 如下编辑网卡配置文件/etc/network/interfaces:: 

   #Not internet connected(used for OpenStack management)
   auto eth0
   iface eth0 inet static
   address 192.168.8.51
   netmask 255.255.255.0

   #For Exposing OpenStack API over the internet
   auto eth1
   iface eth1 inet static
   address 211.68.39.91
   netmask 255.255.255.192
   gateway 211.68.39.65
   dns-nameservers 8.8.8.8

* 重启网络服务::

   service networking restart

* 开启路由转发::

   sed -i 's/#net.ipv4.ip_forward=1/net.ipv4.ip_forward=1/' /etc/sysctl.conf
   sysctl -p

2.3. 安装MySQL
------------

* 安装MySQL并为root用户设置密码::

   apt-get install -y mysql-server python-mysqldb

* 配置mysql监听所有网络接口请求::

   sed -i 's/127.0.0.1/0.0.0.0/g' /etc/mysql/my.cnf
   service mysql restart

2.4. 安装RabbitMQ和NTP
------------

* 安装RabbitMQ::

   apt-get install -y rabbitmq-server 

* 安装NTP服务::

   apt-get install -y ntp

2.5. 创建数据库
------------

* 创建数据库::

   mysql -u root -p
   
   #Keystone
   CREATE DATABASE keystone;
   GRANT ALL ON keystone.* TO 'keystoneUser'@'%' IDENTIFIED BY 'keystonePass';
   
   #Glance
   CREATE DATABASE glance;
   GRANT ALL ON glance.* TO 'glanceUser'@'%' IDENTIFIED BY 'glancePass';

   #Quantum
   CREATE DATABASE quantum;
   GRANT ALL ON quantum.* TO 'quantumUser'@'%' IDENTIFIED BY 'quantumPass';

   #Nova
   CREATE DATABASE nova;
   GRANT ALL ON nova.* TO 'novaUser'@'%' IDENTIFIED BY 'novaPass';      

   #Cinder
   CREATE DATABASE cinder;
   GRANT ALL ON cinder.* TO 'cinderUser'@'%' IDENTIFIED BY 'cinderPass';

   quit;

2.6. 配置Keystone
------------

* 安装keystone软件包::

   apt-get install -y keystone

* 在/etc/keystone/keystone.conf中设置连接到新创建的数据库::

   nano /etc/keystone/keystone.conf
   connection = mysql://keystoneUser:keystonePass@192.168.8.51/keystone
   #sed -i '/connection = .*/{s|sqlite:///.*|mysql://'"keystoneUser"':'"keystonePass"'@'"192.168.8.51"'/keystone|g}' /etc/keystone/keystone.conf

* 重启身份认证服务并同步数据库::

   service keystone restart
   keystone-manage db_sync

* 使用git仓库中脚本填充keystone数据库： `脚本文件夹 <https://github.com/fmyzjs/OpenStack-Grizzly-Multi-2NICs-Install-Guide-CN/tree/master/KeystoneScripts>`_ ::

   #注意在执行脚本前请按你的网卡配置修改HOST_IP和HOST_IP_EXT

   wget https://raw.github.com/fmyzjs/OpenStack-Grizzly-Multi-2NICs-Install-Guide-CN/master/KeystoneScripts/keystone_basic.sh
   wget https://raw.github.com/fmyzjs/OpenStack-Grizzly-Multi-2NICs-Install-Guide-CN/master/KeystoneScripts/keystone_endpoints_basic.sh

   chmod +x keystone_basic.sh
   chmod +x keystone_endpoints_basic.sh

   nano keystone_basic.sh
   ./keystone_basic.sh
   nano keystone_endpoints_basic.sh
   ./keystone_endpoints_basic.sh

   +-------------+----------------------------------+
   |   Property  |              Value               |
   +-------------+----------------------------------+
   | description |    OpenStack Compute Service     |
   |      id     | abeeea7dfd334f4d97e9c93535939d70 |
   |     name    |               nova               |
   |     type    |             compute              |
   +-------------+----------------------------------+
   +-------------+----------------------------------+
   |   Property  |              Value               |
   +-------------+----------------------------------+
   | description |     OpenStack Volume Service     |
   |      id     | b15b82a0da6b4e0f93e6a78a74934507 |
   |     name    |              cinder              |
   |     type    |              volume              |
   +-------------+----------------------------------+
   +-------------+----------------------------------+
   |   Property  |              Value               |
   +-------------+----------------------------------+
   | description |     OpenStack Image Service      |
   |      id     | 07a1401347574aa98aec10bc12dbe8b0 |
   |     name    |              glance              |
   |     type    |              image               |
   +-------------+----------------------------------+
   +-------------+----------------------------------+
   |   Property  |              Value               |
   +-------------+----------------------------------+
   | description |        OpenStack Identity        |
   |      id     | cb854f3955da484aa83193a63040f3b3 |
   |     name    |             keystone             |
   |     type    |             identity             |
   +-------------+----------------------------------+
   +-------------+----------------------------------+
   |   Property  |              Value               |
   +-------------+----------------------------------+
   | description |      OpenStack EC2 service       |
   |      id     | 967fa49601e84adebae9f73814253897 |
   |     name    |               ec2                |
   |     type    |               ec2                |
   +-------------+----------------------------------+
   +-------------+----------------------------------+
   |   Property  |              Value               |
   +-------------+----------------------------------+
   | description |   OpenStack Networking service   |
   |      id     | c1d8d60ac84a4f2abbb76afa5c280155 |
   |     name    |             quantum              |
   |     type    |             network              |
   +-------------+----------------------------------+
   +-------------+-------------------------------------------+
   |   Property  |                   Value                   |
   +-------------+-------------------------------------------+
   |   adminurl  | http://192.168.8.51:8774/v2/$(tenant_id)s |
   |      id     |      c5862532eacb422d852e533057143b9c     |
   | internalurl | http://192.168.8.51:8774/v2/$(tenant_id)s |
   |  publicurl  | http://211.68.39.91:8774/v2/$(tenant_id)s |
   |    region   |                 RegionOne                 |
   |  service_id |      abeeea7dfd334f4d97e9c93535939d70     |
   +-------------+-------------------------------------------+
   +-------------+-------------------------------------------+
   |   Property  |                   Value                   |
   +-------------+-------------------------------------------+
   |   adminurl  | http://192.168.8.51:8776/v1/$(tenant_id)s |
   |      id     |      bee68f1b03e041689e5238f2d84ff92d     |
   | internalurl | http://192.168.8.51:8776/v1/$(tenant_id)s |
   |  publicurl  | http://211.68.39.91:8776/v1/$(tenant_id)s |
   |    region   |                 RegionOne                 |
   |  service_id |      b15b82a0da6b4e0f93e6a78a74934507     |
   +-------------+-------------------------------------------+
   +-------------+----------------------------------+
   |   Property  |              Value               |
   +-------------+----------------------------------+
   |   adminurl  |   http://192.168.8.51:9292/v2    |
   |      id     | db2419eeaf2244be97b5a234f3f75748 |
   | internalurl |   http://192.168.8.51:9292/v2    |
   |  publicurl  |   http://211.68.39.91:9292/v2    |
   |    region   |            RegionOne             |
   |  service_id | 07a1401347574aa98aec10bc12dbe8b0 |
   +-------------+----------------------------------+
   +-------------+----------------------------------+
   |   Property  |              Value               |
   +-------------+----------------------------------+
   |   adminurl  |  http://192.168.8.51:35357/v2.0  |
   |      id     | 7ea00ef7c10a473289532434b76cd8be |
   | internalurl |  http://192.168.8.51:5000/v2.0   |
   |  publicurl  |  http://211.68.39.91:5000/v2.0   |
   |    region   |            RegionOne             |
   |  service_id | cb854f3955da484aa83193a63040f3b3 |
   +-------------+----------------------------------+
   +-------------+-----------------------------------------+
   |   Property  |                  Value                  |
   +-------------+-----------------------------------------+
   |   adminurl  | http://192.168.8.51:8773/services/Admin |
   |      id     |     4b43b0181c3144b9b3596d9561fa46af    |
   | internalurl | http://192.168.8.51:8773/services/Cloud |
   |  publicurl  | http://211.68.39.91:8773/services/Cloud |
   |    region   |                RegionOne                |
   |  service_id |     967fa49601e84adebae9f73814253897    |
   +-------------+-----------------------------------------+
   +-------------+----------------------------------+
   |   Property  |              Value               |
   +-------------+----------------------------------+
   |   adminurl  |    http://192.168.8.51:9696/     |
   |      id     | 50dd072203434f59b90a490c8f5edcfb |
   | internalurl |    http://192.168.8.51:9696/     |
   |  publicurl  |    http://211.68.39.91:9696/     |
   |    region   |            RegionOne             |
   |  service_id | c1d8d60ac84a4f2abbb76afa5c280155 |
   +-------------+----------------------------------+

* 创建一个简单的凭据文件，这样稍后就不会因为输入过多的环境变量而感到厌烦::

   nano creds

   #Paste the following:
   export OS_TENANT_NAME=admin
   export OS_USERNAME=admin
   export OS_PASSWORD=admin_pass
   export OS_AUTH_URL="http://211.68.39.91:5000/v2.0/"

   # Load it:
   source creds

* 通过命令行列出Keystone中添加的用户::

   keystone user-list
   +----------------------------------+---------+---------+--------------------+
   |                id                |   name  | enabled |       email        |
   +----------------------------------+---------+---------+--------------------+
   | d93b2bf0e87e4bb7b67cb0dace65b56d |  admin  |   True  |  admin@domain.com  |
   | e9fc80f6b439478b94cad10e7e239878 |  cinder |   True  | cinder@domain.com  |
   | 25b101b31d2f40d78cdcc0654cfd2ab4 |  glance |   True  | glance@domain.com  |
   | b20580c0ecf041d588ddbfaa50f65abd |   nova  |   True  |  nova@domain.com   |
   | 81e6255cb957445e8fcab5048e435796 | quantum |   True  | quantum@domain.com |
   +----------------------------------+---------+---------+--------------------+

2.7. 设置Glance
------------

* 安装Glance::

   apt-get install -y glance

* 按下面更新/etc/glance/glance-api-paste.ini::

   [filter:authtoken]
   paste.filter_factory = keystoneclient.middleware.auth_token:filter_factory
   delay_auth_decision = true
   auth_host = 192.168.8.51
   auth_port = 35357
   auth_protocol = http
   admin_tenant_name = service
   admin_user = glance
   admin_password = service_pass

   #或者执行
   #cat <<EOF >> /etc/glance/glance-api-paste.ini
   #auth_host = 192.168.8.51
   #auth_port = 35357
   #auth_protocol = http
   #admin_tenant_name = service
   #admin_user = glance
   #EOF


* 按下面更新/etc/glance/glance-registry-paste.ini::

   [filter:authtoken]
   paste.filter_factory = keystoneclient.middleware.auth_token:filter_factory
   auth_host = 192.168.8.51
   auth_port = 35357
   auth_protocol = http
   admin_tenant_name = service
   admin_user = glance
   admin_password = service_pass

   #或者执行
   #cat <<EOF >> /etc/glance/glance-registry-paste.ini
   #auth_host = 192.168.8.51
   #auth_port = 35357
   #auth_protocol = http
   #admin_tenant_name = service
   #admin_user = glance
   #admin_password = service_pass
   #EOF



* 按下面更新/etc/glance/glance-api.conf::

   sql_connection = mysql://glanceUser:glancePass@192.168.8.51/glance
   #sed -i '/sql_connection = .*/{s|sqlite:///.*|mysql://'"glanceUser"':'"glancePass"'@'"192.168.8.51"'/glance|g}' /etc/glance/glance-api.conf

* 和::

   [paste_deploy]
   flavor = keystone

   #或者
   #echo flavor = keystone >> /etc/glance/glance-api.conf
   
* 按下面更新/etc/glance/glance-registry.conf::

   sql_connection = mysql://glanceUser:glancePass@192.168.8.51/glance
   #sed -i '/sql_connection = .*/{s|sqlite:///.*|mysql://'"glanceUser"':'"glancePass"'@'"192.168.8.51"'/glance|g}' /etc/glance/glance-registry.conf 



* 和::

   [paste_deploy]
   flavor = keystone
   #或者
   #echo flavor = keystone >> /etc/glance/glance-api.conf


* 重启glance-api和glance-registry服务::

   service glance-api restart; service glance-registry restart

* 同步glance数据库::

   glance-manage db_sync

* 重启服务使配置生效::

   service glance-registry restart; service glance-api restart

* 测试Glance, 从网络上传cirros云镜像::

   glance image-create --name cirros --is-public true --container-format bare --disk-format qcow2 --location https://launchpad.net/cirros/trunk/0.3.0/+download/cirros-0.3.0-x86_64-disk.img

   注意：通过此镜像创建的虚拟机可通过用户名/密码登陆， 用户名：cirros 密码：cubswin:)

* 本地创建Ubuntu云镜像::


   glance image-create --name myFirstImage --is-public true --container-format bare --disk-format qcow2 --location https://launchpad.net/cirros/trunk/0.3.0/+download/cirros-0.3.0-x86_64-disk.img

   +------------------+--------------------------------------+
   | Property         | Value                                |
   +------------------+--------------------------------------+
   | checksum         | None                                 |
   | container_format | bare                                 |
   | created_at       | 2013-06-01T02:58:03                  |
   | deleted          | False                                |
   | deleted_at       | None                                 |
   | disk_format      | qcow2                                |
   | id               | d69827a0-9ff2-4b38-b6c7-d6c047461904 |
   | is_public        | True                                 |
   | min_disk         | 0                                    |
   | min_ram          | 0                                    |
   | name             | myFirstImage                         |
   | owner            | a6ecf481397d4eab954af8318b336bfe     |
   | protected        | False                                |
   | size             | 9761280                              |
   | status           | active                               |
   | updated_at       | 2013-06-01T02:58:03                  |
   +------------------+--------------------------------------+


* 列出镜像检查是否上传成功::

   glance image-list

2.8. 设置Quantum
------------

* 安装Quantum组件::

   apt-get install -y quantum-server

* 编辑/etc/quantum/api-paste.ini ::

   [filter:authtoken]
   paste.filter_factory = keystoneclient.middleware.auth_token:filter_factory
   auth_host = 192.168.8.51
   auth_port = 35357
   auth_protocol = http
   admin_tenant_name = service
   admin_user = quantum
   admin_password = service_pass

* 编辑OVS配置文件/etc/quantum/plugins/openvswitch/ovs_quantum_plugin.ini:: 

   #Under the database section
   [DATABASE]
   sql_connection = mysql://quantumUser:quantumPass@192.168.8.51/quantum

   #sed -i '/sql_connection = .*/{s|sqlite:///.*|mysql://'"quantumUser"':'"quantumPass"'@'"192.168.8.51"'/quantum|g}'  /etc/quantum/plugins/openvswitch/ovs_quantum_plugin.ini

   #Under the OVS section
   [OVS]
   tenant_network_type = gre
   tunnel_id_ranges = 1:1000
   enable_tunneling = True

   #sed -i -e " s/# Example: tenant_network_type = gre/tenant_network_type = gre/g; 
                s/# Example: tunnel_id_ranges = 1:1000/tunnel_id_ranges = 1:1000/g; 
                s/# Default: enable_tunneling = False/enable_tunneling = True/g
              " /etc/quantum/plugins/openvswitch/ovs_quantum_plugin.ini 


   #Firewall driver for realizing quantum security group function
   [SECURITYGROUP]
   firewall_driver = quantum.agent.linux.iptables_firewall.OVSHybridIptablesFirewallDriver

   #sed -i 's/# firewall_driver = quantum.agent.linux.iptables_firewall.OVSHybridIptablesFirewallDriver/firewall_driver = quantum.agent.linux.iptables_firewall.OVSHybridIptablesFirewallDriver/g' /etc/quantum/plugins/openvswitch/ovs_quantum_plugin.ini

* 编辑/etc/quantum/quantum.conf::

   [keystone_authtoken]
   auth_host = 192.168.8.51
   auth_port = 35357
   auth_protocol = http
   admin_tenant_name = service
   admin_user = quantum
   admin_password = service_pass
   signing_dir = /var/lib/quantum/keystone-signing

   #或者
   sed -i -e " s/auth_host = 127.0.0.1/auth_host = 192.168.8.51/g; 
               s/%SERVICE_TENANT_NAME%/service/g; 
               s/%SERVICE_USER%/quantum/g; 
               s/%SERVICE_PASSWORD%/service_pass/g; 
             " /etc/quantum/quantum.conf


* 重启quantum所有服务::

   cd /etc/init.d/; for i in $( ls quantum-* ); do sudo service $i restart; done

2.9. 设置Nova
------------------

* 安装nova组件::

   apt-get install -y nova-api nova-cert novnc nova-consoleauth nova-scheduler nova-novncproxy nova-doc nova-conductor

* 在/etc/nova/api-paste.ini配置文件中修改认证信息::

   [filter:authtoken]
   paste.filter_factory = keystoneclient.middleware.auth_token:filter_factory
   auth_host = 192.168.8.51
   auth_port = 35357
   auth_protocol = http
   admin_tenant_name = service
   admin_user = nova
   admin_password = service_pass
   signing_dirname = /tmp/keystone-signing-nova
   # Workaround for https://bugs.launchpad.net/nova/+bug/1154809
   auth_version = v2.0

   #或者

   sed -i -e " s/auth_host = 127.0.0.1/auth_host = 192.168.8.51/g; 
               s/%SERVICE_TENANT_NAME%/service/g; s/%SERVICE_USER%/nova/g; 
               s/%SERVICE_PASSWORD%/service_pass/g; 
             " /etc/nova/api-paste.ini


* 如下修改/etc/nova/nova.conf::

   [DEFAULT] 
   logdir=/var/log/nova
   state_path=/var/lib/nova
   lock_path=/run/lock/nova
   verbose=True
   api_paste_config=/etc/nova/api-paste.ini
   compute_scheduler_driver=nova.scheduler.simple.SimpleScheduler
   rabbit_host=192.168.8.51
   nova_url=http://192.168.8.51:8774/v1.1/
   sql_connection=mysql://novaUser:novaPass@192.168.8.51/nova
   root_helper=sudo nova-rootwrap /etc/nova/rootwrap.conf

   # Auth
   use_deprecated_auth=false
   auth_strategy=keystone

   # Imaging service
   glance_api_servers=192.168.8.51:9292
   image_service=nova.image.glance.GlanceImageService

   # Vnc configuration
   novnc_enabled=true
   novncproxy_base_url=http://211.68.39.91:6080/vnc_auto.html
   novncproxy_port=6080
   vncserver_proxyclient_address=192.168.8.51
   vncserver_listen=0.0.0.0

   # Network settings
   network_api_class=nova.network.quantumv2.api.API
   quantum_url=http://192.168.8.51:9696
   quantum_auth_strategy=keystone
   quantum_admin_tenant_name=service
   quantum_admin_username=quantum
   quantum_admin_password=service_pass
   quantum_admin_auth_url=http://192.168.8.51:35357/v2.0
   libvirt_vif_driver=nova.virt.libvirt.vif.LibvirtHybridOVSBridgeDriver
   linuxnet_interface_driver=nova.network.linux_net.LinuxOVSInterfaceDriver
   #If you want Quantum + Nova Security groups
   firewall_driver=nova.virt.firewall.NoopFirewallDriver
   security_group_api=quantum
   #If you want Nova Security groups only, comment the two lines above and uncomment line -1-.
   #-1-firewall_driver=nova.virt.libvirt.firewall.IptablesFirewallDriver

   #Metadata
   service_quantum_metadata_proxy = True
   quantum_metadata_proxy_shared_secret = helloOpenStack

   # Compute #
   compute_driver=libvirt.LibvirtDriver

   # Cinder #
   volume_api_class=nova.volume.cinder.API
   osapi_volume_listen_port=5900
 
* 同步数据库::

   nova-manage db sync

* 重启所有nova服务::

   cd /etc/init.d/; for i in $( ls nova-* ); do sudo service $i restart; done   

* 检查所有nova服务是否启动正常::

   nova-manage service list
   Binary           Host                                 Zone             Status     State Updated_At
   nova-cert        91openstack                          internal         enabled    :-)   2013-06-01 03:10:51
   nova-conductor   91openstack                          internal         enabled    :-)   2013-06-01 03:10:51
   nova-scheduler   91openstack                          internal         enabled    :-)   2013-06-01 03:10:51
   nova-consoleauth 91openstack                          internal         enabled    :-)   2013-06-01 03:10:51


2.10. 设置Cinder
------------------

* 安装软件包::

   apt-get install -y cinder-api cinder-scheduler cinder-volume iscsitarget open-iscsi iscsitarget-dkms

* 配置iscsi服务::

   sed -i 's/false/true/g' /etc/default/iscsitarget

* 重启服务::
   
   service iscsitarget start
   service open-iscsi start

* 如下配置/etc/cinder/api-paste.ini::

   [filter:authtoken]
   paste.filter_factory = keystoneclient.middleware.auth_token:filter_factory
   service_protocol = http
   service_host = 211.68.39.91
   service_port = 5000
   auth_host = 192.168.8.51
   auth_port = 35357
   auth_protocol = http
   admin_tenant_name = service
   admin_user = cinder
   admin_password = service_pass

   #或者
   sed -i -e " s/service_host = 127.0.0.1/service_host = 211.68.39.91/g; 
               s/auth_host = 127.0.0.1/auth_host = 192.168.8.51/g; 
               s/%SERVICE_TENANT_NAME%/service/g; 
               s/%SERVICE_USER%/cinder/g; 
               s/%SERVICE_PASSWORD%/service_pass/g; 
             " /etc/cinder/api-paste.ini


* 编辑/etc/cinder/cinder.conf::

   [DEFAULT]
   rootwrap_config=/etc/cinder/rootwrap.conf
   sql_connection = mysql://cinderUser:cinderPass@192.168.8.51/cinder
   api_paste_config = /etc/cinder/api-paste.ini
   iscsi_helper=ietadm
   volume_name_template = volume-%s
   volume_group = cinder-volumes
   verbose = True
   auth_strategy = keystone
   #osapi_volume_listen_port=5900

   #其实只要插入以下内容
   cat <<EOF >> /etc/cinder/cinder.conf
   sql_connection = mysql://cinderUser:cinderPass@192.168.8.51/cinder
   iscsi_ip_address=192.168.8.51
   EOF


* 接下来同步数据库::

   cinder-manage db sync

   #输出
   2013-06-01 11:16:11     INFO [migrate.versioning.api] 0 -> 1...
   2013-06-01 11:16:12     INFO [migrate.versioning.api] done
   2013-06-01 11:16:12     INFO [migrate.versioning.api] 1 -> 2...
   2013-06-01 11:16:13     INFO [migrate.versioning.api] done
   2013-06-01 11:16:13     INFO [migrate.versioning.api] 2 -> 3...
   2013-06-01 11:16:13     INFO [migrate.versioning.api] done
   2013-06-01 11:16:13     INFO [migrate.versioning.api] 3 -> 4...
   2013-06-01 11:16:13     INFO [004_volume_type_to_uuid] Created foreign key volume_type_extra_specs_ibfk_1
   2013-06-01 11:16:13     INFO [migrate.versioning.api] done
   2013-06-01 11:16:13     INFO [migrate.versioning.api] 4 -> 5...
   2013-06-01 11:16:13     INFO [migrate.versioning.api] done
   2013-06-01 11:16:13     INFO [migrate.versioning.api] 5 -> 6...
   2013-06-01 11:16:14     INFO [migrate.versioning.api] done
   2013-06-01 11:16:14     INFO [migrate.versioning.api] 6 -> 7...
   2013-06-01 11:16:14     INFO [migrate.versioning.api] done
   2013-06-01 11:16:14     INFO [migrate.versioning.api] 7 -> 8...
   2013-06-01 11:16:14     INFO [migrate.versioning.api] done
   2013-06-01 11:16:14     INFO [migrate.versioning.api] 8 -> 9...
   2013-06-01 11:16:14     INFO [migrate.versioning.api] done


* 最后别忘了创建一个卷组命名为cinder-volumes::

   dd if=/dev/zero of=cinder-volumes bs=1 count=0 seek=2G
   losetup /dev/loop2 cinder-volumes
   fdisk /dev/loop2
   #Type in the followings:
   n
   p
   1
   ENTER
   ENTER
   t
   8e
   w

* 创建物理卷和卷组::

   pvcreate /dev/loop2
   vgcreate cinder-volumes /dev/loop2

**注意:** 重启后卷组不会自动挂载 (点击 `这个 <https://github.com/mseknibilel/OpenStack-Folsom-Install-guide/blob/master/Tricks%26Ideas/load_volume_group_after_system_reboot.rst>`_  设置在重启后自动挂载) 

* 重启cinder服务::

   cd /etc/init.d/; for i in $( ls cinder-* ); do sudo service $i restart; done

* 确认cinder服务在运行::

   cd /etc/init.d/; for i in $( ls cinder-* ); do sudo service $i status; done

2.11. 设置Horizon
------------------

* 如下安装horizon ::

   apt-get install -y openstack-dashboard memcached

* 如果你不喜欢OpenStack ubuntu主题, 你可以停用它::

   dpkg --purge openstack-dashboard-ubuntu-theme

* 重启Apache和memcached服务::

   service apache2 restart; service memcached restart

3. 网络节点
================

3.1. 准备节点
-----------------

* 安装好Ubuntu 12.04 Server 64bits后, 进入sudo模式直到完成本指南::

   sudo su -

* 添加Grizzly仓库::

   apt-get install ubuntu-cloud-keyring python-software-properties software-properties-common python-keyring
   echo deb http://ubuntu-cloud.archive.canonical.com/ubuntu precise-updates/grizzly main >> /etc/apt/sources.list.d/grizzly.list
   sed -i 's/us.archive.ubuntu.com/mirrors.4.tuna.tsinghua.edu.cn/g' /etc/apt/sources.list


* 升级系统::

   apt-get update

* 安装ntp服务::

   apt-get install -y ntp

* 配置ntp服务从控制节点同步时间::

   #Comment the ubuntu NTP servers
   sed -i 's/server 0.ubuntu.pool.ntp.org/#server 0.ubuntu.pool.ntp.org/g' /etc/ntp.conf
   sed -i 's/server 1.ubuntu.pool.ntp.org/#server 1.ubuntu.pool.ntp.org/g' /etc/ntp.conf
   sed -i 's/server 2.ubuntu.pool.ntp.org/#server 2.ubuntu.pool.ntp.org/g' /etc/ntp.conf
   sed -i 's/server 3.ubuntu.pool.ntp.org/#server 3.ubuntu.pool.ntp.org/g' /etc/ntp.conf
   
   #Set the network node to follow up your conroller node
   sed -i 's/server ntp.ubuntu.com/server 192.168.8.51/g' /etc/ntp.conf

   service ntp restart
* 安装vlan bridge-utils::
   
   apt-get install -y vlan bridge-utils

3.2. 配置网络
-----------------

* 2块网卡如下设置::

   auto eth1
   iface eth1 inet static
           address 211.68.39.92
           netmask 255.255.255.192
           gateway 211.68.39.65
           dns-nameservers 8.8.8.8
   #Not internet connected(used for OpenStack management)
   auto eth0
   iface eth0 inet static
           address 192.168.8.52
           netmask 255.255.255.0

   /etc/init.d/networking restart


* 开启路由转发::

   sed -i 's/#net.ipv4.ip_forward=1/net.ipv4.ip_forward=1/' /etc/sysctl.conf
   sysctl -p


3.3. OpenVSwitch
------------

* 安装openVSwitch::

   apt-get install -y openvswitch-switch openvswitch-datapath-dkms

* 建立网桥::

   #br-int will be used for VM integration
   ovs-vsctl add-br br-int
   #br-ex is used to make to VM accessible from the internet
   ovs-vsctl add-br br-ex

3.4. Quantum-*
------------

* 安装Quantum组件::

   apt-get -y install quantum-plugin-openvswitch-agent quantum-dhcp-agent quantum-l3-agent quantum-metadata-agent

* 编辑/etc/quantum/api-paste.ini ::

   [filter:authtoken]
   paste.filter_factory = keystoneclient.middleware.auth_token:filter_factory
   auth_host = 192.168.8.51
   auth_port = 35357
   auth_protocol = http
   admin_tenant_name = service
   admin_user = quantum
   admin_password = service_pass

* 编辑OVS配置文件/etc/quantum/plugins/openvswitch/ovs_quantum_plugin.ini:: 

   #Under the database section
   [DATABASE]
   sql_connection = mysql://quantumUser:quantumPass@192.168.8.51/quantum

   #sed -i '/sql_connection = .*/{s|sqlite:///.*|mysql://'"quantumUser"':'"quantumPass"'@'"192.168.8.51"'/quantum|g}'  /etc/quantum/plugins/openvswitch/ovs_quantum_plugin.ini

   #Under the OVS section
   [OVS]
   tenant_network_type = gre
   tunnel_id_ranges = 1:1000
   integration_bridge = br-int
   tunnel_bridge = br-tun
   local_ip = 192.168.8.52
   enable_tunneling = True

   #或者执行
   sed -i -e " s/# Example: tenant_network_type = gre/tenant_network_type = gre/g; 
               s/# Example: tunnel_id_ranges = 1:1000/tunnel_id_ranges = 1:1000/g; 
               s/# Default: integration_bridge = br-int/integration_bridge = br-int/g; 
               s/# Default: tunnel_bridge = br-tun/tunnel_bridge = br-tun/g; 
               s/# Default: local_ip =/local_ip = 192.168.8.52/g; 
               s/# Default: enable_tunneling = False/enable_tunneling = True/g
             " /etc/quantum/plugins/openvswitch/ovs_quantum_plugin.ini 


   #Firewall driver for realizing quantum security group function
   [SECURITYGROUP]
   firewall_driver = quantum.agent.linux.iptables_firewall.OVSHybridIptablesFirewallDriver

   #sed -i 's/# firewall_driver = quantum.agent.linux.iptables_firewall.OVSHybridIptablesFirewallDriver/firewall_driver = quantum.agent.linux.iptables_firewall.OVSHybridIptablesFirewallDriver/g' /etc/quantum/plugins/openvswitch/ovs_quantum_plugin.ini


* 更新/etc/quantum/metadata_agent.ini::

   # The Quantum user information for accessing the Quantum API.
   auth_url = http://192.168.8.51:35357/v2.0
   auth_region = RegionOne
   admin_tenant_name = service
   admin_user = quantum
   admin_password = service_pass

   # IP address used by Nova metadata server
   nova_metadata_ip = 192.168.8.51

   # TCP Port used by Nova metadata server
   nova_metadata_port = 8775

   metadata_proxy_shared_secret = helloOpenStack

   #或者执行
   #sed -i -e " s/localhost/192.168.8.51/g; 
                s/%SERVICE_TENANT_NAME%/service/g; 
                s/%SERVICE_USER%/quantum/g; 
                s/%SERVICE_PASSWORD%/service_pass/g; 
                s/# nova_metadata_ip = 127.0.0.1/nova_metadata_ip = 192.168.8.51/g; 
                s/# nova_metadata_port = 8775/nova_metadata_port = 8775/g; 
                s/# metadata_proxy_shared_secret =/metadata_proxy_shared_secret = helloOpenStack/g
              " /etc/quantum/metadata_agent.ini


* 编辑/etc/quantum/quantum.conf::

   # 确保RabbitMQ IP指向了控制节点
   rabbit_host = 192.168.8.51

   [keystone_authtoken]
   auth_host = 192.168.8.51
   auth_port = 35357
   auth_protocol = http
   admin_tenant_name = service
   admin_user = quantum
   admin_password = service_pass
   signing_dir = /var/lib/quantum/keystone-signing

   #或者执行
   sed -i -e " s/# rabbit_host = localhost/rabbit_host = 192.168.8.51/g; 
               s/auth_host = 127.0.0.1/auth_host = 192.168.8.51/g; 
               s/%SERVICE_TENANT_NAME%/service/g; 
               s/%SERVICE_USER%/quantum/g; 
               s/%SERVICE_PASSWORD%/service_pass/g 
             " /etc/quantum/quantum.conf


* 编辑 /etc/sudoers::

   nano /etc/sudoers.d/quantum_sudoers
   
   #Modify the quantum user
   quantum ALL=NOPASSWD: ALL


* 重启quantum所有服务::

   cd /etc/init.d/; for i in $( ls quantum-* ); do sudo service $i restart; done

3.4. OpenVSwitch (Part2)
------------------
* 将内部外部网卡加入br-ex并清除外部网卡的IP::
   ovs-vsctl  add-port  br-ex  eth1
   ifconfig  eth1  0
   ifconfig  br-ex  211.68.39.92  netmask  255.255.255.192
   route  add  default  gw  211.68.39.65
*上面的设置在重启电脑后配置就会无效，要想重启有效，就写入配置文件/etc/network/interfaces（这样修改后，启动后br-ex和eth1是满足要求了，但是启动的虚拟机又无法ping通，解决办法是：将上述命令写入脚本文件，然后再链接到rc2.d（ln –s XXX.sh /etc/rc2.d/S99XX）中，开机后执行脚本，这样就可以解决了）


4. 计算节点
================

4.1. 准备节点
-----------------

* 安装好Ubuntu 12.04 Server 64bits后, 进入sudo模式直到完成本指南::

   sudo su 

* 添加Grizzly仓库::

   apt-get install ubuntu-cloud-keyring python-software-properties software-properties-common python-keyring
   echo deb http://ubuntu-cloud.archive.canonical.com/ubuntu precise-updates/grizzly main >> /etc/apt/sources.list.d/grizzly.list
   sed -i 's/us.archive.ubuntu.com/mirrors.4.tuna.tsinghua.edu.cn/g' /etc/apt/sources.list

* 升级::

   apt-get update


* 安装ntp服务::

   apt-get install -y ntp

* 配置ntp服务从控制节点同步时间::

   #Comment the ubuntu NTP servers
   sed -i 's/server 0.ubuntu.pool.ntp.org/#server 0.ubuntu.pool.ntp.org/g' /etc/ntp.conf
   sed -i 's/server 1.ubuntu.pool.ntp.org/#server 1.ubuntu.pool.ntp.org/g' /etc/ntp.conf
   sed -i 's/server 2.ubuntu.pool.ntp.org/#server 2.ubuntu.pool.ntp.org/g' /etc/ntp.conf
   sed -i 's/server 3.ubuntu.pool.ntp.org/#server 3.ubuntu.pool.ntp.org/g' /etc/ntp.conf
   
   #Set the network node to follow up your conroller node
   sed -i 's/server ntp.ubuntu.com/server 192.168.8.51/g' /etc/ntp.conf

   service ntp restart


4.2. 配置网络
-----------------

* 如下配置网络::

   auto eth1
   iface eth1 inet static
           address 211.68.39.93
           netmask 255.255.255.192
           network 211.68.39.64
           broadcast 211.68.39.127
           gateway 211.68.39.65
           # dns-* options are implemented by the resolvconf package, if installed
           dns-nameservers 8.8.8.8
   auto eth0
   iface eth0 inet static
           address 192.168.8.53
           netmask 255.255.255.0

   /etc/init.d/networking restart


* 开启路由转发::

   sed -i 's/#net.ipv4.ip_forward=1/net.ipv4.ip_forward=1/' /etc/sysctl.conf
   sysctl -p

4.3. KVM
------------------

* 确保你的硬件启用virtualization::

   apt-get install cpu-checker
   kvm-ok

* 现在安装kvm并配置它::

   apt-get install -y kvm libvirt-bin pm-utils

* 在/etc/libvirt/qemu.conf配置文件中启用cgroup_device_acl数组::

   cgroup_device_acl = [
   "/dev/null", "/dev/full", "/dev/zero",
   "/dev/random", "/dev/urandom",
   "/dev/ptmx", "/dev/kvm", "/dev/kqemu",
   "/dev/rtc", "/dev/hpet","/dev/net/tun"
   ]

* 删除默认的虚拟网桥::

   virsh net-destroy default
   virsh net-undefine default

* 更新/etc/libvirt/libvirtd.conf配置文件::

   listen_tls = 0
   listen_tcp = 1
   auth_tcp = "none"
   
   #或者执行
   sed -i -e " s/#listen_tls = 0/listen_tls = 0/g; 
               s/#listen_tcp = 1/listen_tcp = 1/g; 
               s/#auth_tcp = \"sasl\"/auth_tcp = \"none\"/g 
             " /etc/libvirt/libvirtd.conf


* E编辑libvirtd_opts变量在/etc/init/libvirt-bin.conf配置文件中::

   env libvirtd_opts="-d -l"

   #sed -i 's/env libvirtd_opts="-d"/env libvirtd_opts="-d -l"/g' /etc/init/libvirt-bin.conf

* 编辑/etc/default/libvirt-bin文件 ::

   libvirtd_opts="-d -l"

   #sed -i 's/libvirtd_opts="-d"/libvirtd_opts="-d -l"/g' /etc/default/libvirt-bin


* 重启libvirt服务使配置生效::

   service libvirt-bin restart

4.4. OpenVSwitch
------------------

* 安装OpenVSwitch软件包::

   apt-get install -y openvswitch-switch openvswitch-datapath-dkms

* 创建网桥::

   #br-int will be used for VM integration
   ovs-vsctl add-br br-int

4.5. Quantum
------------------

* 安装Quantum openvswitch代理::

   apt-get -y install quantum-plugin-openvswitch-agent

* 编辑OVS配置文件/etc/quantum/plugins/openvswitch/ovs_quantum_plugin.ini:: 

   #Under the database section
   [DATABASE]
   sql_connection = mysql://quantumUser:quantumPass@192.168.8.51/quantum

   #sed -i '/sql_connection = .*/{s|sqlite:///.*|mysql://'"quantumUser"':'"quantumPass"'@'"192.168.8.51"'/quantum|g}'  /etc/quantum/plugins/openvswitch/ovs_quantum_plugin.ini

   #Under the OVS section
   [OVS]
   tenant_network_type = gre
   tunnel_id_ranges = 1:1000
   integration_bridge = br-int
   tunnel_bridge = br-tun
   local_ip = 192.168.8.51
   enable_tunneling = True


   #或者执行
   sed -i -e " s/# Example: tenant_network_type = gre/tenant_network_type = gre/g; 
               s/# Example: tunnel_id_ranges = 1:1000/tunnel_id_ranges = 1:1000/g; 
               s/# Default: integration_bridge = br-int/integration_bridge = br-int/g; 
               s/# Default: tunnel_bridge = br-tun/tunnel_bridge = br-tun/g; 
               s/# Default: local_ip =/local_ip = 192.168.8.53/g; s/# Default: enable_tunneling = False/enable_tunneling = True/g
             " /etc/quantum/plugins/openvswitch/ovs_quantum_plugin.ini 


   #Firewall driver for realizing quantum security group function
   [SECURITYGROUP]
   firewall_driver = quantum.agent.linux.iptables_firewall.OVSHybridIptablesFirewallDriver

   #sed -i 's/# firewall_driver = quantum.agent.linux.iptables_firewall.OVSHybridIptablesFirewallDriver/firewall_driver = quantum.agent.linux.iptables_firewall.OVSHybridIptablesFirewallDriver/g' /etc/quantum/plugins/openvswitch/ovs_quantum_plugin.ini


* 编辑/etc/quantum/quantum.conf::

   # 确保RabbitMQ IP指向了控制节点
   rabbit_host = 192.168.8.51

   [keystone_authtoken]
   auth_host = 192.168.8.51
   auth_port = 35357
   auth_protocol = http
   admin_tenant_name = service
   admin_user = quantum
   admin_password = service_pass
   signing_dir = /var/lib/quantum/keystone-signing

   #或者执行
   sed -i -e " s/# rabbit_host = localhost/rabbit_host = 192.168.8.51/g; 
               s/auth_host = 127.0.0.1/auth_host = 192.168.8.51/g; 
               s/%SERVICE_TENANT_NAME%/service/g; 
               s/%SERVICE_USER%/quantum/g; s/%SERVICE_PASSWORD%/service_pass/g 
             " /etc/quantum/quantum.conf


* 重启Quantum openvswitch代理服务::

   service quantum-plugin-openvswitch-agent restart

4.6. Nova
------------------

* 安装nova组件::

   apt-get install -y nova-compute-kvm

   注意：如果你的宿主机不支持kvm虚拟化，可把nova-compute-kvm换成nova-compute-qemu
   同时/etc/nova/nova-compute.conf配置文件中的libvirt_type=qemu

* 在/etc/nova/api-paste.ini配置文件中修改认证信息::

   [filter:authtoken]
   paste.filter_factory = keystoneclient.middleware.auth_token:filter_factory
   auth_host = 192.168.8.51
   auth_port = 35357
   auth_protocol = http
   admin_tenant_name = service
   admin_user = nova
   admin_password = service_pass
   signing_dirname = /tmp/keystone-signing-nova
   # Workaround for https://bugs.launchpad.net/nova/+bug/1154809
   auth_version = v2.0

   

   #或者执行
   sed -i -e " s/auth_host = 127.0.0.1/auth_host = 192.168.8.51/g; 
               s/%SERVICE_TENANT_NAME%/service/g; 
               s/%SERVICE_USER%/nova/g; 
               s/%SERVICE_PASSWORD%/service_pass/g; 
             " /etc/nova/api-paste.ini


* 如下修改/etc/nova/nova.conf::

   [DEFAULT]
   logdir=/var/log/nova
   state_path=/var/lib/nova
   lock_path=/run/lock/nova
   verbose=True
   api_paste_config=/etc/nova/api-paste.ini
   compute_scheduler_driver=nova.scheduler.simple.SimpleScheduler
   rabbit_host=192.168.8.51
   nova_url=http://192.168.8.51:8774/v1.1/
   sql_connection=mysql://novaUser:novaPass@192.168.8.51/nova
   root_helper=sudo nova-rootwrap /etc/nova/rootwrap.conf

   # Auth
   use_deprecated_auth=false
   auth_strategy=keystone

   # Imaging service
   glance_api_servers=192.168.8.51:9292
   image_service=nova.image.glance.GlanceImageService

   # Vnc configuration
   novnc_enabled=true
   novncproxy_base_url=http://211.68.39.91:6080/vnc_auto.html
   novncproxy_port=6080
   vncserver_proxyclient_address=192.168.8.53 #此处填写本节点IP
   vncserver_listen=0.0.0.0
   
   # Metadata
   service_quantum_metadata_proxy = True
   quantum_metadata_proxy_shared_secret = helloOpenStack
   
   # Network settings
   network_api_class=nova.network.quantumv2.api.API
   quantum_url=http://192.168.8.51:9696
   quantum_auth_strategy=keystone
   quantum_admin_tenant_name=service
   quantum_admin_username=quantum
   quantum_admin_password=service_pass
   quantum_admin_auth_url=http://192.168.8.51:35357/v2.0
   libvirt_vif_driver=nova.virt.libvirt.vif.QuantumLinuxBridgeVIFDriver
   linuxnet_interface_driver=nova.network.linux_net.LinuxBridgeInterfaceDriver
   firewall_driver=nova.virt.libvirt.firewall.IptablesFirewallDriver

   # Compute #
   compute_driver=libvirt.LibvirtDriver
  
   # Cinder #
   volume_api_class=nova.volume.cinder.API
   osapi_volume_listen_port=5900



* 修改/etc/nova/nova-compute.conf::

   [DEFAULT]
   libvirt_type=kvm
   compute_driver=libvirt.LibvirtDriver
   libvirt_ovs_bridge=br-int
   libvirt_vif_type=ethernet
   libvirt_vif_driver=nova.virt.libvirt.vif.LibvirtHybridOVSBridgeDriver
   libvirt_use_virtio_for_bridges=True

* 重启所有nova服务::

   cd /etc/init.d/; for i in $( ls nova-* ); do sudo service $i restart; done   

* 检查所有nova服务是否启动正常::

   nova-manage service list

5. OpenStack使用
================

网络拓扑如下：

.. image:: http://i.imgur.com/OhcrgKy.jpg

5.1. 为admin租户创建内网、外网、路由器和虚拟机
------------------

* 设置环境变量::

   # cat creds-admin

   export OS_TENANT_NAME=admin
   export OS_USERNAME=admin
   export OS_PASSWORD=admin_pass
   export OS_AUTH_URL="http://211.68.39.91:5000/v2.0/"

* 使环境变量生效::

   # source creds-admin


* 列出已创建的租户::

   # keystone tenant-list

   +----------------------------------+---------+---------+
   |                id                |   name  | enabled |
   +----------------------------------+---------+---------+
   | a6ecf481397d4eab954af8318b336bfe |  admin  |   True  |
   | 6a51172646bc4e06b74c5e81f09f2cce | service |   True  |
   +----------------------------------+---------+---------+

* 为admin租户创建网络::

   # quantum net-create --tenant-id a6ecf481397d4eab954af8318b336bfe admin_int

   Created a new network:
   +---------------------------+--------------------------------------+
   | Field                     | Value                                |
   +---------------------------+--------------------------------------+
   | admin_state_up            | True                                 |
   | id                        | 80598fc8-0b91-47ff-8b39-ebd2a3c3661c |
   | name                      | admin_int                            |
   | provider:network_type     | gre                                  |
   | provider:physical_network |                                      |
   | provider:segmentation_id  | 1                                    |
   | router:external           | False                                |
   | shared                    | False                                |
   | status                    | ACTIVE                               |
   | subnets                   |                                      |
   | tenant_id                 | a6ecf481397d4eab954af8318b336bfe     |
   +---------------------------+--------------------------------------+

# 为admin租户创建子网::

   # quantum subnet-create --tenant-id a6ecf481397d4eab954af8318b336bfe admin_int 192.168.8.0/24 

   Created a new subnet:
   +------------------+--------------------------------------------------+
   | Field            | Value                                            |
   +------------------+--------------------------------------------------+
   | allocation_pools | {"start": "192.168.8.2", "end": "192.168.8.254"} |
   | cidr             | 192.168.8.0/24                                   |
   | dns_nameservers  |                                                  |
   | enable_dhcp      | True                                             |
   | gateway_ip       | 192.168.8.1                                      |
   | host_routes      |                                                  |
   | id               | 3c9e0733-32e0-43ad-9fde-3b75259ff39e             |
   | ip_version       | 4                                                |
   | name             |                                                  |
   | network_id       | 80598fc8-0b91-47ff-8b39-ebd2a3c3661c             |
   | tenant_id        | a6ecf481397d4eab954af8318b336bfe                 |
   +------------------+--------------------------------------------------+

* 为admin租户创建路由器::

   # quantum router-create --tenant-id a6ecf481397d4eab954af8318b336bfe router_admin

   Created a new router:
   +-----------------------+--------------------------------------+
   | Field                 | Value                                |
   +-----------------------+--------------------------------------+
   | admin_state_up        | True                                 |
   | external_gateway_info |                                      |
   | id                    | 9cbc5687-7bf9-4145-89a1-cf614e377f7a |
   | name                  | router_admin                         |
   | status                | ACTIVE                               |
   | tenant_id             | a6ecf481397d4eab954af8318b336bfe     |
   +-----------------------+--------------------------------------+

* 把路由加入子网::

   # quantum router-interface-add 9cbc5687-7bf9-4145-89a1-cf614e377f7a 3c9e0733-32e0-43ad-9fde-3b75259ff39e   



* 创建外部网络::

   # quantum net-create --tenant-id 6a51172646bc4e06b74c5e81f09f2cce ext_net --router:external=True


   # Note： $id_of_service_tenant 来自租户“service”，可用keystone tenant-list 查看获取；

   Created a new network:
   +---------------------------+--------------------------------------+
   | Field                     | Value                                |
   +---------------------------+--------------------------------------+
   | admin_state_up            | True                                 |
   | id                        | 20cd57e4-f4b5-4c1d-a96d-0ca0a01b2b06 |
   | name                      | ext_net                              |
   | provider:network_type     | gre                                  |
   | provider:physical_network |                                      |
   | provider:segmentation_id  | 2                                    |
   | router:external           | True                                 |
   | shared                    | False                                |
   | status                    | ACTIVE                               |
   | subnets                   |                                      |
   | tenant_id                 | 6a51172646bc4e06b74c5e81f09f2cce     |
   +---------------------------+--------------------------------------+

* 创建外网用子网211.68.39.64/26::

   # quantum subnet-create --tenant-id 6a51172646bc4e06b74c5e81f09f2cce --allocation-pool start=211.68.39.68,end=211.68.39.74 --gateway 211.68.39.65 ext_net 211.68.39.64/26 --enable_dhcp=False


   Created a new subnet:
   +------------------+--------------------------------------------------+
   | Field            | Value                                            |
   +------------------+--------------------------------------------------+
   | allocation_pools | {"start": "211.68.39.68", "end": "211.68.39.74"} |
   | cidr             | 211.68.39.64/26                                  |
   | dns_nameservers  |                                                  |
   | enable_dhcp      | False                                            |
   | gateway_ip       | 211.68.39.65                                     |
   | host_routes      |                                                  |
   | id               | 6270aa2e-a923-495b-a397-0bdeb01ea795             |
   | ip_version       | 4                                                |
   | name             |                                                  |
   | network_id       | 20cd57e4-f4b5-4c1d-a96d-0ca0a01b2b06             |
   | tenant_id        | 6a51172646bc4e06b74c5e81f09f2cce                 |
   +------------------+--------------------------------------------------+



* 关联外网和admin的路由::

   # quantum router-gateway-set 9cbc5687-7bf9-4145-89a1-cf614e377f7a 20cd57e4-f4b5-4c1d-a96d-0ca0a01b2b06


6. 联系
===========

   fmyzjs  : fmyzjs@gmail.com

7. 参考
=================

This work has been based on:

* Bilel Msekni's Folsom Install guide [https://github.com/mseknibilel/OpenStack-Folsom-Install-guide]
* OpenStack Grizzly Install Guide (Master Branch) [https://github.com/mseknibilel/OpenStack-Grizzly-Install-Guide]
* Ubuntu13.04安装多机Grizzly版本的OpenStack35 [http://wenku.baidu.com/view/3b428f38bd64783e09122bf4?fr=prin]
*OpenStack Grizzly-Install Guide CN [https://github.com/ist0ne/OpenStack-Grizzly-Install-Guide-CN/blob/OVS_MutliNode/OpenStack_Grizzly_Install_Guide.rst]