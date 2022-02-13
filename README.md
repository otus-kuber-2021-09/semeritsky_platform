### Подходы к развертыванию и обновлению production кластера
 1. установлен кластер Kubernetes 1.17.4 при помощи Kubeadm
 2. обновлен до 1.18.0
 3. развернут ластер с помощью Kubespray

## Разворачиваем кластер

```

sudo cat > /etc/sysctl.d/99-kubernetes-cri.conf <<EOF
net.bridge.bridge-nf-call-iptables = 1
net.ipv4.ip_forward = 1
net.bridge.bridge-nf-call-ip6tables = 1
EOF
sudo sysctl --system

# install docker:

sudo apt-get update && sudo apt-get install -y apt-transport-https ca-certificates curl software-properties-common gnupg2

sudo curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo apt-key add -

sudo add-apt-repository \
"deb [arch=amd64] https://download.docker.com/linux/ubuntu \
$(lsb_release -cs) \
stable"

sudo apt-get update && sudo apt-get install -y \
containerd.io=1.2.13-1 \
docker-ce=5:19.03.8~3-0~ubuntu-$(lsb_release -cs) \
docker-ce-cli=5:19.03.8~3-0~ubuntu-$(lsb_release -cs)

sudo cat > /etc/docker/daemon.json <<EOF
{
"exec-opts": ["native.cgroupdriver=systemd"],
"log-driver": "json-file",
"log-opts": {
"max-size": "100m"
},
"storage-driver": "overlay2"
}
EOF

sudo mkdir -p /etc/systemd/system/docker.service.d
# Restart docker.
systemctl daemon-reload
systemctl restart docker

# Установка kubeadm, kubelet and kubectl

apt-get update && apt-get install -y apt-transport-https curl

curl -s https://packages.cloud.google.com/apt/doc/apt-key.gpg | apt-key add -
cat <<EOF > /etc/apt/sources.list.d/kubernetes.list
deb https://apt.kubernetes.io/ kubernetes-xenial main
EOF
apt-get update
apt-get install -y kubelet=1.17.4-00 kubeadm=1.17.4-00 kubectl=1.17.4-00
```

## На мастер-ноде
Создаем кластер kubeadm init --pod-network-cidr=192.168.0.0/24

```
Your Kubernetes control-plane has initialized successfully!
To start using your cluster, you need to run the following as a regular user:
  mkdir -p $HOME/.kube
  sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
  sudo chown $(id -u):$(id -g) $HOME/.kube/config
You should now deploy a pod network to the cluster.
Run "kubectl apply -f [podnetwork].yaml" with one of the options listed at:
  https://kubernetes.io/docs/concepts/cluster-administration/addons/
Then you can join any number of worker nodes by running the following on each as root:
kubeadm join 10.156.0.2:6443 --token 8nw282.zhx2bo3xtrd51p44 \
    --discovery-token-ca-cert-hash sha256:e267cf3423f41b8bb88071a3b9972eb87e733357f4362c774aa365b411723431
```

Копируем конфиг kubectl
```
mkdir -p $HOME/.kube
cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
chown $(id -u):$(id -g) $HOME/.kube/config
```

Устанавливаем Calico kubectl apply -f https://docs.projectcalico.org/manifests/calico.yaml Проверяем

```
root@master:~# kubectl get nodes
NAME     STATUS   ROLES    AGE   VERSION
master   Ready    master   13m   v1.17.4
```

добавим ноды, на каждом worker выполним kubeadm join 10.156.0.2:6443 --token 8nw282.zhx2bo3xtrd51p44 --discovery-token-ca-cert-hash sha256:e267cf3423f41b8bb88071a3b9972eb87e733357f4362c774aa365b411723431


```
root@master:~# kubectl get nodes
NAME       STATUS   ROLES    AGE   VERSION
master     Ready    master   49m   v1.17.4
worker-1   Ready    <none>   23m   v1.17.4
worker-2   Ready    <none>   14m   v1.17.4
worker-3   Ready    <none>   44s   v1.17.4
```

