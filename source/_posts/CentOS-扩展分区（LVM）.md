---
title: CentOS 扩展分区（LVM）
date: 2022-01-20 09:09:12
categories:
- CentOS
tags:
- CentOS
- 分区扩展
---

# 新磁盘扩展

```shell
# 查看磁盘及分区情况
lsblk
# 列出信息的最后一列为 disk 是磁盘，part 是分区，part 下有 LVM 表示该磁盘在使用，没有的表示还没有使用
# 首先创建 PV
pvcreate /dev/sdb
# /dev/sdb 为新磁盘的挂载点，根据实际情况替换
# 查看 VG 分组
vgs
# 扩展 VG
vgextend vg_centos7 /dev/sdb
# 第一个参数为要扩展的 VG 分组名，第二个参数为创建的 PV
# 查看 LVM信息
lvs
# 扩展目标分区
lvextend -L +100G /dev/mapper/vg_centos7-lv_root
# +100G 为扩展的大小
# /dev/mapper/vg_centos7-lv_root 为要扩展的分区
# 更新分区大小
xfs_growfs /dev/mapper/vg_centos7-lv_root
# 或者
resize2fs /dev/mapper/vg_centos7-lv_root
```

# 原有分区扩展

以合并 `/home` 分区为例。

```shell
# 查看磁盘及分区情况
lsblk
# 卸载要合并掉的分区，要记住该分区的大小
umount /home
# 移除对应的 LVM
lvremove /dev/mapper/vg_centos7-lv_home
# 扩展目标分区
lvextend -L +100G /dev/mapper/vg_centos7-lv_root
# +100G 为上一步移除的 /home 分区的大小
# /dev/mapper/vg_centos7-lv_root 为要扩展的分区
# 更新分区大小
xfs_growfs /dev/mapper/vg_centos7-lv_root
# 或者
resize2fs /dev/mapper/vg_centos7-lv_root
```
