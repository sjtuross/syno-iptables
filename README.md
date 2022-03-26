# 本仓库提供群晖系统缺失的一些iptables模块

* 用于透明代理
* 支持原生Docker IPv6 NAT模式

理论上只要架构、内核以及iptables版本吻合，预编译的模块就可以使用，或者说小版本的系统升级一般不会升级内核，可以继续使用。不吻合切勿尝试，可能造成未知的系统问题。

## 准备工作

通过该页面[Synology Architectures](https://github.com/SynoCommunity/spksrc/wiki/Synology-and-SynoCommunity-Package-Architectures)查询架构，比如DS918+的架构为apollolake

通过`uname -a`命令查询内核版本，比如DS918+ 7.0.1-42218系统内核为4.4.180+（结尾的加号代表自定义编译的4.X内核）

```text
Linux DSM7 4.4.180+ #42218 SMP Mon Oct 18 19:17:56 CST 2021 x86_64 GNU/Linux synology_apollolake_918+
```

通过`iptables -V`命令查询iptables版本

```text
iptables v1.8.3 (legacy)
```

本仓库提供以下系统的预编译模块，经测试可以正常加载。

| arch       | kernel   | iptables version | system model | platform version |
| :--------- | :------- | :--------------- | :----------- | :--------------- |
| apollolake | 4.4.180+ | v1.8.3           | DS918+       | 7.0.1-42218      |
| apollolake | 4.4.59+  | v1.6.0           | DS918+       | 6.2.3-25426      |
| broadwell  | 3.10.105 | v1.6.0           | DS3617xs     | 6.2.3-25426      |
| bromolow   | 3.10.105 | v1.6.0           | DS3615xs     | 6.2.3-25426      |

## 安装并尝试加载

上传相应的ko模块至/lib/modules/，上传相应的so模块至/usr/lib/iptables/，即可。

📝 文件名含ip6的用于支持Docker IPv6 NAT，其余的用于透明代理，根据需要选择。

📝 ko内核模块和so用户模块需要同时安装。

⚠️ Windows用户注意，模块文件名是区分大小写的，大写的为标记模块，小写的为匹配模块，它们之间是相辅相成的，切勿彼此覆盖。

以下以DS3617xs 6.2.3-25426为例，尝试加载透明代理所需的ko内核模块。

由于模块互相有依赖性，需按一定顺序加载，有些是系统自带的模块。如果提示File Exists，说明已经加载，如果没有提示，说明加载成功。

```bash
sudo -i
insmod /lib/modules/nfnetlink.ko
insmod /lib/modules/ip_set.ko
insmod /lib/modules/ip_set_hash_ip.ko
insmod /lib/modules/xt_set.ko
insmod /lib/modules/ip_set_hash_net.ko
insmod /lib/modules/xt_mark.ko
insmod /lib/modules/xt_connmark.ko
insmod /lib/modules/xt_comment.ko
insmod /lib/modules/nf_tproxy_core.ko
insmod /lib/modules/xt_TPROXY.ko
insmod /lib/modules/xt_socket.ko
insmod /lib/modules/iptable_mangle.ko
```

📝 运行`lsmod`查看已加载的内核模块列表，或运行`dmesg | tail`查看加载失败的原因。

⚠️ 不同内核版本netfilter编译生成的ko内核模块可能不完全一样。比如，nf_tproxy_core.ko模块只有3.X内核才会有，nf_nat_masquerade_ipv6.ko模块只有4.X内核才会有。

## 如何自编译

本仓库无法提供适合所有群晖系统的预编译模块，或者不愿意使用预编译模块，可以尝试[自编译](BUILD.md)。

## 具体实践分享

* [群晖实现v2rayA透明代理模式](usage/v2raya-transparent-proxy.md)
* [群晖支持原生Docker IPv6 NAT模式 (DSM 6)](usage/docker-ipv6-nat-dsm6.md)
* [群晖支持原生Docker IPv6 NAT模式 (DSM 7)](usage/docker-ipv6-nat-dsm7.md)
* [群晖Docker安装OpenWrt旁路由](usage/docker-openwrt-transparent-proxy.md)

## 感谢

* [在群晖部署适用 IPv6、Fullcone NAT 的旁路由透明代理](https://blog.kaaass.net/archives/1576)
* [spksrc - a cross compilation framework](https://github.com/SynoCommunity/spksrc)
* [fix synology docker ipv6 issue](https://github.com/wangliangliang2/fix_synology_docker_ipv6)
