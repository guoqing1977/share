明白了，你的场景是：已经有一个正在运行的 bond0（手动配置了静态 IP），现在想把这个 bond0 上的 IP “迁移”到 br0 上，并且 bond0 继续作为聚合链路，只是不再持有 IP。下面给出针对“已有手动配置 bond0”的改造步骤。

⚠️ 关键前提

· 你的 bond0 目前是 手动配置文件（如 /etc/sysconfig/network-scripts/ifcfg-bond0）还是由 NetworkManager 管理？建议先确认。
  ```bash
  nmcli connection show | grep bond0
  ```
  · 如果没有输出，说明 NetworkManager 未接管，需要先导入。
  · 如果有输出，说明已被 NetworkManager 管理，可以直接修改。

下面按 NetworkManager 接管 的方式操作（现代麒麟系统推荐），这样修改持久且不易出错。

---

✅ 正确步骤（已有 bond0 静态 IP → 迁移到 br0）

1️⃣ 记录 bond0 的静态 IP 信息

```bash
# 查看 bond0 当前配置（如果是手动配置，查看文件）
cat /etc/sysconfig/network-scripts/ifcfg-bond0
# 或者用 ip 命令查看
ip addr show bond0
```

记下 IP、掩码、网关、DNS。

2️⃣ 让 NetworkManager 接管现有的 bond0（如果尚未接管）

```bash
# 创建 bond0 的 NetworkManager 连接（复用现有接口，不改变其 bond 模式等）
nmcli connection add type bond ifname bond0 con-name bond0 \
bond.options "mode=active-backup,miimon=100"
# 将物理网卡加入 bond0（若原手动配置中已绑定，这里需指定相同网卡）
nmcli connection add type ethernet ifname eth0 con-name bond-slave-eth0 master bond0
nmcli connection add type ethernet ifname eth1 con-name bond-slave-eth1 master bond0
```

注意：替换 eth0、eth1 为实际网卡名。如果原 bond0 配置中已经有两块网卡，这里添加后 NetworkManager 会接管。

3️⃣ 创建 br0 网桥（继承 bond0 的静态 IP，并开启 STP）

```bash
nmcli connection add type bridge ifname br0 con-name br0 \
ipv4.method manual ipv4.addresses <原IP/掩码> \
ipv4.gateway <原网关> ipv4.dns <原DNS> \
bridge.stp yes connection.autoconnect yes
```

4️⃣ 将 bond0 的 IP 清除，并设置为 br0 的从属

```bash
# 清除 bond0 上的 IP 配置
nmcli connection modify bond0 ipv4.method disabled
# 将 bond0 加入网桥 br0
nmcli connection modify bond0 master br0
```

5️⃣ 按正确顺序激活连接

```bash
# 先激活底层（网卡 slave 和 bond0）
nmcli connection up bond-slave-eth0
nmcli connection up bond-slave-eth1
nmcli connection up bond0
# 再激活网桥
nmcli connection up br0
```

6️⃣ 验证

```bash
# IP 现在应该在 br0 上
ip addr show br0
# bond0 应该没有 IP
ip addr show bond0
# 网桥成员
bridge link show
# 连通性
ping -c 4 <网关>
# 查看 bond0 聚合状态
cat /proc/net/bonding/bond0
```

---

🔁 回滚方案（如果失败）

```bash
# 将 bond0 移出 br0，恢复其 IP
nmcli connection modify bond0 master ""
nmcli connection modify bond0 ipv4.method manual ipv4.addresses <原IP/掩码> ipv4.gateway <原网关> ipv4.dns <原DNS>
nmcli connection down br0; nmcli connection up bond0
```

---

❗ 特别注意

· STP 必须开启：bridge.stp yes 防止环路（你之前遇到的交换机端口关闭问题很大程度上源于此）。
· 不要额外创建 “br0-bond-slave” 接口：直接将 bond0 的 master 设为 br0 即可。
· 确保手动配置不再冲突：备份并移除 /etc/sysconfig/network-scripts/ifcfg-bond0，否则 NetworkManager 可能读取两份配置。

如果执行过程中遇到“连接已存在”等错误，可以先用 nmcli connection delete 删除旧的连接再重新创建。建议在维护窗口或通过带外管理进行操作。
