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
## 第三週 07/20

## 第四週 08/03

```
echo $'apiVersion: v1
kind: Pod
metadata:
  name: myapp-pod
spec:
  containers:
  - name: myapp-container
    image: quay.io/cloudwalker/alp.base
    command: [\'sh\', \'-c\', \'echo The Pod is running && sleep 10\']
  restartPolicy: Never' > ~/myLifecyclePod-1.yaml
```

`kubectl describe pod myapp-pod | grep 'Status:'`
Status:           Failed
```
echo $'apiVersion: v1
kind: Pod
metadata:
  name: myapp-pod
spec:
  containers:
  - name: myapp-container
    image: quay.io/cloudwalker/alp.base
    command: [\'sh\', \'-c\', \'echo The Pod is running && exit 1\']
  restartPolicy: Never' > ~/myLifecyclePod-2.yaml
```

`kubectl get nodes --show-labels`
NAME                STATUS   ROLES           AGE   VERSION   LABELS
c30-control-plane   Ready    control-plane   13d   v1.30.0   app=taroko,beta.kubernetes.io/arch=amd64,beta.kubernetes.io/os=linux,ingress-ready=true,kubernetes.io/arch=amd64,kubernetes.io/hostname=c30-control-plane,kubernetes.io/os=linux,node-role.kubernetes.io/control-plane=,node.kubernetes.io/exclude-from-external-load-balancers=
c30-worker          Ready    <none>          13d   v1.30.0   beta.kubernetes.io/arch=amd64,beta.kubernetes.io/os=linux,disktype=ssd,kubernetes.io/arch=amd64,kubernetes.io/hostname=c30-worker,kubernetes.io/os=linux
c30-worker2         Ready    <none>          13d   v1.30.0   beta.kubernetes.io/arch=amd64,beta.kubernetes.io/os=linux,disk=spinning,kubernetes.io/arch=amd64,kubernetes.io/hostname=c30-worker2,kubernetes.io/os=linux
```
echo 'apiVersion: v1
kind: Pod
metadata:
  name: pton
spec:
  containers:
  - name: pton
    image: quay.io/cloudwalker/alp.base
    imagePullPolicy: Never
    tty: true
  nodeSelector:
    disktype: ssd '> ~/nodeselect.yaml
```


cat kind/bin/kto
image的備份檔透過kind部署到所有的node上
```
      sudo kind load image-archive ~/kindimg/flannel:v0.21.3.tar --name $1 &>/dev/null
```



kubectl run nginx-kusc00401 --image=nginx --dry-run=client -o yaml > nginx-kusc00401.yaml
$ ls
auth.json  canal.yaml  kind     metrics.yaml           myLifecyclePod-2.yaml  nodeselect.yaml
bin        cni         kindimg  myLifecyclePod-1.yaml  nginx-kusc00401.yaml
$ vi nginx-kusc00401.yaml
```
  nodeSelector:
    disk: spinning
```
$ ka -f nginx-kusc00401.yaml
pod/nginx-kusc00401 configured
$ kg all -o wide
NAME                  READY   STATUS    RESTARTS      AGE   IP            NODE          NOMINATED NODE   READINESS GATES
pod/kucc8             2/2     Running   0             13d   10.244.1.11   c30-worker    <none>           <none>
pod/myapp-pod         0/1     Error     0             70m   10.244.1.17   c30-worker    <none>           <none>
pod/nginx-kusc00401   1/1     Running   0             13d   10.244.2.13   c30-worker2   <none>           <none>
pod/pton              1/1     Running   0             22m   10.244.2.15   c30-worker2   <none>           <none>
pod/sharepid          2/2     Running   2 (93m ago)   13d   10.244.1.10   c30-worker    <none>           <none>

NAME                 TYPE        CLUSTER-IP   EXTERNAL-IP   PORT(S)   AGE   SELECTOR
service/kubernetes   ClusterIP   10.96.0.1    <none>        443/TCP   13d   <none>



