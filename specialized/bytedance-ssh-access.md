# ByteDance SSH 访问

用于登录 ByteDance 内部机器，尤其是 root 登录、Kerberos/GSSAPI、IPv6 目标、known_hosts 变化、`.k5login` 授权和 `jump.byted.org` 二跳。

## 直接登录

先尝试非交互 SSH：

```bash
ssh -o BatchMode=yes -o ConnectTimeout=8 root@HOST 'hostname; uname -a; whoami'
```

IPv6 目标使用 `-6` 并引用 target：

```bash
ssh -6 -o BatchMode=yes -o ConnectTimeout=8 'root@IPv6_ADDR' 'hostname; uname -a; whoami'
```

## 常见失败处理

host key 变化且用户确认机器重建或 key 应刷新时：

```bash
ssh-keygen -f "$HOME/.ssh/known_hosts" -R "HOST"
ssh -o StrictHostKeyChecking=accept-new -o BatchMode=yes -o ConnectTimeout=8 root@HOST 'hostname; uname -a; whoami'
```

GSSAPI/Kerberos 失败时先查本地 ticket 和 SSH 配置：

```bash
klist
ssh -G root@HOST | rg -i 'gssapi|preferredauth|user|hostname'
```

如果本地有有效 `USER@BYTEDANCE.COM` ticket 但 root 登录被拒，通常需要目标机 `/root/.k5login` 授权该 principal。

## IPv6 网络区分

ICMPv6 可达但 SSH 超时时，先区分路由、主机和 TCP/22：

```bash
ip -6 route get IPv6_ADDR
ping -6 -c 3 -W 2 IPv6_ADDR
```

如能登录目标机，再检查：

```bash
ss -lntp | grep ':22'
ip6tables -S
```

必要时用源地址过滤 tcpdump，判断 TCP/22 SYN 是否到达目标：

```bash
tcpdump -ni any 'ip6 and src SOURCE_IPV6 and dst TARGET_IPV6 and tcp port 22'
```

## jump.byted.org

`jump.byted.org` 可能拒绝标准 `ProxyJump` 和远程 exec：

```text
server not allowed channel type: direct-tcpip
exec request failed on channel 0
```

这种情况用交互 TTY：

```bash
ssh -t jump.byted.org
ssh -6 root@IPv6_ADDR
```

二跳后跑简单命令，避免复杂 shell quoting 被 jump 环境破坏：

```bash
ssh -6 root@IPv6_ADDR hostname
ssh -6 root@IPv6_ADDR uname -a
ssh -6 root@IPv6_ADDR whoami
```
