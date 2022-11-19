
-----

ansible是目前业内比较流行的自动化部署工具，这里我们实现了基于ansible的pollybase自动化部署，用户只需要简单的配置，即可快速创建pollybase集群。

### 一、准备工作

ansible只需要在一台机器上安装，就可以控制多台其它服务器，其它服务器不需要安装任何agent，底层通信是基于ssh。

安装ansible的这台机器我们称作主控机，对主控机的硬件配置没有任何要求，可以是虚拟机，也可以是用来部署pollybase集群的服务器当中任意一个，安装ansible的主控机需要`外网通信`
####1、主控机root账户与待部署机器的ssh免密
为了方便，需要这台主控机与要安装`pollybase`的所有服务器节点`ssh免密`。
因为ansible在部署安装服务的时候， 还需要创建账户，设置sudo权限等，因此需要建立主控机与其它机器的`root`账户`ssh免密登录`权限。

如果主控机和其它机器都是新装的操作系统，没有使用过，最简单的ssh免密登录方式是这样的：
在`主控机`的`root`账户执行：
```
ssh-keygen -t rsa
```
命令提示全部回车确认，然后会在`/root`目录下生成`.ssh`目录

如果您需要从主控机免密ssh到任何一台服务器(例如192.168.239.136)，您需要执行下面的命令：
```
ssh-copy-id 192.168.239.136
```
此时需要您输入192.168.239.136的root账户的密码，完成后，您执行下面的命令，将不会再要求您输入密码
```
ssh 192.168.239.136 "echo ok"
```
这就实现了主控机root账户免密ssh到192.168.239.136的功能，其余待部署节点，执行同样的`ssh-copy-id`命令即可。

**如果无法实现root账户的免密登录，那么可以使用普通账户，具体操作流程：**
1）在主控机和全部待部署的机器创建一个普通账户，例如`test`
```
useradd test
设置一个密码，用于待会设置test账户免密登录时ssh-copy-id命令的密码输入
passwd test
```
2)  实现主控机与待部署机器`test`账户ssh免密登录, 例如192.168.239.136
在主控机`test`账户上执行
```
ssh-keygen -t rsa
```
命令提示全部回车确认，然后会在`/home/test`目录下生成`.ssh`目录

如果您需要从主控机免密ssh到任何一台服务器(例如192.168.239.136)，您需要在`test`账户执行下面的命令：
```
ssh-copy-id 192.168.239.136
```
此时需要您输入192.168.239.136的`test`账户的密码，完成后，您执行下面的命令，将不会再要求您输入密码
```
ssh 192.168.239.136 "echo ok"
```
这就实现了主控机`test`账户免密ssh到test@192.168.239.136的功能，其余待部署节点，执行同样的`ssh-copy-id`命令即可。

3）要让全部待部署的机器上的test账户sudo 免密
在主控机新建一个文本文件：`test_nopasswd`，(名字可以随意，可以参考这种写法)
```
test ALL=(ALL:ALL) NOPASSWD: ALL
```
写入一行内容，如上，如果您的账户不是test，替换上面的test即可。
把这个文件放到待部署机器的`/etc/sudoers.d/`目录，然后设置权限为`0440`即可实现test账户`sudo`免密。
如果待部署机器上没有其它可sudo，免密ssh的账户，那么这个过程可能需要逐个机器手动执行。
完后这个步骤以后，您需要在主控机`test`账户上对全部待部署机器执行验证：
```
ssh 192.168.239.136 "sudo echo ok"
```
如果全部正常返回，那么实现了test账户的sudo免密

#### 2、确保待部署机器之间网络和端口互通
待部署服务器之间应使用内网通信，确保端口互通，一般做法是关闭防火墙：
```
systemctl disable firewalld
```
否则可能在部署过程会出现zookeeper无法互联导致部署中断。
#### 3、确保待安装服务器时钟同步
请运维同学安装`ntp时钟同步`服务，pollybase是时序数据的存储查询引擎，如果服务器之间时钟差异较大，可能会出现意想不到的异常。

#### 4、主控机安装ansible
```
yum install ansible -y
安装完成后，执行：
ansible --version
```
如果能正常看到ansible的版本信息，说明ansible安装成功，经测试无误的`ansible`版本是`2.9.27`

#### 5、准备pollybase集群的`roles`
在`/root/.ansible/`目录下，创建`roles`目录，这是ansible默认会扫描的`roles`目录之一.
将`roles.tar.gz`当中的`polly-jre`,`polly-zookeeper`,`polly-kafka`,`polly-server`全部放到`/root/.ansible/roles`目录下即可。

### 二、集群配置
zookeeper、kafka、pollybase的配置信息分别在各自的`defaults/main.yml`文件中。
`polly-zookeeper/defaults/main.yml`:
```
polly_jre_home: /usr/lib/polly-jre
zookeeper_user: zookeeper
zookeeper_version: 3.5.10
client_port: 2181
init_limit: 5
sync_limit: 2
tick_time: 2000
zookeeper_autopurge_purgeInterval: 0
zookeeper_autopurge_snapRetainCount: 10
zookeeper_cluster_ports: "2888:3888"
zookeeper_max_client_connections: 60
zookeeper_admin_server: false
zookeeper_data_dir: "/home/{{ zookeeper_user }}/polly-zookeeper-data"
zookeeper_data_log_dir: "{{ zookeeper_data_dir }}"
zookeeper_dir: "/home/{{ zookeeper_user }}/polly-zookeeper-{{ zookeeper_version }}"
zookeeper_log_dir: "{{ zookeeper_dir }}/logs"
zookeeper_conf_dir: "{{ zookeeper_dir }}/conf"
zookeeper_rolling_log_file_max_size: 10MB
zookeeper_max_rolling_log_file_count: 10
zookeeper: "{{ groups['zookeeper'] }}"
```
简要说明：为zookeeper节点创建zookeeper账户，zookeeper的数据文件默认安装在zookeeper账户的家目录。
_____

