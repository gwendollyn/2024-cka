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

---
### DNS
$ kubectl -n kube-system get svc
NAME             TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)                  AGE
dkreg            ClusterIP   10.101.231.189   <none>        5000/TCP                 13d
kadm             ClusterIP   10.110.95.219    <none>        22100/TCP                13d
kube-dns         ClusterIP   10.96.0.10       <none>        53/UDP,53/TCP,9153/TCP   14d
metrics-server   ClusterIP   10.108.217.205   <none>        443/TCP                  14d

$ kubectl -n kube-system get endpoints
NAME             ENDPOINTS                                                  AGE
dkreg            10.244.0.11:5000                                           14d
kadm             10.244.0.11:22100                                          14d
kube-dns         10.244.0.13:53,10.244.0.14:53,10.244.0.13:53 + 3 more...   14d
metrics-server   10.244.2.12:10250                                          14d

$ kubectl get pods -n kube-system -o wide
NAME                                        READY   STATUS    RESTARTS        AGE   IP            NODE                NOMINATED NODE   READINESS GATES
calico-kube-controllers-ddf655445-fdjk7     1/1     Running   1 (4h17m ago)   14d   10.244.0.17   c30-control-plane   <none>           <none>
canal-77qg4                                 1/2     Running   2 (4h17m ago)   14d   10.89.0.2     c30-control-plane   <none>           <none>
canal-hqvc8                                 1/2     Running   2 (4h17m ago)   14d   10.89.0.3     c30-worker2         <none>           <none>
canal-lh9zf                                 1/2     Running   2 (4h17m ago)   14d   10.89.0.4     c30-worker          <none>           <none>
coredns-7db6d8ff4d-sch2j                    1/1     Running   1 (4h17m ago)   14d   10.244.0.13   c30-control-plane   <none>           <none>
coredns-7db6d8ff4d-whfdt                    1/1     Running   1 (4h17m ago)   14d   10.244.0.14   c30-control-plane   <none>           <none>
etcd-c30-control-plane                      1/1     Running   1 (4h17m ago)   14d   10.89.0.2     c30-control-plane   <none>           <none>
kube-apiserver-c30-control-plane            1/1     Running   1 (4h17m ago)   14d   10.89.0.2     c30-control-plane   <none>           <none>
kube-controller-manager-c30-control-plane   1/1     Running   1 (4h17m ago)   14d   10.89.0.2     c30-control-plane   <none>           <none>
kube-kadm                                   2/2     Running   2 (4h17m ago)   14d   10.244.0.11   c30-control-plane   <none>           <none>
kube-proxy-ddh5p                            1/1     Running   1 (4h17m ago)   14d   10.89.0.2     c30-control-plane   <none>           <none>
kube-proxy-jhmjx                            1/1     Running   1 (4h17m ago)   14d   10.89.0.3     c30-worker2         <none>           <none>
kube-proxy-l9c75                            1/1     Running   1 (4h17m ago)   14d   10.89.0.4     c30-worker          <none>           <none>
kube-scheduler-c30-control-plane            1/1     Running   1 (4h17m ago)   14d   10.89.0.2     c30-control-plane   <none>           <none>
metrics-server-d994c478f-xshzs              1/1     Running   3 (4h15m ago)   14d   10.244.2.12   c30-worker2         <none>           <none>

$ kg svc kube-dns -n kube-system --show-labels
NAME       TYPE        CLUSTER-IP   EXTERNAL-IP   PORT(S)                  AGE   LABELS
kube-dns   ClusterIP   10.96.0.10   <none>        53/UDP,53/TCP,9153/TCP   14d   k8s-app=kube-dns,kubernetes.io/cluster-service=true,kubernetes.io/name=CoreDNS
服務的dns系統靠這個標籤去定位到下面的pod
k8s叢集重開，服務的IP位置不會換
$ kg pods coredns-7db6d8ff4d-sch2j -n kube-system --show-labels
NAME                       READY   STATUS    RESTARTS        AGE   LABELS
coredns-7db6d8ff4d-sch2j   1/1     Running   1 (4h35m ago)   14d   k8s-app=kube-dns,pod-template-hash=7db6d8ff4d
$ kg pods coredns-7db6d8ff4d-whfdt -n kube-system --show-labels
NAME                       READY   STATUS    RESTARTS        AGE   LABELS
coredns-7db6d8ff4d-whfdt   1/1     Running   1 (4h35m ago)   14d   k8s-app=kube-dns,pod-template-hash=7db6d8ff4d

