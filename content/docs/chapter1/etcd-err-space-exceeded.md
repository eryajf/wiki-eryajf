---
title: ETCD配额问题处理与验证 Error etcdserver mvcc database space exceeded
linktitle: ETCD配额问题处理与验证 Error etcdserver mvcc database space exceeded
type: book
date: "2021-05-23T00:00:00+01:00"
# Prev/next pager order (if `docs_section_pager` enabled in `params.toml`)
weight: 4
---

## 1，部署

先来部署一个单机版的etcd。国外难以下载，感谢华为镜像站：https://mirrors.huaweicloud.com/etcd/

下载到本机解压放到对应目录：

```
tar xf etcd-v3.3.11-linux-amd64.tar.gz
cd etcd-v3.3.11-linux-amd64/
mv etcd etcdctl /usr/bin/
```

因为默认的空间配额是2G，官方下载的包内，有如下说明：

```
The default storage size limit is 2GB, configurable with `--quota-backend-bytes` flag. 8GB is a suggested maximum size for normal environments and etcd warns at startup if the configured value exceeds it.
```

这里将配额指定为16M便于测试：

```
$ etcd --quota-backend-bytes=$((16*1024*1024))
```

启动之后，首先查看一下状态：

```
$ETCDCTL_API=3 etcdctl --endpoints="http://127.0.0.1:2379" --write-out=table endpoint status
+-----------------------+------------------+---------+---------+-----------+-----------+------------+
|       ENDPOINT        |        ID        | VERSION | DB SIZE | IS LEADER | RAFT TERM | RAFT INDEX |
+-----------------------+------------------+---------+---------+-----------+-----------+------------+
| http://127.0.0.1:2379 | 8e9e05c52164694d |  3.3.11 |   20 KB |      true |         2 |         15 |
+-----------------------+------------------+---------+---------+-----------+-----------+------------+
```

## 2，测试

接着批量写入，查看状态：

```
$ while [ 1 ]; do dd if=/dev/urandom bs=1024 count=1024  | ETCDCTL_API=3 etcdctl put key  || break; done

。。。。。。

1024+0 records in
1024+0 records out
1048576 bytes (1.0 MB) copied, 0.0830886 s, 12.6 MB/s
Error: etcdserver: mvcc: database space exceeded
```

看到一个报错说：`Error: etcdserver: mvcc: database space exceeded`，亦即超出了配额，执行简单的写入：

```
$ ETCDCTL_API=3 etcdctl put newkey 123

Error: etcdserver: mvcc: database space exceeded
```

依然是这个错误，再看服务状态：

```
$ ETCDCTL_API=3 etcdctl --endpoints="http://127.0.0.1:2379" --write-out=table endpoint status
+-----------------------+------------------+---------+---------+-----------+-----------+------------+
|       ENDPOINT        |        ID        | VERSION | DB SIZE | IS LEADER | RAFT TERM | RAFT INDEX |
+-----------------------+------------------+---------+---------+-----------+-----------+------------+
| http://127.0.0.1:2379 | 8e9e05c52164694d |  3.3.11 |   17 MB |      true |         3 |         18 |
+-----------------------+------------------+---------+---------+-----------+-----------+------------+
```

很明显超出了配额。

使用如下命令查看告警信息：

```
$ ETCDCTL_API=3 etcdctl put newkey 123
Error: etcdserver: mvcc: database space exceeded
```

所有的运维管理都在操作 Etcd 的存储空间。存储空间的配额用于控制 Etcd 数据空间的大小，如果 Etcd 节点磁盘空间不足了，配额会触发告警，然后 Etcd 系统将进入操作受限的维护模式。

## 3，压缩

处理的方案是可以针对如上内容进行压缩。

```
#使用API3
$ export ETCDCTL_API=3 

# 获取当前版本
$ rev=$(etcdctl --endpoints=http://127.0.0.1:2379 endpoint status --write-out="json" | egrep -o '"revision":[0-9]*' | egrep -o '[0-9].*')

# 压缩掉所有旧版本
etcdctl --endpoints=http://127.0.0.1:2379 compact $rev

# 整理多余的空间
etcdctl --endpoints=http://127.0.0.1:2379 defrag

# 取消告警信息
etcdctl --endpoints=http://127.0.0.1:2379 alarm disarm
```

然后再次查看状态：

```
$ETCDCTL_API=3 etcdctl --endpoints="http://127.0.0.1:2379" --write-out=table endpoint status
+-----------------------+------------------+---------+---------+-----------+-----------+------------+
|       ENDPOINT        |        ID        | VERSION | DB SIZE | IS LEADER | RAFT TERM | RAFT INDEX |
+-----------------------+------------------+---------+---------+-----------+-----------+------------+
| http://127.0.0.1:2379 | 8e9e05c52164694d |  3.3.11 |  1.1 MB |      true |         3 |         21 |
+-----------------------+------------------+---------+---------+-----------+-----------+------------+
```

验证写入：

```
$ ETCDCTL_API=3 etcdctl put eryajf test
OK
$ ETCDCTL_API=3 etcdctl get eryajf
eryajf
test
```

服务正常了。

![](http://t.eryajf.net/imgs/2021/05/dc37be98ce0a1d1e.jpg)

## 4，systemd管理

通常我们使用systemd管理服务，于是可以在启动的时候加入配置信息：

```
$cat /etc/systemd/system/etcd.service
[Unit]
Description=Etcd
After=network.target
Before=flanneld.service

[Service]
User=root
ExecStart=/usr/bin/etcd  -name etcd1 -data-dir /var/lib/etcd  --advertise-client-urls http://10.3.7.7:2379,http://127.0.0.1:2379 --listen-client-urls http://10.3.7.7:2379,http://127.0.0.1:2379 --auto-compaction-retention=1 --quota-backend-bytes=8388608000
Restart=on-failure
Type=notify
LimitNOFILE=65536

[Install]
WantedBy=multi-user.target
```

- `auto-compaction-retention`：表示每隔一个小时自动压缩一次
- `quota-backend-bytes`：磁盘空间调整为8G，官方建议最大8G。

然后重启即可。

参考：https://etcd.io/docs/v3.4/faq/#what-does-mvcc-database-space-exceeded-mean-and-how-do-i-fix-it
