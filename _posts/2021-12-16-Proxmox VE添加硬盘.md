# Proxmox VE6.4-4添加硬盘

## 背景
装完pve，创建N个主机后，发现硬盘根本不够分。只能加多一块机械盘作为数据盘。

## 格式化及挂载到PVE机
```
mkfs -t ext4 /dev/sda1
mkdir -p /mnt/pve/sdb
mount -t ext4 /dev/sda1 /mnt/pve/sdb/
echo /dev/sda1 /mnt/pve/sdb/ ext4 defaults 1 2 >> /etc/fstab
```

## 虚机机添加磁盘
控制界面点击虚拟机->Hardare->Add->Hard Disk