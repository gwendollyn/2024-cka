# 2024-cka

## 第二週 07/13
編輯乙太網路設定 `/etc/network/interfaces`，並將vm設定從NAT改為連接到橋接器Bridge後重開機
```
auto lo
iface lo inet loopback

auto eth0
#iface eth0 inet dhcp

iface eth0 inet static
        address 120.96.143.193
        netmask 255.255.255.0
        gateway 120.96.143.254
```
### workload
`workload`: 多個pod組成一個workload，用一堆workload resources讓Workload長長久久穩穩當當
Workload目標
- Replication of components (**HA**)
- Auto-scaling
- Load balancing

- Rolling updates
可觀察，監控，prometheus+grafana圖形介面監控
- Logging across components
- Monitoring and health checking

- Service discovery: 名稱解析，用名稱溝通而不是IP。
- Authentication: 憑證系統，k8s全部用憑證作業，CKA及CKAD應該有2題。

`workload resource`: Controllers
- Deployment and ReplicaSet
- StatefulSet
- DaemonSet
- Job and CronJob
