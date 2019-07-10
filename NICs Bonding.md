## NICs Bonding
> 2017/07/10
#### Linux网卡绑定mode共有七种(0~6) bond0、bond1、bond2、bond3、bond4、bond5、bond6
- 常用的有三种

mode=0：平衡负载模式，有自动备援，但需要”Switch”支援及设定。

mode=1：自动备援模式，其中一条线若断线，其他线路将会自动备援。

mode=6：平衡负载模式，有自动备援，不必”Switch”支援及设定。
```bash
需要说明的是如果想做成mode 0的负载均衡,仅仅设置这里options bond0 miimon=100 mode=0是不够的,与网卡相连的交换机必须做特殊配置（这两个端口应该采取聚合方式），
因为做bonding的这两块网卡是使用同一个MAC地址.从原理分析一下（bond运行在mode 0下）：mode 0下bond所绑定的网卡的IP都被修改成相同的mac地址，如果这些网卡都被接在
同一个交换机，那么交换机的arp表里这个mac地址对应的端口就有多个，那么交换机接受到发往这个mac地址的包应该往哪个端口转发呢？正常情况下mac地址是全球唯一的，
一个mac地址对应多个端口肯定使交换机迷惑了。所以mode0下的bond如果连接到交换机，交换机这几个端口应该采取聚合方式（cisco称为ethernetchannel
foundry称为portgroup），因为交换机做了聚合后，聚合下的几个端口也被捆绑成一个mac地址.我们的解决办法是，两个网卡接入不同的交换机即可。mode6模式下无需配置交换机，
因为做bonding的这两块网卡是使用不同的MAC地址。
```
#### bond0
- 查看物理网卡信息
```bash
$nmcli device
DEVICE  TYPE      STATE        CONNECTION        
bond0   bond      connected    Bond connection 1 
em1     ethernet  connected    bond0 slave 1     
em2     ethernet  connected    bond0 slave 2     
em3     ethernet  unavailable  --                
em4     ethernet  unavailable  --                
lo      loopback  unmanaged    --           
```
- 查看网卡连接信息
```bash
$nmcli connection show
NAME               UUID                                  TYPE      DEVICE 
Bond connection 1  b90c46e5-081f-4521-872f-6bfca11abc9a  bond      bond0  
bond0 slave 1      b5b2fbb6-4c09-4cb9-b863-006bcfeb7808  ethernet  em1    
bond0 slave 2      d0114827-68fc-4cea-8780-86b39aeee28b  ethernet  em2    
em1                64f9f475-ec5c-4b94-b056-4552790c331f  ethernet  --     
em2                a0eb938a-433c-4616-87f0-62d6507c6971  ethernet  --     
em3                e09c6a41-edc2-4719-8cbc-bbd9504f3fde  ethernet  --     
em4                fb82c9d6-1044-4065-a97c-bc164b777f7f  ethernet  --  
```
- 删除网卡连接信息
```bash
$nmcli connection delete "Bond connection 1"
Connection 'Bond connection 1' (b90c46e5-081f-4521-872f-6bfca11abc9a) successfully deleted.
网卡连接信息删除成功，这里删除的其实就是/etc/sysconfig/network-scripts目录下网卡的配置文件。
```
#### 创建bond0
- 按照下面的语法，用 nmcli 命令为网络组接口创建一个连接
```bash
$nmcli con add type team con-name CNAME ifname INAME [config JSON]
CNAME 指代连接的名称，INAME 是接口名称，JSON (JavaScript Object Notation) 指定所使用的处理器(runner)。JSON语法格式如下：'{"runner":{"name":"METHOD"}}'
METHOD 是以下的其中一个：broadcast、activebackup、roundrobin、loadbalance 或者 lacp。
例如：
$nmcli con add type team con-name bond0 ifname bond0 config '{"runner":{"name":"roundrobin"}}'
Connection 'bond0' (6c3d21cf-4479-4242-8205-214f268d4a57) successfully added.
```
- 配置 bond0 的 IP、GW、DNS
```bash
$nmcli con modify bond0 ipv4.address "x.x.x.x/28" ipv4.gateway "x.x.x.1"
$nmcli con modify bond0 ipv4.dns "8.8.8.8"
```
- 设置 bond0 的属性为 manual
```bash
$nmcli con modify bond0 ipv4.method manual
```
- 添加 slave 网卡
```bash
$nmcli con add type team-slave con-name bond0-em1 ifname em1 master bond0
Connection 'bond0-em1' (df74a4c7-f8ff-4ae3-b04f-3dd1210598cd) successfully added.
$nmcli con add type team-slave con-name bond0-em2 ifname em2 master bond0 
```
- 启动 bond0
```bash
$nmcli con up bond0
Connection successfully activated (master waiting for slaves) (D-Bus active path: /org/freedesktop/NetworkManager/ActiveConnection/10)
```
- 检查bond0状态
```bash
$teamdctl  bond0 sta
setup:
  runner: roundrobin
ports:
  em1
    link watches:
      link summary: up
      instance[link_watch_0]:
        name: ethtool
        link: up
        down count: 0
  em2
    link watches:
      link summary: up
      instance[link_watch_0]:
        name: ethtool
        link: up
        down count: 0
```
```bash
$cat /etc/sysconfig/network-scripts/ifcfg-bond0
TEAM_CONFIG="{\"runner\":{\"name\":\"roundrobin\"}}"
PROXY_METHOD=none
BROWSER_ONLY=no
BOOTPROTO=none
DEFROUTE=yes
IPV4_FAILURE_FATAL=no
IPV6INIT=yes
IPV6_AUTOCONF=yes
IPV6_DEFROUTE=yes
IPV6_FAILURE_FATAL=no
IPV6_ADDR_GEN_MODE=stable-privacy
NAME=bond0
UUID=6c3d21cf-4479-4242-8205-214f268d4a57
DEVICE=bond0
ONBOOT=yes
DEVICETYPE=Team
IPADDR=x.x.x.x
PREFIX=28
GATEWAY=x.x.x.1
DNS1=8.8.8.8

$cat /etc/sysconfig/network-scripts/ifcfg-em1
TYPE=Ethernet
PROXY_METHOD=none
BROWSER_ONLY=no
BOOTPROTO=dhcp
DEFROUTE=yes
IPV4_FAILURE_FATAL=no
IPV6INIT=yes
IPV6_AUTOCONF=yes
IPV6_DEFROUTE=yes
IPV6_FAILURE_FATAL=no
IPV6_ADDR_GEN_MODE=stable-privacy
NAME=em1
UUID=64f9f475-ec5c-4b94-b056-4552790c331f
DEVICE=em1
ONBOOT=no

$cat /etc/sysconfig/network-scripts/ifcfg-em2
TYPE=Ethernet
PROXY_METHOD=none
BROWSER_ONLY=no
BOOTPROTO=dhcp
DEFROUTE=yes
IPV4_FAILURE_FATAL=no
IPV6INIT=yes
IPV6_AUTOCONF=yes
IPV6_DEFROUTE=yes
IPV6_FAILURE_FATAL=no
IPV6_ADDR_GEN_MODE=stable-privacy
NAME=em2
UUID=a0eb938a-433c-4616-87f0-62d6507c6971
DEVICE=em2
ONBOOT=no
```