```
root@master:~# kubectl get po
NAME                               READY   STATUS    RESTARTS   AGE
nginx-deployment-c8fd555cc-5j9qf   1/1     Running   0          22s
nginx-deployment-c8fd555cc-84hzm   1/1     Running   0          22s
nginx-deployment-c8fd555cc-b6kvk   1/1     Running   0          22s
nginx-deployment-c8fd555cc-q9zkg   1/1     Running   0          22s
```

### Обновим кластер до 1.18.0

apt-get update && apt-get install -y kubeadm=1.18.0-00 kubelet=1.18.0-00 kubectl=1.18.0-00
```
root@master:~# kubectl get nodes
NAME       STATUS   ROLES    AGE   VERSION
master     Ready    master   66m   v1.18.0
worker-1   Ready    <none>   40m   v1.17.4
worker-2   Ready    <none>   31m   v1.17.4
worker-3   Ready    <none>   18m   v1.17.4
root@master:~# kubelet --version
Kubernetes v1.18.0
```

обновляем workers

```
root@master:~# kubeadm upgrade plan
[upgrade/config] Making sure the configuration is correct:
[upgrade/config] Reading configuration from the cluster...
[upgrade/config] FYI: You can look at this config file with 'kubectl -n kube-system get cm kubeadm-config -oyaml'
[preflight] Running pre-flight checks.
[upgrade] Running cluster health checks
[upgrade] Fetching available versions to upgrade to
[upgrade/versions] Cluster version: v1.17.13
[upgrade/versions] kubeadm version: v1.18.0
I1025 12:41:52.834255    8709 version.go:252] remote version is much newer: v1.19.3; falling back to: stable-1.18
[upgrade/versions] Latest stable version: v1.18.10
[upgrade/versions] Latest stable version: v1.18.10
[upgrade/versions] Latest version in the v1.17 series: v1.17.13
[upgrade/versions] Latest version in the v1.17 series: v1.17.13
Components that must be upgraded manually after you have upgraded the control plane with 'kubeadm upgrade apply':
COMPONENT   CURRENT       AVAILABLE
Kubelet     3 x v1.17.4   v1.18.10
            1 x v1.18.0   v1.18.10
Upgrade to the latest stable version:
COMPONENT            CURRENT    AVAILABLE
API Server           v1.17.13   v1.18.10
Controller Manager   v1.17.13   v1.18.10
Scheduler            v1.17.13   v1.18.10
Kube Proxy           v1.17.13   v1.18.10
CoreDNS              1.6.5      1.6.7
Etcd                 3.4.3      3.4.3-0
You can now apply the upgrade by executing the following command:
        kubeadm upgrade apply v1.18.10
Note: Before you can perform this upgrade, you have to update kubeadm to v1.18.10.
_____________________________________________________________________
root@master:~# kubeadm upgrade apply v1.18.0
[upgrade/config] Making sure the configuration is correct:
[upgrade/config] Reading configuration from the cluster...
[upgrade/config] FYI: You can look at this config file with 'kubectl -n kube-system get cm kubeadm-config -oyaml'
[preflight] Running pre-flight checks.
:
:
[addons] Applied essential addon: CoreDNS
[addons] Applied essential addon: kube-proxy
[upgrade/successful] SUCCESS! Your cluster was upgraded to "v1.18.0". Enjoy!
[upgrade/kubelet] Now that your control plane is upgraded, please proceed with upgrading your kubelets if you haven't already done so.
```

убираем нагрузку с worker-1 kubectl drain worker-1 --ignore-daemonsets

```
root@master:~# kubectl drain worker-1 --ignore-daemonsets
node/worker-1 cordoned
WARNING: ignoring DaemonSet-managed Pods: kube-system/calico-node-czdhn, kube-system/kube-proxy-msddh
evicting pod default/nginx-deployment-c8fd555cc-84hzm
evicting pod default/nginx-deployment-c8fd555cc-q9zkg
pod/nginx-deployment-c8fd555cc-q9zkg evicted
pod/nginx-deployment-c8fd555cc-84hzm evicted
node/worker-1 evicted
```

