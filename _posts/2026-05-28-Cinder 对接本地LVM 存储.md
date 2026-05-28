**1、安装lvm2** 

```
yum install lvm2 -y
```

**2、创建本地 loop 设备**

```
dd if=/dev/zero of=volume_test bs=51200 count=1M              
```

**3、挂载/卸载本地 loop设备**

```
losetup /dev/loop0 volume_test     ##挂载
losetup -d /dev/loop0 volume_test  ##卸载
```

**4、创建 pv，vg，lvm**

```
 pvcreate /dev/loop0 volume_vg
 vgcreate volume_vg  /dev/loop0
 lvcreate -L 1g -n lvm_test volume_vg
```

**5、配置实例可以访问块存储卷组**

```
底层的操作系统管理这些设备并将其与卷关联。默认情况下，LVM卷扫描工具会扫描``/dev`` 目录，查找包含卷的块存储设备。如果项目在他们的卷上使用LVM，扫描工具检测到这些卷时会尝试缓存它们，可能会在底层操作系统和项目卷上产生各种问题。所以必须重新配置LVM，让它只扫描包含``cinder-volume``卷组的设备。编辑``/etc/lvm/lvm.conf``文件并完成下面的操作：
在``devices``部分，添加一个过滤器，只接受``/dev/sdb``设备，拒绝其他所有设备

devices {

...

filter = [ "a/loop0/", "r/.*/"]
...
```

**6、cinder.conf 配置**

```
[DEFAULT]
default_volume_type=lvm
enabled_backends=lvm
iscsi_ip_address = 192.169.4.90
glance_api_servers = http://192.169.4.90:9292
volume_driver=cinder.volume.drivers.lvm.LVMVolumeDriver

[lvm]
image_volume_cache_enabled = True
volume_backend_name=lvm
volume_driver=cinder.volume.drivers.lvm.LVMVolumeDriver
iscsi_ip_address=192.169.4.90
iscsi_helper=lioadm
volume_group=volume_vg  (vg池的名字)
volumes_dir=/var/lib/cinder/volumes/volume_vg
```

**7、启动相关服务**

```
systemctl restart openstack-cinder-volume.service target.service  
```

**8、创建卷和卷虚机测试**