# SideStore 远程无感刷新全闭环部署指南 (sidestore-vpn)

本教程记录了如何通过 **sidestore-vpn** 实现 iOS 应用全网自动刷新。该方案支持内网 Wi-Fi 及外网（远程 Wi-Fi）通过 WireGuard 隧道进行免本地 VPN 的无感重签。

---

## 一、 环境架构

* **主路由器 (中兴)**: `10.10.10.1` (基础网关)
* **旁路由 (OpenWrt)**: `10.10.10.252` (WireGuard 节点、防火墙调度)
* **Linux 服务端 (Debian)**: `10.10.10.253` (Docker 运行 sidestore-vpn 工具)
* **iOS 设备虚拟网段**: `10.0.0.0/24` (手机通过 WG 拨入后的 IP)
* **SideStore 虚拟电脑 IP**: `10.7.0.1`

---

## 二、 服务端部署 (Debian)

### 1. 编写 Dockerfile

在 `/root/sidestore-vpn` 目录下创建 `Dockerfile`：dockerfile

```bash
# 第一阶段：编译

FROM rust:latest as builder
WORKDIR /usr/src/sidestore-vpn
RUN git clone https://github.com/xddxdd/sidestore-vpn.git . &&

cargo build --release

# 第二阶段：运行

FROM debian:bookworm-slim
WORKDIR /app
RUN apt-get update && apt-get install -y iproute2 ca-certificates && rm -rf /var/lib/apt/lists/*
COPY --from=builder /usr/src/sidestore-vpn/target/release/sidestore-vpn .
RUN chmod +x sidestore-vpn
ENTRYPOINT ["./sidestore-vpn"]
```


### 2. 构建与运行容器
执行以下命令启动服务：

```bash
# 构建镜像
docker build -t sidestore-vpn.

# 启动容器 (必须使用 host 模式并授予网络管理权限)
docker run -d \
  --name sidestore-vpn \
  --restart always \
  --network host \
  --cap-add=NET_ADMIN \
  --device /dev/net/tun:/dev/net/tun \
  sidestore-vpn

```

### 3. 内核参数调优与回程路由

在 Debian 宿主机执行，解决反射包被拦截和路径丢失问题 ：

```bash
# 开启 IP 转发
sudo sysctl -w net.ipv4.ip_forward=1

# 彻底关闭反向路径过滤 (rp_filter)，允许封包反射
sudo sysctl -w net.ipv4.conf.all.rp_filter=0
sudo sysctl -w net.ipv4.conf.default.rp_filter=0
sudo sysctl -w net.ipv4.conf.ens18.rp_filter=0
sudo sysctl -w net.ipv4.conf.sidestore.rp_filter=0

# 添加回程路由：确保反射包能通过旁路由回到手机虚拟 IP 段
sudo ip route add 10.0.0.0/24 via 10.10.10.252

```

---

## 三、 旁路由配置 (OpenWrt)

### 1. 添加静态路由

进入 **网络 -> 路由 -> 静态 IPv4 路由**，点击添加：

* **接口**: `lan`
* **目标**: `10.7.0.1/32`
* **网关**: `10.10.10.253` (Debian 机器 IP) 



### 2. 防火墙区域设置

进入 **网络 -> 防火墙 -> 区域设置**，为 WireGuard 建立名为 `sidestore` 的新区域：

* **覆盖网络**: 选择 `wg0`
* **入站/出站/转发**: 全部设为 `接受`
* **允许转发到目标区域**: 勾选 `lan`
* **允许来自源区域的转发**: 勾选 `lan`
* **IP 动态伪装 (Masquerading)**: **务必取消勾选** (全局伪装会破坏反射包的源 IP) 



### 3. 配置精准 NAT 规则 (解决断网)

为了让手机能正常上网，进入 **防火墙 -> NAT 规则**，添加：

* **出站区域**: `lan`
* **源地址**: `10.0.0.0/24`
* **目标地址**: 输入 `!10.7.0.1` (或在“取反/排除”选项打钩)
* **操作**: `MASQUERADE`
* **目的**: 对上网流量进行伪装，但唯独放过发往 SideStore 的反射流量。

### 4. 代理插件排除

在 PassWall 或 OpenClash 的 **“直连列表”** 或 **“不代理 IP 列表”** 中添加 `10.7.0.1`。

---

## 四、 iOS 客户端配置

### WireGuard 设置

编辑手机端配置文件：

* **Allowed IPs**: 设置为 `0.0.0.0/0` (确保所有 Apple 验证流量都能回家处理) 


* **DNS**: 填写家里的 OpenWrt IP `10.10.10.252`
* **Persistent Keepalive**: 设置为 `25` 秒 (保持隧道活跃)

---

## 五、 验证成功标志

1. **手机端**: 在 SideStore 点击 **Refresh All**，进度条无延迟走动并完成。
2. **服务端**: 执行 `sudo tcpdump -i any host 10.7.0.1 -n` 观察到成对的包：
* `10.0.0.2 > 10.7.0.1` (请求进入)
* `10.7.0.1 > 10.0.0.2` (反射成功) 





---

**完成！** 现在你已拥有一套稳健的远程无感刷新系统。
