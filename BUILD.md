# 如何自编译

本仓库无法提供适合所有群晖系统的预编译模块，或者不愿意使用预编译模块，可以尝试自编译。

## 启动docker编译镜像

```bash
git clone https://github.com/SynoCommunity/spksrc.git
cd spksrc
docker run -it -v $(pwd):/spksrc ghcr.io/synocommunity/spksrc /bin/bash
```

## 在编译容器中准备及下载所需的kernel和toolchain

syno-apollolake-7.0为对应系统版本的代号为目录名，请自行调整

```bash
make setup
cd /spksrc/kernel/syno-apollolake-7.0
make
cd /spksrc/toolchain/syno-apollolake-7.0
make
```

## 设置交叉编译环境变量

⚠️ syno-apollolake-7.0为对应系统版本的代号为目录名，请自行调整

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
```

## 编译netfilter内核模块

⚠️ syno-apollolake-7.0为对应系统版本的代号为目录名，请自行调整

```bash
cd /spksrc/toolchain/syno-apollolake-7.0/work
MODULE=/spksrc/kernel/syno-apollolake-7.0/work/linux/net/netfilter
MODULE6=/spksrc/kernel/syno-apollolake-7.0/work/linux/net/ipv6/netfilter
export CONFIG_NETFILTER_XT_CONNMARK=m
export CONFIG_NETFILTER_TPROXY=m
export CONFIG_NETFILTER_XT_TARGET_TPROXY=m
export CONFIG_IP6_NF_TARGET_MASQUERADE=m
export CONFIG_IP6_NF_NAT=m
export CONFIG_IP6_NF_RAW=m
export CONFIG_NF_NAT_IPV6=m
export CONFIG_NF_NAT_MASQUERADE_IPV6=m
make ARCH=$ARCH CROSS_COMPILE=$CROSS_COMPILE -C $KSRC M=$MODULE clean
make ARCH=$ARCH CROSS_COMPILE=$CROSS_COMPILE -C $KSRC M=$MODULE6 clean
make ARCH=$ARCH CROSS_COMPILE=$CROSS_COMPILE -C $KSRC M=$MODULE modules -j 4
make ARCH=$ARCH CROSS_COMPILE=$CROSS_COMPILE -C $KSRC M=$MODULE6 modules -j 4
rm -rf build
mkdir build
mkdir build/ipset
cp $MODULE/*.ko build
cp $MODULE/ipset/*.ko build/ipset
cp $MODULE6/*.ko build
```

📝 编译之后ko文件在build目录中

## 编译iptables用户模块

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

📝 编译之后so文件在extensions子目录中