$ env
KUBERNETES_SERVICE_PORT=443
MAIL=/var/mail/bigred
USER=bigred
SSH_CLIENT=120.96.143.193 53862 22100
SHLVL=1
HOME=/home/bigred
SSH_TTY=/dev/pts/1
PAGER=less
PS1=\u@\h:\w$
NOW=--force --grace-period 0
LOGNAME=bigred
KUBE_EDITOR=nano
TERM=xterm-256color
LC_COLLATE=C
PATH=/home/bigred/wulin/wk/usdt/bin:/home/bigred/wulin/bin:/home/bigred/.krew/bin:/home/bigred/wulin/bin:/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin
LANG=C.UTF-8
SHELL=/bin/ash
KUBERNETES_SERVICE_PORT_HTTPS=443
POD_NAMESPACE=default
PWD=/home/bigred
KUBERNETES_SERVICE_HOST=kubernetes.default
SSH_CONNECTION=120.96.143.193 53862 10.244.0.11 22100
CHARSET=UTF-8
TZ=Asia/Taipei


$ kg svc kubernetes -n default --show-labels
NAME         TYPE        CLUSTER-IP   EXTERNAL-IP   PORT(S)   AGE   LABELS
kubernetes   ClusterIP   10.96.0.1    <none>        443/TCP   14d   component=apiserver,provider=kubernetes
$ kubectl -n default get svc
NAME         TYPE        CLUSTER-IP   EXTERNAL-IP   PORT(S)   AGE
kubernetes   ClusterIP   10.96.0.1    <none>        443/TCP   14d
$ kubectl describe svc
Name:              kubernetes
Namespace:         default
Labels:            component=apiserver
                   provider=kubernetes
Annotations:       <none>
Selector:          <none>
Type:              ClusterIP
IP Family Policy:  SingleStack
IP Families:       IPv4
IP:                10.96.0.1
IPs:               10.96.0.1
Port:              https  443/TCP
TargetPort:        6443/TCP
Endpoints:         10.89.0.2:6443
Session Affinity:  None
Events:            <none>


$ nslookup
> server 10.96.0.10
Default server: 10.96.0.10
Address: 10.96.0.10#53
> kubernetes.default.svc.k1.org
Server:         10.96.0.10
Address:        10.96.0.10#53

Name:   kubernetes.default.svc.k1.org
Address: 10.96.0.1
> dkreg.kube-system.svc.k1.org
Server:         10.96.0.10
Address:        10.96.0.10#53

Name:   dkreg.kube-system.svc.k1.org
Address: 10.101.231.189
> server 8.8.8.8
Default server: 8.8.8.8
Address: 8.8.8.8#53
> dkreg.kube-system.svc.k1.org
Server:         8.8.8.8
Address:        8.8.8.8#53

Non-authoritative answer:
Name:   dkreg.kube-system.svc.k1.org.kube-system.svc.k1.org
Address: 13.248.169.48
Name:   dkreg.kube-system.svc.k1.org.kube-system.svc.k1.org
Address: 76.223.54.146
> dkreg.kube-system.svc.k1.org.
Server:         8.8.8.8
Address:        8.8.8.8#53

Non-authoritative answer:
Name:   dkreg.kube-system.svc.k1.org
Address: 76.223.54.146
Name:   dkreg.kube-system.svc.k1.org
Address: 13.248.169.48

> set type=ns
> ibm.com.
Server:         8.8.8.8
Address:        8.8.8.8#53

Non-authoritative answer:
ibm.com nameserver = usw2.akam.net.
ibm.com nameserver = eur5.akam.net.
ibm.com nameserver = usc2.akam.net.
ibm.com nameserver = eur2.akam.net.
ibm.com nameserver = ns1-206.akam.net.
ibm.com nameserver = asia3.akam.net.
ibm.com nameserver = ns1-99.akam.net.
ibm.com nameserver = usc3.akam.net.

Authoritative answers can be found from:
> ntu.edu.tw.
Server:         8.8.8.8
Address:        8.8.8.8#53

Non-authoritative answer:
ntu.edu.tw      nameserver = ntu3.ntu.edu.tw.
ntu.edu.tw      nameserver = dns.ntu.edu.tw.
ntu.edu.tw      nameserver = dns.tp1rc.edu.tw.

