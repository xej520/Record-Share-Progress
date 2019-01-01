# 异常分析

## 报错

```
[root@mic-4 ~]# docker rm -f 2205f
Error response from daemon: Driver devicemapper failed to remove root filesystem 2205f0562812bf3007155f3bb2670508783e23d12428e31fa42caa0d8f68ca22: remove /data01/var/lib/docker/devicemapper/mnt/01bf74d1cd26d2fed52d2eb2f881acd0401431952e7c115ffc3107311c191341: device or resource busy
```

## 原因分析

挂载泄露

## 处理方式

以下方法亲测不可用：

```
docker stop 2205f 1>/dev/null 2>&1 | exit 0
docker rm -f 2205f 1>/dev/null 2>&1 | exit 0  
```

#### 查看挂载进程

```
[root@mic-4 ~]# grep /data01/var/lib/docker/devicemapper/mnt/01bf74d1cd26d2fed52d2eb2f881acd0401431952e7c115ffc3107311c191341 /proc/*/mountinfo
/proc/12437/mountinfo:141 140 253:5 / /data01/var/lib/docker/devicemapper/mnt/01bf74d1cd26d2fed52d2eb2f881acd0401431952e7c115ffc3107311c191341 rw,relatime shared:92 - xfs /dev/mapper/docker-252:32-268640386-01bf74d1cd26d2fed52d2eb2f881acd0401431952e7c115ffc3107311c191341 rw,nouuid,attr2,inode64,sunit=1024,swidth=2048,noquota
```

#### 检查进程名

```
[root@mic-4 ~]# ps -p 12437 -o comm=
ceph-osd
```

### 停止进程，删除容器

```
systemctl stop ceph-osd@1.service
docker rm -f id
```