---
title: 如何强制删除"Device Busy"状态的devicemapper设备
categories:
  - 专业
  - 开发
  - Linux
  - devicemapper
tags:
  - Linux
  - devicemapper
  - storage
index_img: https://cdn.jsdelivr.net/gh/simonyangchao/resources@image/linux.jpeg
banner_img: https://cdn.jsdelivr.net/gh/simonyangchao/resources@image/linux.jpeg
---



`dmsetup remove <device>`设备时，偶尔出现`Device Busy`无法删除。本文提供一种强制删除dm设备的操作步骤，可能无法解决所有的问题。后续需要深入研究Linux devicemapper的IO栈，找到根因。

<!-- more -->

## Step 1

`mount | grep xxx`， 查看设备是否正在被挂载使用；如果设备正在被挂载，使用Linux umount解挂载。注意，如果设备有多个挂载点，需要多次执行umount知道所有挂载点被解挂载。

## Step 2

`lsof `& `fuser`查看是否有进程正在使用该设备；如果有进程正在使用，在确保进程可以杀死后，`kill -9 <pid>`

## Step 3

如果没有挂载且没有进程正在使用

1. `dmsetup status pvc-xxx`， 查看设备table

```
[root@node-172-16-124-14 ~]# dmsetup status pvc-af945d77-234c-49b8-bc4c-3d360c01bb19
0 419430400 thin 286720 419430399
```

2. `dmsetup reload`重写dm table将设备设置为error

```
[root@node-172-16-124-14 ~]# dmsetup reload pvc-af945d77-234c-49b8-bc4c-3d360c01bb19 --table "0 419430400 error"
```

3. `dmsetup suspend`

```
[root@node-172-16-124-14 ~]# dmsetup suspend --noflush pvc-af945d77-234c-49b8-bc4c-3d360c01bb19
```

4. `dmsetup resume`

```
[root@node-172-16-124-14 ~]# dmsetup resume pvc-af945d77-234c-49b8-bc4c-3d360c01bb19
```

5. `dmsetup status pvc-xxx`，确认状态是error

```
[root@node-172-16-124-14 ~]# dmsetup status pvc-af945d77-234c-49b8-bc4c-3d360c01bb19
0 419430400 error 
```

6. `dmsetup remove --force`删除设备

```
[root@node-172-16-124-14 ~]# dmsetup remove --force pvc-af945d77-234c-49b8-bc4c-3d360c01bb19
```

