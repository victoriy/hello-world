## CentOS 7添加永久静态路由
>2019/07/11
#### ip route显示和设定路由
- 显示路由表
```bash
$ip route show
default via 172.31.10.193 dev ens192 proto static metric 100 
172.31.10.192/28 dev ens192 proto kernel scope link src 172.31.10.201 metric 100 
or
$netstat -r
Kernel IP routing table
Destination     Gateway         Genmask         Flags   MSS Window  irtt Iface
default         gateway         0.0.0.0         UG        0 0          0 ens192
172.31.10.192   0.0.0.0         255.255.255.240 U         0 0          0 ens192
```
- 添加静态路由 
```bash
#重启网卡或重启设备后新增的静态路由会丢失

$ip route add 192.168.1.0/24 via 172.31.10.193 dev ens192
$netstat -r
Kernel IP routing table
Destination     Gateway         Genmask         Flags   MSS Window  irtt Iface
0.0.0.0         172.31.10.193   0.0.0.0         UG        0 0          0 ens192
172.31.10.192   0.0.0.0         255.255.255.240 U         0 0          0 ens192
192.168.1.0     172.31.10.193   255.255.255.0   UG        0 0          0 ens192
$systemctl restart network
$netstat -r
Kernel IP routing table
Destination     Gateway         Genmask         Flags   MSS Window  irtt Iface
default         gateway         0.0.0.0         UG        0 0          0 ens192
172.31.10.192   0.0.0.0         255.255.255.240 U         0 0          0 ens192
```
- 删除静态路由
```bash
$ip route del 192.168.1.0/24
```
- 设置永久的静态路由
```bash
#永久静态路由需要写到/etc/sysconfig/network-scripts/route-interface 文件中,无此文件新建即可。
#ifcfg-ens192文件改名为 ifcfg-eth0后，route-ens192文件也要改名为 route-eth0。

$vim /etc/sysconfig/network-scripts/route-ens192
#Add permanent static routes
192.168.1.0/24 via 172.31.10.193 dev ens192

#新增路由条目要重启网卡服务
$systemctl restart network
$netstat -r
```
- 屏蔽某一路由
```bash
#当我们不让系统到达某个子网范围或者某个主机是就可以手动的来进行屏蔽。

$route add -net 192.168.1.0/24 reject
```
## End!

