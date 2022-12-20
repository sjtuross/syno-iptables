# How to self compile
This repository provides precompiled modules for some Synology NAS modesl but not for all.
In case there are no precompiled modules for your model or for any other reason you can self compile the modules  

The following guide will focus on compiling for Synology DSM920+ with a processor architecture "geminilake"
check your architecture beforehand and adjust accordingly: https://github.com/SynoCommunity/spksrc/wiki/Synology-and-SynoCommunity-Package-Architectures 

!!! Pre-requisite: docker up and running !!!

### Set up compilation environment
Use spksrc tooling to compile things for synology DSM. (download the repo & run the spksrc docker image and enter it's bash)

```
git clone --depth 1 https://github.com/SynoCommunity/spksrc.git && \
sudo docker run -it --rm -u $(id -u ${USER}):$(id -g ${USER}) -v $(pwd)/spksrc:/spksrc ghcr.io/synocommunity/spksrc /bin/bash
```

### Prepare & compile the specific
```
make setup
```
```
cd /spksrc/kernel/syno-geminilake-7.0
```
```
make
```
```
cd /spksrc/toolchain/syno-geminilake-7.1
```
```
make
```
```
cd /spksrc/kernel/syno-geminilake-7.0/work/linux/
```
Uncomment YYLTYPE yylloc to prevent double declaration - https://github.com/SynoCommunity/spksrc/issues/5465
```
sed -i 's/^YYLTYPE yylloc$/\/\/&/' /spksrc/kernel/syno-geminilake-7.0/work/linux/scripts/dtc/dtc-parser.tab.c_shipped
```
Continue compilation prep
```
make prepare
```
```
make modules_prepare
```

### Set cross-compilation environment variables (x86_64 platform)
 ```
export PATH="${PATH}:/spksrc/toolchain/syno-geminilake-7.1/work/x86_64-pc-linux-gnu/bin"
export CROSS=x86_64-pc-linux-gnu
export CC=${CROSS}-gcc
export LD=${CROSS}-ld
export AS=${CROSS}-as
export CXX=${CROSS}-g++
export CROSS_COMPILE=/spksrc/toolchain/syno-geminilake-7.1/work/x86_64-pc-linux-gnu/bin/x86_64-pc-linux-gnu-
export ARCH=x86_64
export KSRC=/spksrc/kernel/syno-geminilake-7.0/work/linux
```

### Compile the netfilter kernel module
```
cd /spksrc/kernel/syno-geminilake-7.0/work/
MODULE=net/netfilter
MODULE6=net/ipv6/netfilter
LIB=lib
export CONFIG_NETFILTER_XT_CONNMARK=m
export CONFIG_NETFILTER_XT_MATCH_COMMENT=m
export CONFIG_NETFILTER_XT_MATCH_SOCKET=m
export CONFIG_NETFILTER_XT_MATCH_STRING=m
export CONFIG_NETFILTER_TPROXY=m
export CONFIG_NETFILTER_XT_TARGET_TPROXY=m
export CONFIG_IP6_NF_TARGET_MASQUERADE=m
export CONFIG_IP6_NF_NAT=m
export CONFIG_IP6_NF_RAW=m
export CONFIG_NF_NAT_IPV6=m
export CONFIG_NF_NAT_MASQUERADE_IPV6=m
export CONFIG_TEXTSEARCH=m
export CONFIG_TEXTSEARCH_BM=m
make ARCH=$ARCH CROSS_COMPILE=$CROSS_COMPILE -C $KSRC M=$MODULE clean
make ARCH=$ARCH CROSS_COMPILE=$CROSS_COMPILE -C $KSRC M=$MODULE6 clean
make ARCH=$ARCH CROSS_COMPILE=$CROSS_COMPILE -C $KSRC M=$LIB clean
make ARCH=$ARCH CROSS_COMPILE=$CROSS_COMPILE -C $KSRC M=$MODULE modules -j 4
make ARCH=$ARCH CROSS_COMPILE=$CROSS_COMPILE -C $KSRC M=$MODULE6 modules -j 4
make ARCH=$ARCH CROSS_COMPILE=$CROSS_COMPILE -C $KSRC M=$LIB modules -j 4
rm -rf build
mkdir build
mkdir build/ipset
cp linux/$MODULE/*.ko build
cp linux/$MODULE/ipset/*.ko build/ipset
cp linux/$MODULE6/*.ko build
cp linux/$LIB/*.ko build
find build/ -iname "*.ko" -exec ${CROSS_COMPILE}strip --strip-unneeded {} \;
```

### Download & Compile iptables user module
iptables v1.8.3, suitable for syno-apollolake-7.0, please adjust by yourself
```
wget https://www.netfilter.org/pub/iptables/iptables-1.8.3.tar.bz2
tar xjf iptables-1.8.3.tar.bz2
cd iptables-1.8.3
./configure --host=$CROSS --prefix="/spksrc/toolchain/syno-geminilake-7.0/work/build" --disable-nftables
make && make install
```

📝After compilation, the ko files can be found in /spksrc/kernel/syno-geminilake-7.0/work/builddirectory

📝After compilation, the so files can be found in /spksrc/kernel/syno-geminilake-7.0/work/iptables-1.8.3/extensions


# ARM notes

### Set cross-compilation environment variables (arm64 platform)
```
export PATH="${PATH}:/spksrc/toolchain/syno-rtd1296-7.0/work/aarch64-unknown-linux-gnu/bin"
export CROSS=aarch64-unknown-linux-gnu
export CC=${CROSS}-gcc
export LD=${CROSS}-ld
export AS=${CROSS}-as
export CXX=${CROSS}-g++
export CROSS_COMPILE=/spksrc/toolchain/syno-rtd1296-7.0/work/aarch64-unknown-linux-gnu/bin/aarch64-unknown-linux-gnu-
export ARCH=arm64
export KSRC=/spksrc/kernel/syno-rtd1296-7.0/work/linux
```