$ kubectl taint nodes c30-control-plane node-role.kubernetes.io/control-plane:NoSchedule
node/c30-control-plane tainted
$ kubectl describe nodes | egrep "Taints:|Name:" | grep -A 1 c30-control-plane
Name:               c30-control-plane
Taints:             node-role.kubernetes.io/control-plane:NoSchedule
```
echo 'apiVersion: v1
kind: Pod
metadata:
  name: podtt
spec:
  containers:
  - name: podtt
    image: quay.io/cloudwalker/alp.base
    imagePullPolicy: IfNotPresent
    tty: true
  nodeSelector:
    kubernetes.io/hostname: c30-control-plane '> ~/podtt.yaml
```
$ ka -f podtt.yaml
pod/podtt created
$ kg all -o wide
NAME                  READY   STATUS    RESTARTS       AGE   IP            NODE          NOMINATED NODE   READINESS GATES
pod/kucc8             2/2     Running   0              13d   10.244.1.11   c30-worker    <none>           <none>
pod/myapp-pod         0/1     Error     0              96m   10.244.1.17   c30-worker    <none>           <none>
pod/nginx-kusc00401   1/1     Running   0              13d   10.244.2.13   c30-worker2   <none>           <none>
pod/podtt             0/1     Pending   0              3s    <none>        <none>        <none>           <none>
pod/pton              1/1     Running   0              48m   10.244.2.15   c30-worker2   <none>           <none>
pod/sharepid          2/2     Running   2 (118m ago)   13d   10.244.1.10   c30-worker    <none>           <none>

NAME                 TYPE        CLUSTER-IP   EXTERNAL-IP   PORT(S)   AGE   SELECTOR
service/kubernetes   ClusterIP   10.96.0.1    <none>        443/TCP   13d   <none>
$ kd pod/podtt
pod "podtt" deleted
$ vi podtt.yaml
```
  tolerations:
  - effect: NoSchedule
    operator: Exists
```
狀態顯示為拉鏡像，因為指派在control-plane上，先前kind設定檔中推送鏡像的目標機為worker，所以control-plane上沒有鏡像。
$ kg all -o wide
NAME                  READY   STATUS              RESTARTS       AGE    IP            NODE                NOMINATED NODE   READINESS GATES
pod/kucc8             2/2     Running             0              13d    10.244.1.11   c30-worker          <none>           <none>
pod/myapp-pod         0/1     Error               0              101m   10.244.1.17   c30-worker          <none>           <none>
pod/nginx-kusc00401   1/1     Running             0              13d    10.244.2.13   c30-worker2         <none>           <none>
pod/podtt             0/1     ContainerCreating   0              29s    <none>        c30-control-plane   <none>           <none>
pod/pton              1/1     Running             0              53m    10.244.2.15   c30-worker2         <none>           <none>
pod/sharepid          2/2     Running             2 (124m ago)   13d    10.244.1.10   c30-worker          <none>           <none>

NAME                 TYPE        CLUSTER-IP   EXTERNAL-IP   PORT(S)   AGE   SELECTOR
service/kubernetes   ClusterIP   10.96.0.1    <none>        443/TCP   13d   <none>
$ kg pods
NAME              READY   STATUS    RESTARTS       AGE
kucc8             2/2     Running   0              13d
myapp-pod         0/1     Error     0              102m
nginx-kusc00401   1/1     Running   0              13d
podtt             1/1     Running   0              77s
pton              1/1     Running   0              54m
sharepid          2/2     Running   2 (125m ago)   13d



$ kubectl get nodes | grep 'Ready'
c30-control-plane   Ready    control-plane   13d   v1.30.0
c30-worker          Ready    <none>          13d   v1.30.0
c30-worker2         Ready    <none>          13d   v1.30.0
$ kubectl describe nodes | grep 'NoSchedule'
Taints:             node-role.kubernetes.io/control-plane:NoSchedule
$ echo "2" > /opt/KUSC00402/kusc00402.txt


修改configMap
kubectl edit cm -n kube-system kubelet-config
