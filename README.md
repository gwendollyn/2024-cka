# 2024-cka
## 第一週 07/06
- mobe ![image](https://github.com/user-attachments/assets/b391de27-2352-41f4-bd5b-34b65ed1f57b)
- docker 碼頭工人 ![image](https://github.com/user-attachments/assets/6629ebb5-513c-4f04-b74b-7f9ffdfb69fb)

- Application container 程式貨櫃
隔離沒有hypervisor，虛擬電腦
Control group 規範container只能用多少CPU記憶體
Chroot+overlay2使container有自己獨立的檔案系統

Image -> Application container
- Runc (go)
- Crun (C)
- runsc/gvisor (go)
- kata/vm (qemu)


## 第二週 07/13
編輯乙太網路設定 `/etc/network/interfaces`，並將vm設定從NAT改為連接到橋接器Bridge後重開機
```
auto lo
iface lo inet loopback

#auto eth0
#iface eth0 inet dhcp

auto eth0
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

workload運作架構
![image](https://github.com/user-attachments/assets/d2f43a52-2a55-4a1f-93a0-324680cfcb9a)
- `rs`: ReplicaSet，由Deployment Object產生，deployment負責進/退版，rs則產生workload
- `workload關機`: rs把pod全都關機
- `hpa`: autoscaling監測workload，若負載超過80%
- `cm`: ConfigMaps設定檔存放物件
- `secret`: 加密保護帳號密碼
**資料跟設定分開**
  `pv` & `pvc`: 不經過模擬層，直接使用idc的檔案系統 (e.g., local, nfs, iscsi)，融合檔案系統 [ceph](https://ceph.io/en/) (需要網路100G$$$$$)，替代[minio](https://min.io/)
  `svc`: k8s的service，kube-proxy做，可以做多套，IP就會多，就需要一個reverse proxy來連接該多套服務。
  `ing`: 前述的reverse proxy，ingress虛擬入口。
  `MetalLB`: 讓這個IP真的有對外連接的能力。最一開始推出的k8s自建的沒有這功能，一定要上雲才有。
  `ns`: namespace，辦公室， \textcolor{red}{red} 每個workload一定要有自己的辦公室。否則會影響別人的workload。
  `netpol`: network policy防火牆，要套用在ns上
  `limits`: 先押一個數字進去，如果真的有反應很慢，再來慢慢增加。
  `quota`: 一個ns裡最多只能做30 pod, 6 pv, 3 svc。

啟動叢集
```
kci c30
```

登入k8s的其中一個pod
```
ssh bigred@120.96.143.x -p 22100
mkdir -p wulin/yaml
kubectl run echoserver --image=registry.k8s.io/echoserver:1.10 --restart=Never --port 8080
# 映射port是為了之後控制svc找得到這個pod
kubectl get pods
```
# 注意到`READY`欄位的狀態是1/1，代表真正有上線。Pod裡只要有container在跑，`STATUS`就是`Running`，但是程式真的可以用，是要看`READY`。
NAME         READY   STATUS    RESTARTS   AGE
echoserver   1/1     Running   0          62s

```
kubectl get pods -o wide
```

NAME         READY   STATUS    RESTARTS   AGE     IP            NODE         NOMINATED NODE   READINESS GATES
echoserver   1/1     Running   0          4m36s   10.244.1.15   c30-worker   <none>           <none>

```
ifconfig podman1
kubectl exec echoserver -- hostname -i
**1.24版後命令行格式有修改，kubectl及命令之間要以--分隔
kgip echoserver; echo
curl -s http://`kgip echoserver`:8080 | grep Hostname
kubectl exec -it echoserver -- sh
**bak程式需要有一個終端機**
kau 101
**
Linux user c30-101 ok
namespace/c30-101 created
serviceaccount/c30-101 created
secret/c30-101-secret created
role.rbac.authorization.k8s.io/c30-101-role created
rolebinding.rbac.authorization.k8s.io/c30-101-rbind created
clusterrole.rbac.authorization.k8s.io/c30-101-clusterole created
clusterrolebinding.rbac.authorization.k8s.io/c30-101-clusterbind created
K8S Service Account c30-101 ok
c30-101 secret dkreg ok
**
klu
[Cluster Users]
c30-101:si9wV!Oc
```


```
ssh c30-101@120.96.143.193 -p 22100
**
c30-101@120.96.143.193's password:
Welcome to Taroko Kadm Console v24.07 (task,kubectl,talosctl,mc,rclone,s3fs,JuiceFS Client)
**
kubectl run e1 --image=registry.k8s.io/echoserver:1.10 --port 8080
```
