# 解决群晖docker ipv6无法正常使用

理论上来说，如果系统架构、内核版本、iptables版本以及机器型号一致，那么本目录下提供的模块可以使用；系统小版本不一致可以尝试使用，但未经测试，不保证有位置风险，后果请自负。如果本仓库没有你所需要的模块或者不放心使用，请自行编译。

## 准备工作

内容和根目录README高度相似，我懒得重复了

补充一下已验证的机器架构以及型号（仅限于docker ipv6问题）：

```bash
# uname -a
Linux DiskStation 3.10.105 #24922 SMP Wed Jul 3 16:37:23 CST 2019 x86_64 GNU/Linux synology_broadwell_3617xs
```



| arch      | kernel   | iptables version | system model | platform version |
| --------- | -------- | ---------------- | ------------ | ---------------- |
| broadwell | 3.10.105 | v1.6.0           | DS3617xs     | 6.2.2-24922      |



## 使用

上传ko内的文件至 `/lib/modules/`，so内的文件至`/usr/lib/iptables/` 目录，找到`/usr/syno/etc.defaults/iptables_modules_list`，如果提示没有此文件请执行sudo -i，**务必记得备份该文件**，使用vi或者其他编辑器打开此文件，找到：`IPV6_MODULES=`这一行，在后面追加ko里面的文件名，注意空格隔开，修改完成后保存并退出。修改后的内容参考如下：

`IPV6_MODULES="x_tables.ko nf_conntrack.ko ip6_tables.ko ip6table_filter.ko nf_defrag_ipv6.ko nf_conntrack_ipv6.ko nf_nat_ipv6.ko ip6t_MASQUERADE.ko ip6table_nat.ko ip6table_raw.ko ip6table_mangle.ko"`

docker配置文件参考，以下为开启ipv6后的配置：

```json
/var/packages/Docker/etc/dockerd.json
{
	"data-root" : "/var/packages/Docker/target/docker",
	"log-driver" : "db",
	"registry-mirrors" : [],
	"storage-driver" : "btrfs",
	"ipv6": true,
	"fixed-cidr-v6": "fd00::/80",
	"experimental": true,
	"ip6tables": true,
	"userland-proxy": true
}
```

配置好后重启docker，如果提示启动失败那么重启一下系统

## 测试

完成以上操作之后，docker在开启ipv6选项的条件下能够正常启动（synoservice --restart pkgctl-Docker）不报错（调试模式启动仍显示缺失组件 package bridge is not installed，怀疑是docker自身问题），运行`docker run --rm -it busybox ping -6 -c4 ipv6-test.com` 也能通过测试。但是，我检查之后发现启动的容器只有一个能通过ipv6访问，进一步发现只有它监听了ipv6，原因未知。执行ip6table -t nat -nvL 后结果如下：

```bash
Chain PREROUTING (policy ACCEPT 1 packets, 76 bytes)
 pkts bytes target     prot opt in     out     source               destination

Chain INPUT (policy ACCEPT 1 packets, 76 bytes)
 pkts bytes target     prot opt in     out     source               destination

Chain OUTPUT (policy ACCEPT 19 packets, 2112 bytes)
 pkts bytes target     prot opt in     out     source               destination
  121 12999 DEFAULT_OUTPUT  all      *      *       ::/0                 ::/0

Chain POSTROUTING (policy ACCEPT 19 packets, 2112 bytes)
 pkts bytes target     prot opt in     out     source               destination
  122 13081 DEFAULT_POSTROUTING  all      *      *       ::/0                 ::/0

Chain DEFAULT_OUTPUT (1 references)
 pkts bytes target     prot opt in     out     source               destination
    0     0 DOCKER     all      *      *       ::/0                !::1                  ADDRTYPE match dst-type LOCAL

Chain DEFAULT_POSTROUTING (1 references)
 pkts bytes target     prot opt in     out     source               destination
    1    82 MASQUERADE  all      *      !docker0  fd00::/80            ::/0
    0     0 MASQUERADE  tcp      *      *       fd00::242:ac11:3     fd00::242:ac11:3     tcp dpt:3306

Chain DOCKER (1 references)
 pkts bytes target     prot opt in     out     source               destination
    0     0 RETURN     all      docker0 *       ::/0                 ::/0
    0     0 DNAT       tcp      !docker0 *       ::/0                 ::/0                 tcp dpt:3306 to:[fd00::242:ac11:3]:3306
```



另外，请教过大佬之后得知，20版本后的docker nat不会自动转换ipv6。如果需要其他容器也能**被**ipv6访问，只需找到容器目录下 `hostconfig.json`文件，将其中hostIp监听的地址从`"hostIp":"0.0.0.0"`改为 `"hostIp":""`即可同时监听ipv6和ipv6。至此，问题解决。

## 感谢

- [spksrc - a cross compilation framework](https://github.com/SynoCommunity/spksrc)

- [fix_synology_docker_ipv6](https://github.com/wangliangliang2/fix_synology_docker_ipv6)