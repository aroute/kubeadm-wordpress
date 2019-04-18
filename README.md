# kubeadm-wordpress

## Deploy Kubeadm (Kubernetes 1.14) on RHEL 7 Developer

https://developers.redhat.com/products/rhel/hello-world/#fndtn-windows
```console
su -
subscription-manager repos --list-enabled
yum -y update
reboot
subscription-manager repos --enable=rhel-server-rhscl-7-rpms
subscription-manager repos --enable=rhel-7-server-optional-rpms
```

## Install Docker
https://access.redhat.com/documentation/en-us/red_hat_enterprise_linux_atomic_host/7/html-single/getting_started_with_containers/index#getting_docker_in_rhel_7

```console
subscription-manager register --force
subscription-manager list --available
Pool ID:             <pool id>
subscription-manager attach --pool=<pool id>

subscription-manager repos --enable=rhel-7-server-rpms
subscription-manager repos --enable=rhel-7-server-extras-rpms
subscription-manager repos --enable=rhel-7-server-optional-rpms

yum install -y docker device-mapper-libs device-mapper-event-libs
systemctl start docker.service
systemctl enable docker.service
systemctl status docker.service

groupadd docker
usermod -aG docker $USER
```

## Kubeadm setup

### Firewall

```console
for i in 6443 2379:2380 10250 10251 10252 30000 - 32767
do
firewall-cmd --zone=public --add-port=${i}/tcp --permanent
done
firewall-cmd --reload
```

### Swap
```console
swapoff -a
```

### Kubernetes

```console
cat <<EOF > /etc/yum.repos.d/kubernetes.repo
[kubernetes]
name=Kubernetes
baseurl=https://packages.cloud.google.com/yum/repos/kubernetes-el7-x86_64
enabled=1
gpgcheck=1
repo_gpgcheck=1
gpgkey=https://packages.cloud.google.com/yum/doc/yum-key.gpg https://packages.cloud.google.com/yum/doc/rpm-package-key.gpg
exclude=kube*
EOF

setenforce 0
sed -i 's/^SELINUX=enforcing$/SELINUX=permissive/' /etc/selinux/config

yum install -y kubelet kubeadm kubectl --disableexcludes=kubernetes

systemctl enable --now kubelet

cat <<EOF >  /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
EOF
sysctl --system

lsmod | grep br_netfilter
```

### Calico Network

```console
sudo kubeadm init --pod-network-cidr=192.168.0.0/16
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
kubectl apply -f \
https://docs.projectcalico.org/v3.6/getting-started/kubernetes/installation/hosted/kubernetes-datastore/calico-networking/1.7/calico.yaml
watch kubectl get pods --all-namespaces
kubectl taint nodes --all node-role.kubernetes.io/master-
kubectl get nodes -o wide
```

### Join worker - run from the worker node.

```console
kubeadm join 10.0.0.1:6443 --token 7qvvbg.1v21303zq3052kk2 \
    --discovery-token-ca-cert-hash sha256:b114e7bade75eb4d1891ec5e54da3f9e2ff85be95ede9accb7c0b09d465f726f
```

### Wordpress with MySQL

https://kubernetes.io/docs/tutorials/stateful-application/mysql-wordpress-persistent-volume/

#### Storage

```console
sudo mkdir -p srv/data
sudo chmod -R 777 data
```

#### Kustomize deploy

```console
cat <<EOF >>./kustomization.yaml
  resources:
    - mysql-deployment.yaml
    - wordpress-deployment.yaml
EOF
```

```console
kubectl apply -k .

kubectl expose deployment wordpress --type="NodePort" --port=80
```

## Helm

```console
tar -zxvf helm-v2.12.3-linux-amd64.tar.gz
mkdir -p ~/bin
cp ~/Downloads/linux-amd64/helm ~/bin/helm
export PATH=$PATH:$HOME/bin
echo 'export PATH=$PATH:$HOME/bin' >> ~/.bashrc
helm init --history-max 200





