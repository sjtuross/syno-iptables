# 群晖实现v2rayA透明代理模式

安装采用Docker的方式，所以首先确认你的群晖系统是否支持Docker。

## 补齐缺失的iptables模块

按照 [v2rayA Docker方式安装](https://v2raya.org/docs/prologue/installation/docker/) 之后，启用redirect或tproxy透明代理模式时，类似旁路由模式，就会遇到iptables相关的报错。群晖系统是一个定制版Linux系统，透明代理需要的iptables相关模块并未包含，所以解决方案就是补齐缺失的iptables模块，即安装相关的ko和so模块。

## 加载缺失的模块并启动容器

在v2rayA启动时，为了确保所需的内核模块已经加载，可以覆盖默认的entrypoint为一个脚本，负责加载模块然后启动v2rayA，以下为docker run示例。

```bash
docker run -d \
  --restart=always \
  --privileged \
  --network=host \
  --name v2raya \
  -e V2RAYA_ADDRESS=0.0.0.0:2017 \
  -v /lib/modules:/lib/modules \
  -v /etc/resolv.conf:/etc/resolv.conf \
  -v /volume1/docker/v2raya-config:/etc/v2raya \
  --entrypoint /etc/v2raya/bootstrap.sh \
  mzz2017/v2raya
```

⚠️ 请替换 /volume1/docker/v2raya-config 为你自己挂载的配置目录

以DS3617xs 6.2.3-25426为例，bootstrap.sh文件内容如下，同样存放于配置目录中。

```bash
#!/bin/sh
insmod /lib/modules/nfnetlink.ko &> /dev/null
insmod /lib/modules/ip_set.ko &> /dev/null
insmod /lib/modules/ip_set_hash_ip.ko &> /dev/null
insmod /lib/modules/xt_set.ko &> /dev/null
insmod /lib/modules/ip_set_hash_net.ko &> /dev/null
insmod /lib/modules/xt_mark.ko &> /dev/null
insmod /lib/modules/xt_connmark.ko &> /dev/null
insmod /lib/modules/nf_tproxy_core.ko &> /dev/null
insmod /lib/modules/xt_TPROXY.ko &> /dev/null
insmod /lib/modules/iptable_mangle.ko &> /dev/null
v2raya
```

⚠️ 不同群晖系统所需的内核模块可能不完全一样，比如nf_tproxy_core.ko模块，只有3.X内核才会有。

更多关于v2rayA的使用，请访问[官方网站](https://v2raya.org/)。