`polly-kafka/defaults/main.yml`:
```
polly_jre_home: /usr/lib/polly-jre
kafka_user: kafka
kafka_scala_version: 2.12
kafka_version: 3.3.1
kafka_dir: "/home/{{ kafka_user }}/polly-kafka-{{ kafka_version }}"
zookeeper_client_port: 2181
kafka_listeners:
- scheme: "PLAINTEXT"
  host: "{{ inventory_hostname }}"
  port: 9092
kafka_log_dirs: "/home/{{ kafka_user }}/kafka/data"
kafka_jmx_opts: "-Dcom.sun.management.jmxremote -Dcom.sun.management.jmxremote.authenticate=false -Dcom.sun.management.jmxremote.ssl=false "
kafka_opts: "-Djava.rmi.server.useCodebaseOnly=true -Dcom.sun.jndi.ldap.object.trustURLCodebase=false -Dcom.sun.jndi.cosnaming.object.trustURLCodebase=false -Dlog4j.configuration=file:{{ kafka_dir }}/config/log4j.properties"
kafka_java_heap: "-Xms2g -Xmx2g"
```
简要说明：为kafka节点创建kafka账户，kafka的数据文件默认放在在kafka账户的家目录。
安装前，需要根据机器配置，灵活调整数据文件目录和broker的堆内存设置
________
`polly-server/defaults/main.yml`:
```
polly_jre_home: /usr/lib/polly-jre
polly_user: polly
polly_dir: "/home/{{ polly_user }}/pollybase"
polly_audit_log_dir: "{{ polly_dir }}/logs"
polly_log_dir: "{{ polly_dir }}/logs"
polly_version: 1.0.0
polly_hot_data_dir: "/home/{{ polly_user }}/hot-data"
polly_warm_data_enable: false
plly_warm_data_dir: "/home/{{ polly_user }}/warm-data"
polly_hot_data_threshold: 80
polly_http_bind: "0.0.0.0"
polly_http_port: 15630
polly_es_port: 9200
polly_public_url: "http://{{ inventory_hostname }}:{{polly_http_port}}"
polly_external_url: "http://{{ inventory_hostname }}:{{polly_http_port}}"
#Xss最低2M
polly_java_opts_Xss: "2M"
polly_java_opts_Xms: "1G"
polly_java_opts_Xmx: "1G"
#最大堆外内存
polly_java_opts_MaxDirectMemorySize: "1G"
zookeeper_cliet_port: 2181
kafka_broker_port: 9092
zookeeper: "{{ groups['zookeeper'] }}"
kafka: "{{ groups['kafka'] }}"
```
______
### 二、安装
#### 1、单机安装示例
```
#以下各分组名字不可更改哦
#is_datanode=false则这个节点不存储任何实例，也不参与消费
[polly_leader]
192.168.239.131 is_datanode=true
#broker_id是手动给kafka分配的id, 从0开始递增
[kafka]
192.168.239.131 broker_id=0
#zid则是手动给zookeeper分配的id，从1开始递增
[zookeeper]
192.168.239.131 zid=1
```
**准备剧本：**
** 如果您使用的部署`root`账户`ssh免密`，而是其它普通账户，那么需要把下面全部的`remote_user`换成该账户的名字。**
```
- hosts: kafka
  remote_user: root
  roles:
    - polly-kafka
- hosts: polly_leader
  remote_user: root
  roles:
    - polly-server
```
使用Chrome浏览器，访问地址`http://192.168.239.131:15630`

#### 2、集群安装
在主控机准备好机器列表和安装剧本：
主机列表文件hosts:
```
#以下各分组名字不可更改哦
#is_datanode=false则这个节点不存储任何实例，也不参与消费

[polly_other]
192.168.239.134 is_datanode=false
192.168.239.135 is_datanode=true
[polly_leader]
192.168.239.131 is_datanode=true

#broker_id是手动给kafka分配的id, 从0开始递增
[kafka]
192.168.239.131 broker_id=0
192.168.239.134 broker_id=1
192.168.239.135 broker_id=2

#zid则是手动给zookeeper分配的id，从1开始递增
[zookeeper]
192.168.239.131 zid=1
192.168.239.134 zid=2
192.168.239.135 zid=3
```
 安装剧本：test.yml
```
- hosts: kafka
  remote_user: root
  roles:
    - polly-kafka
#集群部署的时候，必须先启动一个节点，再启动其余节点
- hosts: polly_leader
  remote_user: root
  roles:
    - polly-server

- hosts: polly_leader
  remote_user: root
  tasks:
    - name: "等待主节点初始化完成"
      pause:
        seconds: 15

- hosts: polly_other
  remote_user: root
  roles:
    - polly-server
```
kafka集群正常安装即可，这里要说明一下剧本为什么这么写：
pollybase需要先启动一个节点，这个节点作为集群第一个节点，会完成一些元数据的初始化，其余集群节点会自动从这个节点同步已经初始化的元数据，如果全部节点同时启动，那么可能导致元数据逻辑错误。

执行安装：
```
ansible-playbook -i hosts test.yml
```
安装安装完成后，需要在集群管理界面给集群节点分配消费分区和存储分区。



