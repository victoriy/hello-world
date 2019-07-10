## 虚拟机新增磁盘空间
> 2019/05/03
- 登录系统查看挂载磁盘信息
```bash
$fdisk -l

Disk /dev/sda: 85.9 GB, 85899345920 bytes, 167772160 sectors
Units = sectors of 1 * 512 = 512 bytes
Sector size (logical/physical): 512 bytes / 512 bytes
I/O size (minimum/optimal): 512 bytes / 512 bytes
Disk label type: dos
Disk identifier: 0x000be849

   Device Boot      Start         End      Blocks   Id  System
/dev/sda1   *        2048     2099199     1048576   83  Linux
/dev/sda2         2099200   167526399    82713600   8e  Linux LVM

Disk /dev/sdb: 536.9 GB, 536870912000 bytes, 1048576000 sectors
Units = sectors of 1 * 512 = 512 bytes
Sector size (logical/physical): 512 bytes / 512 bytes
I/O size (minimum/optimal): 512 bytes / 512 bytes
```
- 如果没有信息显示请扫描新磁盘
```bash
$echo "- - -" > /sys/class/scsi_host/host`X`/scan
```
- 再次查看挂载磁盘信息
```bash
$fdisk -l
```
- parted disk
```bash
首先把大容量的磁盘转换为 GPT 格式。由于 GPT 格式的磁盘相当于原来 MBR 磁盘中原来保留 4个 partition table的4*16 个字节只留第一个 16 个字节，
其它的类似于扩展分区，真正的 partition table 在 512 字节之后，所以对 GPT 磁盘表来讲没有四个主分区的限制。

$lsblk
NAME          MAJ:MIN RM  SIZE RO TYPE MOUNTPOINT
sda           253:32   0  600G  0 disk

$parted /dev/sdb
...
(parted) mklabel gpt    # 输入mklable gpt，把vdc改成gpt大分区格式                                      
(parted) print          # 查看sda分区状态，可以看到已经打上了gpt的标签
...
Disk /dev/sdb: 600GB
...
(parted) mkpart primary 0 600gb  # 创建一个主分区，容量从0GB开始到3221GB的全部空间
...
Ignore/Cancel? i
(parted) print    # 可查看分区
...

Number  Start   End     Size    File system  Name     Flags
 1      17.4kB  600GB  600GB               primary
 
(parted) quit   # 退出parted分区工具
...

$lsblk
sda           253:32   0    600G  0 disk 
└─sda1        253:33   0    600G  0 part
```
- PV创建
```bash
$pvcreate /dev/sdb1
```
- VG创建
```bash
$vgcreate datavg /dev/sdb1   ##创建vg
$vgdisplay -v |more  ##显示vg信息
```
- LV01创建，把VG空间全部分配给此LV 
```bash
$lvcreate -n lv01 -l 100%FREE datavg
```
- LV格式化
```bash
$mkfs.xfs /dev/datavg/lv01
```
- 创建挂载目录
```bash
$mkdir /data
$chown -R 755 /data
$mount /dev/mapper/datavg-lv01 /data/   //挂载磁盘
$df -h  //查看挂载目录状态
```
- 使能系统重启后自动挂载磁盘
```bash
$vi /etc/fstab
/dev/mapper/datavg-lv01 /data   xfs defaults        0 0
```
- 重启系统检查磁盘挂载情况
```bash
$df -h
$fdisk -l
```
# End!