Authoritative answers can be found from:



$ ls wulin/images/sshd/
Dockerfile
bigred@kube-kadm:~$ cat wulin/images/sshd/Dockerfile
ARG VER=0.0
FROM quay.io/cloudwalker/alp.base
RUN apk update && apk upgrade && \
    apk add --no-cache openssh-server tzdata && \
    # 設定時區
    cp /usr/share/zoneinfo/Asia/Taipei /etc/localtime && \
    ssh-keygen -t rsa -P "" -f /etc/ssh/ssh_host_rsa_key && \
    echo -e 'Welcome to ALP SSHd 6000\n' > /etc/motd && \
    # 建立管理者帳號 bigred
    adduser -s /bin/bash -h /home/bigred -G wheel -D bigred && \
    echo "%wheel ALL=(ALL) NOPASSWD: ALL" >> /etc/sudoers && \
    echo -e 'bigred\nbigred\n' | passwd bigred &>/dev/null && \
    rm /sbin/reboot && rm /usr/bin/killall

EXPOSE 22

ENTRYPOINT ["/usr/sbin/sshd"]
CMD ["-D"]
bigred@kube-kadm:~$ sudo podman build --tls-verify=false --format=docker --no-cache --force-rm --squash -t dkreg.kube-system:5000/alp.sshd:24.01 wulin/images/sshd/
STEP 1/5: FROM quay.io/cloudwalker/alp.base
Trying to pull quay.io/cloudwalker/alp.base:latest...
Getting image source signatures
Copying blob 19934a922dcd done   |
Copying config ed3b7bfb38 done   |
Writing manifest to image destination
STEP 2/5: RUN apk update && apk upgrade &&     apk add --no-cache openssh-server tzdata &&     cp /usr/share/zoneinfo/Asia/Taipei /etc/localtime &&     ssh-keygen -t rsa -P "" -f /etc/ssh/ssh_host_rsa_key &&     echo -e 'Welcome to ALP SSHd 6000\n' > /etc/motd &&     adduser -s /bin/bash -h /home/bigred -G wheel -D bigred &&     echo "%wheel ALL=(ALL) NOPASSWD: ALL" >> /etc/sudoers &&     echo -e 'bigred\nbigred\n' | passwd bigred &>/dev/null &&     rm /sbin/reboot && rm /usr/bin/killall
fetch https://dl-cdn.alpinelinux.org/alpine/v3.20/main/x86_64/APKINDEX.tar.gz
fetch https://dl-cdn.alpinelinux.org/alpine/v3.20/community/x86_64/APKINDEX.tar.gz
v3.20.2-45-g194e307d75d [https://dl-cdn.alpinelinux.org/alpine/v3.20/main]
v3.20.2-58-g730bd64fe94 [https://dl-cdn.alpinelinux.org/alpine/v3.20/community]
OK: 24156 distinct packages available
OK: 554 MiB in 161 packages
fetch https://dl-cdn.alpinelinux.org/alpine/v3.20/main/x86_64/APKINDEX.tar.gz
fetch https://dl-cdn.alpinelinux.org/alpine/v3.20/community/x86_64/APKINDEX.tar.gz
(1/3) Installing openssh-server-common (9.7_p1-r4)
(2/3) Installing openssh-server (9.7_p1-r4)
(3/3) Installing tzdata (2024a-r1)
Executing busybox-1.36.1-r29.trigger
OK: 556 MiB in 164 packages
Generating public/private rsa key pair.
Your identification has been saved in /etc/ssh/ssh_host_rsa_key
Your public key has been saved in /etc/ssh/ssh_host_rsa_key.pub
The key fingerprint is:
SHA256:TsycLDORezJXdmQje4VxT1C7UIkXW1ZZajuugr3OJjs root@kube-kadm
The key's randomart image is:
+---[RSA 3072]----+
|          . =o==&|
|       .   =.=.O+|
|      o   + o.+o.|
|       B + o ....|
|      B S     o. |
|       @     . . |
|        .o    .  |
|        E.+  .   |
|        .*+o.    |
+----[SHA256]-----+
STEP 3/5: EXPOSE 22
STEP 4/5: ENTRYPOINT ["/usr/sbin/sshd"]
STEP 5/5: CMD ["-D"]
COMMIT dkreg.kube-system:5000/alp.sshd:24.01
--> 48b6c9a1acf2
Successfully tagged dkreg.kube-system:5000/alp.sshd:24.01
48b6c9a1acf2fabca57eed87bf09a722b2677a1e1d29319730edc58052b43845
bigred@kube-kadm:~$ docker images
REPOSITORY                       TAG         IMAGE ID      CREATED         SIZE
dkreg.kube-system:5000/alp.sshd  24.01       48b6c9a1acf2  15 seconds ago  782 MB
quay.io/cloudwalker/alp.base     latest      ed3b7bfb3887  46 hours ago    778 MB



$ sudo podman push --tls-verify=false --creds=bigred:bigred dkreg.kube-system:5000/alp.sshd:24.01
Getting image source signatures
Copying blob a8a0fa710f76 done   |
Copying blob 10d4de6521e0 done   |
Copying config 48b6c9a1ac done   |
Writing manifest to image destination

$ kubectl run g1 --image=dkreg.kube-system:5000/alp.sshd:24.01 --image-pull-policy=Always --port=22 -l="app=g1"
pod/g1 created
$ kubectl create service clusterip g1 --tcp=22101:22
service/g1 created
$ kg all
NAME           READY   STATUS    RESTARTS        AGE
pod/g1         1/1     Running   0               4s
pod/sharepid   2/2     Running   2 (6h30m ago)   14d

NAME                 TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)     AGE
service/g1           ClusterIP   10.99.126.115   <none>        22101/TCP   3m5s
service/kubernetes   ClusterIP   10.96.0.1       <none>        443/TCP     14d
$ ssh bigred@g1.default -p 22101
Warning: Permanently added '[g1.default]:22101' (RSA) to the list of known hosts.
bigred@g1.default's password:
Welcome to ALP SSHd 6000

g1:~$


C:\Users\rbean>ssh bigred@192.168.61.141 -p 22100
The authenticity of host '[192.168.61.141]:22100 ([192.168.61.141]:22100)' can't be established.
RSA key fingerprint is SHA256:LgeZDi+EaEPZy3L4CMUpZ5VM0jq3vqN82ls6IC+WJwE.
Are you sure you want to continue connecting (yes/no/[fingerprint])? yes
Warning: Permanently added '[192.168.61.141]:22100' (RSA) to the list of known hosts.
bigred@192.168.61.141's password:
Welcome to Taroko Kadm Console v24.07 (task,kubectl,talosctl,mc,rclone,s3fs,JuiceFS Client)


bigred@kube-kadm:~$ dir wulin
total 36K
drwxr-sr-x  8 bigred  1000 4.0K Aug  9 19:58 .
drwxr-sr-x  1 bigred wheel 4.0K Aug 10 11:17 ..
drwxr-sr-x  2 bigred  1000 4.0K Jul 21 14:44 bin
drwxr-sr-x  4 bigred  1000 4.0K Aug  9 19:58 c30-dtbak
drwxr-sr-x  4 bigred  1000 4.0K Aug  9 19:58 c30-kadmbak
drwxr-sr-x 16 bigred  1000 4.0K Jul 19 00:06 images
drwxr-sr-x 15 bigred  1000 4.0K Jul 22 20:07 wk
drwxr-sr-x  2 bigred  1000 4.0K Jul 19 14:36 yaml
bigred@kube-kadm:~$ ls wulin/
bin  c30-dtbak  c30-kadmbak  images  wk  yaml
bigred@kube-kadm:~$ ls wulin/bin
clusterbind.yaml  clusterole.yaml  kau  kdu  kip  klu  rmpod  role.yaml  rolebind.yaml  sa.yaml  system.sh
bigred@kube-kadm:~$ vim wulin/bin/kau
-ash: vim: not found
bigred@kube-kadm:~$ vi wulin/bin/kau
bigred@kube-kadm:~$ dir wulin/wk
total 60K
drwxr-sr-x 15 bigred 1000 4.0K Jul 22 20:07 .
drwxr-sr-x  8 bigred 1000 4.0K Aug  9 19:58 ..
drwxr-sr-x  2 bigred 1000 4.0K Jul 21 14:02 containerd
drwxr-sr-x  2 bigred 1000 4.0K Jul 18 20:20 crio
drwxr-sr-x  2 bigred 1000 4.0K Jun 19 20:56 derby
drwxr-sr-x  5 bigred 1000 4.0K Aug  8 21:35 dt
drwxr-sr-x  2 bigred 1000 4.0K Jun 26 08:30 fbs
drwxr-sr-x  2 bigred 1000 4.0K Jul 17 15:43 ingress
drwxr-sr-x  2 bigred 1000 4.0K Jun 16 22:25 juicefs
drwxr-sr-x  2 bigred 1000 4.0K Jul  4 11:57 kadm
drwxr-sr-x  2 bigred 1000 4.0K Jun 17 22:29 mariadb
drwxr-sr-x  2 bigred 1000 4.0K Jun 25 08:02 minio
drwxr-sr-x  2 bigred 1000 4.0K Jun 17 22:41 mlb
drwxr-sr-x  2 bigred 1000 4.0K Jun 26 17:15 mysql
drwxr-sr-x  2 bigred 1000 4.0K Jun 16 21:33 redis
bigred@kube-kadm:~$ cat wulin/wk/kadm/kube-kadm-dkreg.yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: kadm
  namespace: kube-system
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: kadm
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: cluster-admin
subjects:
- kind: ServiceAccount
  name: kadm
  namespace: kube-system
---
apiVersion: v1
kind: Service
metadata:
  name: kadm
  namespace: kube-system
spec:
  selector:
    app: kadm
  ports:
    - protocol: TCP
      port: 22100
      targetPort: 22100
---
apiVersion: v1
kind: Service
metadata:
  name: dkreg
  namespace: kube-system
spec:
  selector:
    app: kadm
  ports:
    - protocol: TCP
      port: 5000
      targetPort: 5000
---
apiVersion: v1
kind: Pod
metadata:
  name: kube-kadm
  namespace: kube-system
  labels:
    app: kadm
spec:
  dnsConfig:
    nameservers:
    - 10.98.0.10
    searches:
    - kube-system.svc.k1.org
    - svc-dt.dt.svc.k1.org
    - svc.k1.org
  volumes:
  - name: dkreg-storage
    hostPath:
      path: /opt/dkreg
  - name: podman-storage
    hostPath:
      path: /opt/podman
  - name: wulin-storage
    hostPath:
      path: /opt/wulin
#  - name: dt-storage
#    hostPath:
#      path: /opt/dt/sys
#  - name: kadm-storage
#    persistentVolumeClaim:
#       claimName: pvc-kadm
  serviceAccountName: kadm
  containers:
  - name: kadm
    image: quay.io/cloudwalker/alp.kadm
    imagePullPolicy: Always
    tty: true
    ports:
    - containerPort: 22100
      hostPort: 22100
    lifecycle:
      postStart:
        exec:
          command:
            - /bin/bash
            - -c
            - |
              cp -p /var/tmp/htpasswd /opt/dkreg
    securityContext:
      privileged: true
    volumeMounts:
    - name: dkreg-storage
      mountPath: /opt/dkreg
    - name: podman-storage
      mountPath: /var/lib/containers/storage
    - name: wulin-storage
      mountPath: /home/bigred/wulin
#      mountPropagation: Bidirectional
    env:
    - name: KUBERNETES_SERVICE_HOST
      value: "kubernetes.default"
    - name: KUBERNETES_SERVICE_PORT_HTTPS
      value: "443"
    - name: KUBERNETES_SERVICE_PORT
      value: "443"
    securityContext:
      privileged: true
  - image: quay.io/cloudwalker/registry:2
    name: dkreg
    ports:
    - containerPort: 5000
      hostPort: 5000
    volumeMounts:
    - mountPath: "/var/lib/registry"
      name: dkreg-storage
    env:
    - name: REGISTRY_AUTH
      value: "htpasswd"
    - name: REGISTRY_AUTH_HTPASSWD_PATH
      value: "/var/lib/registry/htpasswd"
    - name: REGISTRY_AUTH_HTPASSWD_REALM
      value: "Registry Realm"
    - name: REGISTRY_STORAGE_DELETE_ENABLED
      value: "true"
  nodeSelector:
    app: taroko
bigred@kube-kadm:~$ vi wulin/wk/kadm/kube-kadm-dkreg.yaml
bigred@kube-kadm:~$ sudo podman build --tls-verify=false --format=docker
--force-rm --squash -t
dkreg.kube-system:5000/alp.sshd:24.01
~/wulin/images/sshd/--no-cache --force-rm --squash -t
dkreg.kube-system:5000/alp.sshd:24.01
~/wulin/images/sshd/Error: no context directory and no Containerfile specified
bigred@kube-kadm:~$ sudo podman build --tls-verify=false --format=docker --no-cache --force-rm --squash -t dkreg.kube-sy
stem:5000/alp.sshd:24.01 ~/wulin/images/sshd/
STEP 1/5: FROM quay.io/cloudwalker/alp.base
Trying to pull quay.io/cloudwalker/alp.base:latest...
Getting image source signatures
Copying blob 1695600ec709 done   |
Copying config 37c2fd1f36 done   |
Writing manifest to image destination
STEP 2/5: RUN apk update && apk upgrade &&     apk add --no-cache openssh-server tzdata &&     cp /usr/share/zoneinfo/Asia/Taipei /etc/localtime &&     ssh-keygen -t rsa -P "" -f /etc/ssh/ssh_host_rsa_key &&     echo "Port 22101" >> /etc/ssh/sshd_config &&     echo 'PubkeyAuthentication yes' >> /etc/ssh/sshd_config &&     echo 'PermitUserEnvironment yes' >> /etc/ssh/sshd_config &&     echo 'StrictHostKeyChecking no' >> /etc/ssh/ssh_config &&     echo -e 'Welcome to GitOps 6000\n' > /etc/motd &&     adduser -s /bin/ash -h /home/bigred -G wheel -D bigred &&     echo "%wheel ALL=(ALL) NOPASSWD: ALL" >> /etc/sudoers &&     echo -e 'bigred\nbigred\n' | passwd bigred &>/dev/null &&     mkdir -p /home/bigred/wulin && chown bigred:wheel /home/bigred/wulin &&     echo "export ENV=/usr/bin/ash.rc" >> /etc/profile &&     rm /sbin/reboot && rm /usr/bin/killall
fetch https://dl-cdn.alpinelinux.org/alpine/v3.20/main/x86_64/APKINDEX.tar.gz
fetch https://dl-cdn.alpinelinux.org/alpine/v3.20/community/x86_64/APKINDEX.tar.gz
v3.20.2-73-g96fd11cea2a [https://dl-cdn.alpinelinux.org/alpine/v3.20/main]
v3.20.2-79-gbdcfa560d54 [https://dl-cdn.alpinelinux.org/alpine/v3.20/community]
OK: 24157 distinct packages available
(1/1) Upgrading xz-libs (5.6.1-r3 -> 5.6.2-r0)
OK: 554 MiB in 161 packages
fetch https://dl-cdn.alpinelinux.org/alpine/v3.20/main/x86_64/APKINDEX.tar.gz
fetch https://dl-cdn.alpinelinux.org/alpine/v3.20/community/x86_64/APKINDEX.tar.gz
(1/3) Installing openssh-server-common (9.7_p1-r4)
(2/3) Installing openssh-server (9.7_p1-r4)
(3/3) Installing tzdata (2024a-r1)
Executing busybox-1.36.1-r29.trigger
OK: 556 MiB in 164 packages
Generating public/private rsa key pair.
Your identification has been saved in /etc/ssh/ssh_host_rsa_key
Your public key has been saved in /etc/ssh/ssh_host_rsa_key.pub
The key fingerprint is:
SHA256:fBf4FPUhnUfMy+qbCGiDXvmy5pJI3mPvwcAMvmuDdgw root@kube-kadm
The key's randomart image is:
+---[RSA 3072]----+
|            .oo=o|
|           . ..+=|
|   .      . o . +|
|  . +  .   o . o |
|   . +  S . o .  |
| E .. + o. . .   |
|  *.o..O .  .    |
| o Bo*oo+ . ...  |
|. o.o.B=o. . o.  |
+----[SHA256]-----+
STEP 3/5: EXPOSE 22101
STEP 4/5: ENTRYPOINT ["/usr/sbin/sshd"]
STEP 5/5: CMD ["-D"]
COMMIT dkreg.kube-system:5000/alp.sshd:24.01
--> 7ff1e3a034ff
Successfully tagged dkreg.kube-system:5000/alp.sshd:24.01
7ff1e3a034ffdd744267bd44098512bc9cb9073d1401ec1c5d26d24bda0c60f4
bigred@kube-kadm:~$ sudo podman push --tls-verify=false --creds=bigred:bigred dkreg.kube-system:5000/alp.sshd:24.01
Getting image source signatures
Copying blob 4893496f040e done   |
Copying blob 3beb57c515db done   |
Copying config 7ff1e3a034 done   |
Writing manifest to image destination
bigred@kube-kadm:~$ dkimg
"alp.sshd"
bigred@kube-kadm:~$ kubectl run g1 --image=dkreg.kube-system:5000/alp.sshd:24.01 --image-pull-policy=Always --port=22101
 -l="app=g1"
pod/g1 created
bigred@kube-kadm:~$ kubectl get pods
NAME   READY   STATUS    RESTARTS   AGE
g1     1/1     Running   0          20s
bigred@kube-kadm:~$ kubectl get pods -o wide
NAME   READY   STATUS    RESTARTS   AGE     IP            NODE          NOMINATED NODE   READINESS GATES
g1     1/1     Running   0          3m58s   10.244.1.17   c30-worker2   <none>           <none>
bigred@kube-kadm:~$ ssh bigred@g1.default -p 22101
bigred@kube-kadm:~$ ssh bigred@10.244.1.17 -p 22101
bigred@10.244.1.17's password:
Welcome to GitOps 6000

bigred@g1:~$ exit
bigred@kube-kadm:~$ kubectl get svc
NAME         TYPE        CLUSTER-IP   EXTERNAL-IP   PORT(S)   AGE
kubernetes   ClusterIP   10.98.0.1    <none>        443/TCP   26h
bigred@kube-kadm:~$ kubectl create service clusterip g1 --tcp=22101:22101
service/g1 created
bigred@kube-kadm:~$ kubectl get svc
NAME         TYPE        CLUSTER-IP    EXTERNAL-IP   PORT(S)     AGE
g1           ClusterIP   10.98.0.229   <none>        22101/TCP   3s
kubernetes   ClusterIP   10.98.0.1     <none>        443/TCP     26h
bigred@kube-kadm:~$ ssh bigred@g1.default -p 22101
bigred@g1.default's password:
Welcome to GitOps 6000

bigred@g1:~$ exit

```
[]bigred@M109:~$ docker exec -ti c30-worker2 bash
echo "10.98.0.229 g1.default.svc.k1.org" >> /etc/hosts
```



bigred@kube-kadm:~$ kubectl create service loadbalancer g1 --tcp=22101:22101
service/g1 created
bigred@kube-kadm:~$ kg all
NAME     READY   STATUS    RESTARTS   AGE
pod/g1   1/1     Running   0          81m

NAME                 TYPE           CLUSTER-IP    EXTERNAL-IP   PORT(S)           AGE
service/g1           LoadBalancer   10.98.0.114   10.89.0.101   22101:31884/TCP   6s
service/kubernetes   ClusterIP      10.98.0.1     <none>        443/TCP           27h


[]bigred@M109:~$ ssh 10.89.0.101 -p 22101
Warning: Permanently added '[10.89.0.101]:22101' (RSA) to the list of known hosts.
bigred@10.89.0.101's password:
Welcome to GitOps 6000

bigred@g1:~$ exit
Connection to 10.89.0.101 closed.
[]bigred@M109:~$ ssh 10.89.0.101 -p 31884
ssh: connect to host 10.89.0.101 port 31884: Connection refused


$ alias
nano='nano -Ynone'
ka='kubectl apply'
kd='kubectl delete'
dir='ls -alh'
docker='sudo /usr/bin/podman'
kg='kubectl get'
kk='kubectl krew'
pingdup='sudo arping -D -I eth0 -c 2 '
dkimg='curl -X GET -s -u bigred:bigred http://dkreg.kube-system:5000/v2/_catalog | jq ".repositories[]"'
kp='kubectl get pods -o wide -A | sed '"'"'s/(.*)//'"'"' | tr -s '"'"' '"'"' | cut -d '"'"' '"'"' -f 1-4,7,8 | column -t'
ks='kubectl get all -n kube-system'
kt='kubectl top'
ssh='ssh -q'
kgip='kubectl get pod --template '"'"'{{.status.podIP}}'"'"
ping='ping -c 4'



$ kubectl expose deployment front-end --port=80 --target-port=http --type=NodePort --name=front-end-svc
