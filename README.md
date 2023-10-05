# Kubernetes installation scripts

## Setup loadbalancer
```sh
stream {
    upstream kubernetes {
        server master1_ip:6443 max_fails=3 fail_timeout=30s;
        server master2_ip:6443 max_fails=3 fail_timeout=30s;
        server master3_ip:6443 max_fails=3 fail_timeout=30s;
    }
server {
        listen 6443;
        listen 443;
        proxy_pass kubernetes;
    }
}

```
## Install containerd
```sh
cat <<EOF | sudo tee /etc/modules-load.d/containerd.conf
overlay
br_netfilter
EOF
sudo modprobe overlay
sudo modprobe br_netfilter
cat <<EOF | sudo tee /etc/sysctl.d/99-kubernetes-cri.conf
net.bridge.bridge-nf-call-iptables  = 1
net.ipv4.ip_forward                 = 1
net.bridge.bridge-nf-call-ip6tables = 1
EOF
sudo sysctl --system
sudo apt install containerd -y
mkdir /etc/containerd
containerd config default > /etc/containerd/config.toml
sed -i 's/SystemdCgroup = false/SystemdCgroup = true/g' /etc/containerd/config.toml
systemctl restart containerd
```

## Install kubeadm, kubelet, kubectl
```sh
sudo apt-get update
sudo apt-get install -y apt-transport-https ca-certificates curl
sudo curl -fsSLo /usr/share/keyrings/kubernetes-archive-keyring.gpg https://packages.cloud.google.com/apt/doc/apt-key.gpg
echo "deb [signed-by=/usr/share/keyrings/kubernetes-archive-keyring.gpg] https://apt.kubernetes.io/ kubernetes-xenial main" | sudo tee /etc/apt/sources.list.d/kubernetes.list
sudo apt-get update
sudo apt-get install -y kubelet kubeadm kubectl
sudo apt-mark hold kubelet kubeadm kubectl
```

## Init cluster
```sh
kubeadm init --control-plane-endpoint=lb_ip:6443 --upload-certs --pod-network-cidr=10.0.0.0/8
```

## Install cni
```sh
helm repo add cilium https://helm.cilium.io/
helm install cilium cilium/cilium --version 1.11.6 --namespace kube-system
```

## Add master node
```sh
kubeadm join lb_ip:6443 --token oylvmu.12pwimke0blaaeji --discovery-token-ca-cert-hash sha256:303a791ef0bdaeb3a3b54ca80f8f4831dff6d0bb1c43c664d9102c9ec569ef61 --control-plane --certificate-key 3b4da12cd25d1c1e7a47abcb908c73405c4abd5e542f99692d8f1b9d368d307a
```

## Add worker node
```sh
kubeadm join lb_ip:6443 --token 9vr73a.a8uxyaju799qwdjv --discovery-token-ca-cert-hash sha256:7c2e69131a36ae2a042a339b33381c6d0d43887e2de83720eff5359e26aec866
```

## Get token list
```sh
kubeadm token list
```

## Create token
```sh
kubeadm token create
```

## Discovery token ca cert hash
```sh
openssl x509 -pubkey -in /etc/kubernetes/pki/ca.crt | openssl rsa -pubin -outform der 2>/dev/null | \
openssl dgst -sha256 -hex | sed 's/^.* //'
```

## Get Certificate key
```
kubeadm init phase upload-certs --upload-certs
```
