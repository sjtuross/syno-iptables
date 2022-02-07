## 本仓库提供群晖系统缺失的一些iptables模块

理论上只要架构、内核以及iptables版本吻合，预编译的模块就可以使用，或者说小版本的系统升级一般不会升级内核，可以继续使用。不吻合切勿尝试，可能造成未知的系统问题。

架构通过该页面[Synology Architectures](https://github.com/SynoCommunity/spksrc/wiki/Synology-and-SynoCommunity-Package-Architectures)查询，比如DS918+的架构为apollolake

内核版本通过uname -a命令查询，比如DS918+ 7.0.1-42218系统内核为4.4.180+（结尾的加号代表自定义编译的内核）

```bash
Linux DSM7 4.4.180+ #42218 SMP Mon Oct 18 19:17:56 CST 2021 x86_64 GNU/Linux synology_apollolake_918+
```

iptables版本通过iptables -V命令查询

```bash
iptables v1.8.3 (legacy)
```

本仓库提供的预编译模块在以下系统测试正常。

| arch       | kernel    | iptables version | system model | platform version |
| :--------- | :-------- | :--------------- | :----------- | :--------------- |
| apollolake | 4.4.180+  | v1.8.3           | DS918+       | 7.0.1-42218      |
| apollolake | 4.4.59+   | v1.6.0           | DS918+       | 6.2.3-25426      |
| broadwell  | 3.10.105+ | v1.6.0           | DS3617xs     | 6.2.3-25426      |

## 安装并尝试加载

上传相应的ko模块至/lib/modules/，上传相应的so模块至/usr/lib/iptables/，即可。

Windows用户注意，模块文件名是区分大小写的，大写的为标记模块，小写的为匹配模块，它们之间是相辅相成的，切勿彼此覆盖。

然后尝试加载ko内核模块。由于模块互相有依赖性，需按一定顺序加载，有些是系统自带的模块。如果提示File Exists，说明已经加载，如果没有提示，说明加载成功。不同内核版本netfilter编译输出的ko内核模块可能不完全一样，以下以DS3617xs 6.2.3-25426为例。

```bash
sudo -i
insmod /lib/modules/nfnetlink.ko
insmod /lib/modules/ip_set.ko
insmod /lib/modules/ip_set_hash_ip.ko
insmod /lib/modules/xt_set.ko
insmod /lib/modules/ip_set_hash_net.ko
insmod /lib/modules/xt_mark.ko
insmod /lib/modules/xt_connmark.ko
insmod /lib/modules/nf_tproxy_core.ko
insmod /lib/modules/xt_TPROXY.ko
insmod /lib/modules/iptable_mangle.ko
```

运行lsmod查看已加载的模块列表，或运行dmesg | tail查看模块加载失败的原因。

## 如何自编译
本仓库无法提供适合所有群晖系统的预编译模块，有能力的同学可以尝试自编译。

### 启动docker编译镜像

```bash
git clone https://github.com/SynoCommunity/spksrc.git
cd spksrc
docker run -it -v $(pwd):/spksrc ghcr.io/synocommunity/spksrc /bin/bash
```

### 在编译容器中准备及下载所需的toolchain和kernel

syno-apollolake-7.0为对应系统版本的代号为目录名，请自行调整

```bash
make setup
cd /spksrc/toolchain/syno-apollolake-7.0
make
cd /spksrc/kernel/syno-apollolake-7.0
make
```

### 编译netfilter内核模块，编译之后ko文件在build目录中

syno-apollolake-7.0为对应系统版本的代号为目录名，请自行调整

```bash
export PATH="${PATH}:/spksrc/toolchain/syno-apollolake-7.0/work/x86_64-pc-linux-gnu/bin"
export CROSS=x86_64-pc-linux-gnu
export CC=${CROSS}-gcc
export LD=${CROSS}-ld
export AS=${CROSS}-as
export CXX=${CROSS}-g++
export CROSS_COMPILE=/spksrc/toolchain/syno-apollolake-7.0/work/x86_64-pc-linux-gnu/bin/x86_64-pc-linux-gnu-
export ARCH=x86_64
export KSRC=/spksrc/kernel/syno-apollolake-7.0/work/linux
MODULE=/spksrc/kernel/syno-apollolake-7.0/work/linux/net/netfilter
export CONFIG_NETFILTER_XT_CONNMARK=m
export CONFIG_NETFILTER_TPROXY=m
export CONFIG_NETFILTER_XT_TARGET_TPROXY=m
make ARCH=$ARCH CROSS_COMPILE=$CROSS_COMPILE -C $KSRC M=$MODULE clean
make ARCH=$ARCH CROSS_COMPILE=$CROSS_COMPILE -C $KSRC M=$MODULE modules -j 4
rm -rf build
mkdir build
cp $MODULE/*.ko build
mkdir build/ipset
cp $MODULE/ipset/*.ko build/ipset

```

### 编译iptables模块，编译之后so文件在extensions子目录中

iptables v1.8.3，适用于syno-apollolake-7.0，请自行调整

```bash
cd /spksrc/toolchain/syno-apollolake-7.0/work
wget https://www.netfilter.org/pub/iptables/iptables-1.8.3.tar.bz2
tar xjf iptables-1.8.3.tar.bz2
cd iptables-1.8.3
./configure --prefix="/spksrc/toolchain/syno-apollolake-7.0/work/build" --disable-nftables
make && make install
```

iptables v1.6.0，适用于syno-apollolake-6.2.3，请自行调整

```bash
cd /spksrc/toolchain/syno-apollolake-6.2.3/work
wget https://www.netfilter.org/pub/iptables/iptables-1.6.0.tar.bz2
tar xjf iptables-1.6.0.tar.bz2
cd iptables-1.6.0
./configure --prefix="/spksrc/toolchain/syno-apollolake-6.2.3/work/build" --disable-nftables
make && make install
```

## 感谢

- [在群晖部署适用 IPv6、Fullcone NAT 的旁路由透明代理](https://blog.kaaass.net/archives/1576)
- [spksrc - a cross compilation framework](https://github.com/SynoCommunity/spksrc)