```
root@master:~# kubectl get nodes -o wide
NAME       STATUS                     ROLES    AGE   VERSION   INTERNAL-IP   EXTERNAL-IP   OS-IMAGE             KERNEL-VERSION   CONTAINER-RUNTIME
master     Ready                      master   81m   v1.18.0   10.156.0.2    <none>        Ubuntu 18.04.5 LTS   5.4.0-1028-gcp   docker://19.3.8
worker-1   Ready,SchedulingDisabled   <none>   55m   v1.17.4   10.156.0.3    <none>        Ubuntu 18.04.5 LTS   5.4.0-1028-gcp   docker://19.3.8
worker-2   Ready                      <none>   46m   v1.17.4   10.156.0.4    <none>        Ubuntu 18.04.5 LTS   5.4.0-1028-gcp   docker://19.3.8
worker-3   Ready                      <none>   32m   v1.17.4   10.156.0.5    <none>        Ubuntu 18.04.5 LTS   5.4.0-1028-gcp   docker://19.3.8
```

```
apt-get install -y kubelet=1.18.0-00 kubeadm=1.18.0-00
systemctl restart kubelet

root@master:~# kubectl get nodes -o wide
NAME       STATUS                     ROLES    AGE   VERSION   INTERNAL-IP   EXTERNAL-IP   OS-IMAGE             KERNEL-VERSION   CONTAINER-RUNTIME
master     Ready                      master   84m   v1.18.0   10.156.0.2    <none>        Ubuntu 18.04.5 LTS   5.4.0-1028-gcp   docker://19.3.8
worker-1   Ready,SchedulingDisabled   <none>   58m   v1.18.0   10.156.0.3    <none>        Ubuntu 18.04.5 LTS   5.4.0-1028-gcp   docker://19.3.8
worker-2   Ready                      <none>   49m   v1.17.4   10.156.0.4    <none>        Ubuntu 18.04.5 LTS   5.4.0-1028-gcp   docker://19.3.8
worker-3   Ready                      <none>   36m   v1.17.4   10.156.0.5    <none>        Ubuntu 18.04.5 LTS   5.4.0-1028-gcp   docker://19.3.8
```

```
kubectl uncordon worker-1

root@master:~# kubectl get po -o wide
NAME                               READY   STATUS    RESTARTS   AGE     IP              NODE       NOMINATED NODE   READINESS GATES
nginx-deployment-c8fd555cc-5j9qf   1/1     Running   0          25m     192.168.0.129   worker-3   <none>           <none>
nginx-deployment-c8fd555cc-5tnt7   1/1     Running   0          8m53s   192.168.0.197   worker-2   <none>           <none>
nginx-deployment-c8fd555cc-b6kvk   1/1     Running   0          25m     192.168.0.193   worker-2   <none>           <none>
nginx-deployment-c8fd555cc-htr75   1/1     Running   0          8m53s   192.168.0.131   worker-3   <none>           <none>
```

```
root@master:~#
root@master:~# kubectl get nodes
NAME       STATUS   ROLES    AGE   VERSION
master     Ready    master   99m   v1.18.0
worker-1   Ready    <none>   73m   v1.18.0
worker-2   Ready    <none>   63m   v1.18.0
worker-3   Ready    <none>   50m   v1.18.0
root@master:~# kubectl get po -o wide
NAME                               READY   STATUS    RESTARTS   AGE     IP              NODE       NOMINATED NODE   READINESS GATES
nginx-deployment-c8fd555cc-4pfp8   1/1     Running   0          3m25s   192.168.0.70    worker-1   <none>           <none>
nginx-deployment-c8fd555cc-b6kvk   1/1     Running   0          37m     192.168.0.193   worker-2   <none>           <none>
nginx-deployment-c8fd555cc-l8dds   1/1     Running   0          69s     192.168.0.132   worker-3   <none>           <none>
nginx-deployment-c8fd555cc-zghzz   1/1     Running   0          53s     192.168.0.133   worker-3   <none>           <none>
```

