# 群晖Docker安装OpenWrt旁路由

## 测试环境

* DSM 7.0.1-42218 Update 2
* eSir Buddha OpenWrt [佛跳墙容器固件](https://hub.docker.com/r/esirpg/buddha)
* ShadowSocksR Plus+ 或 PassWall

⚠️ 依赖 xt_comment.ko，以及 xt_socket.ko（支持TPROXY模式）

## 安装OpenWrt

使用`ifconfig`确认网卡标识符，然后开启混杂模式

```bash
ip link set eth0 promisc on
```

创建一个docker macvlan网络，请自行调整网段，主路由网关地址，以及群晖网卡标识符

```bash
docker network create -d macvlan --subnet=192.168.1.0/24 --gateway 192.168.1.1 -o parent=eth0 mac-net
```

提前拉取OpenWrt容器固件

```bash
docker pull esirpg/buddha:latest
```

启动OpenWrt容器固件，指定旁路由IP 192.168.1.2

```bash
docker run -d \
  --restart always \
  --name esirpg-buddha \
  --privileged \
  -v /volume1/docker/openwrt-config:/etc/config \
  --network mac-net \
  --ip=192.168.1.2 \
  esirpg/buddha:latest \
  /sbin/init
```

⚠️ 可以选择挂载之前备份的config目录，容器重建后绝大部分配置信息都在，一般初次安装不要挂载

进入容器设置OpenWrt LAN IP 192.168.1.2

```bash
docker exec -it esirpg-buddha ash
vi /etc/config/network
/etc/init.d/network restart
```

## 配置OpenWrt

访问 [http://192.168.1.2](http://192.168.1.2)，在luci中进行后续设置

* 设置LAN接口网关为主路由网关192.168.1.1，并关闭DHCP服务
* 防火墙基本设置中，转发务必设置为接受，LAN区域也都设置为接受
* DHCP/DNS设置中设置DNS转发地址为任意上游DNS，并关闭重绑定保护（为了能够解析本地域名）
* 删除WAN，VPN等所有其他接口和防火墙区域，只需保留LAN即可
* 按照常规步骤配置 ShadowSocksR Plus+ 或 PassWall
* 如果使用 PassWall TPROXY 模式，除了需要 xt_socket.ko 模块之外，经测试还需禁用LAN接口桥接
