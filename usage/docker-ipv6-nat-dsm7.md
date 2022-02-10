# 群晖支持原生Docker IPv6 NAT模式 (DSM 7)

方法和步骤类似[DSM 6](docker-ipv6-nat-dsm6.md)，针对DSM 7需略做调整。

修改模块加载列表/usr/syno/etc.defaults/iptables_modules_list的方式在DSM 7中不起作用，经测试是因为系统内置命令iptablestool无法加载模块，原因未知。

采用最直接的方式，修改/var/packages/Docker/scripts/start-stop-status文件。在docker启动之前插入insmod命令强行加载，暂时没有更优雅的方式。

```bash
# install modules
iptablestool --insmod "${DockerServName}" ${InsertModules}

insmod /lib/modules/nf_nat_ipv6.ko
insmod /lib/modules/nf_nat_masquerade_ipv6.ko
insmod /lib/modules/ip6t_MASQUERADE.ko
insmod /lib/modules/ip6table_nat.ko
insmod /lib/modules/ip6table_raw.ko
insmod /lib/modules/ip6table_mangle.ko

# start docker
if ! start_docker_daemon; then
    exit 1
fi
```

然后修改/var/packages/Docker/etc/dockerd.json启用原生IPv6 NAT模式，添加ipv6, fixed-cidr-v6, ip6tables, experimental这4个参数。务必注意json格式正确。

```json
{
  "data-root": "/var/packages/Docker/var/docker",
  "experimental": true,
  "fixed-cidr-v6": "fd07::/64",
  "ip6tables": true,
  "ipv6": true,
  "log-driver": "db",
  "registry-mirrors": [],
  "storage-driver": "btrfs"
}
```

重启docker

```bash
synopkg restart Docker
```

查看docker状态

```bash
synopkg status Docker
```

查看docker日志

```bash
tail /var/log/Docker/docker.log
```

测试IPv6连通性

```bash
docker run --rm -it busybox ping -6 -c4 2400:3200::1
```
