# openfass_kubernetes_cluster_from_RaspberryPI
**Software-Defined System Term Project:** Build the Kubernetes cluster using Raspberry PI with this project
From [Github Project](https://github.com/openfaas/faas-netes)

## Project structure
- Prometheus
- Queue Worker
- OpenFaas
- Nats
- Gateway

## Topology
Kubenetes cluster topology 

| Device                                   |             IP Address             |
| ---------------------------------------- | :--------------------------------: |
| Master 1 [VM]                            |           192.168.0.109            |
| Master 2 [VM]                            |           192.168.0.103            |
| Load Balancer for Master [VM] - HA Proxy |           192.168.0.104            |
| Worker Node [Raspberry PI]               | 192.168.0.105 - 192.168.0.108.     |

## Set up Kubernetes cluster

### Set up Worker Node [Raspberry PI]

Set up process can be followed in [this medium article](https://medium.com/nycdev/k8s-on-pi-9cc14843d43).

- Swap need to be disbled for kubeadm to run correctly.

```sh
sudo swapoff -a
sudo systemctl disable dphys-swapfile.service
```
### Preparing Master Node

We create multi-master using 2 VMs  The chosen linux distribution for master nodes is Mininet-Wifi From Our Class

#### Install Mininet-WIfi for all master node

1. Download from [here](https://klab.cp.eng.chula.ac.th/mininet-wifi.ova).
2. Install on VirtualBox with 2GB of RAM or higher.
3. Configure VM to use bridged adapter as shown in the picture below.

> Important:
> Ensure that each master VM has different hostname ***

#### Install and setup Docker On Master
```curl -sSL get.docker.com | sh && \
sudo usermod pi -aG docker && \
newgrp docker
```

#### Install and setup kubeadm and kubectl On Master
1. Edit the following file
```/etc/apt/sources.list.d/kubernetes.list```
2. Add the following file
```deb http://apt.kubernetes.io/ kubernetes-xenial main```
3. Add the key
```curl -s https://packages.cloud.google.com/apt/doc/apt-key.gpg | apt-key add -```

### Set up Load Balancer for master - HAProxy
  We choose the linux distribution for Loadbalancer is Mininet-Wifi 
  1.Install Haproxy
  ```sudo apt-get update
     sudo apt-get install haproxy
  ```
  2.Edit haproxy.cg
  ``` nano /etc/haproxy/haproxy.cfg ```
  
  3.Add this configuration parameters to HAProxy config :
  ```global
    user haproxy
    group haproxy
defaults
    mode http
    log global
    retries 2
    timeout connect 3000ms
    timeout server 5000ms
    timeout client 5000ms
frontend kubernetes
    bind (IP OF YOUR LOADBALANCE)
    option tcplog
    mode tcp
    default_backend kubernetes-master-nodes
backend kubernetes-master-nodes
    mode tcp
    balance roundrobin
    option tcp-check
    server k8s-master-0 (IP OF YOUR FIRST MASTER NODE) check fall 3 rise 2. 
    server k8s-master-1 (IP OF YOUR SECOND MASTER NODE) ccheck fall 3 rise 2
  ```
  4. Start HAproxy
   ```systemctl start haproxy```
  
### Initialize cluster

After load balancer for master nodes was set, the cluster with stacked etcd-control plane can be initialized. The process is based on 2 sources which is [this medium article](https://medium.com/nycdev/k8s-on-pi-9cc14843d43) and [official kubeadm HA cluster guide](https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/high-availability/).

> From now, we assume that master nodes' load balancer is at 192.168.0.104:6443

#### Initialize first master node

1. Init kubernetes cluster
```sh
sudo kubeadm init --control-plane-endpoint "192.168.0.104:6443" --upload-certs --token-ttl=0
```
2. Setup kubectl
```sh
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```
3. Install weavenet
```sh
kubectl apply -f "https://cloud.weave.works/k8s/net?k8s-version=$(kubectl version | base64 | tr -d '\n')"
```
> If you want to use kubectl on your other machine, use the same config in step 2 to set up kubectl on it.

#### Initialize second master node

Run the following command
```sh
sudo kubeadm join 192.168.0.194:6443 --token <your-token> --discovery-token-ca-cert-hash sha256:<your-discovery-token-ca-cert-hash> --control-plane --certificate-key <your-certificate-key>
```

> Your master nodes should be all set! Try getting nodes status with `kubectl get nodes`

#### Initialize each worker node

Run the following command
```sh
sudo kubeadm join 192.168.0.104:6443 --token <your-token> --discovery-token-ca-cert-hash sha256:<your-discovery-token-ca-cert-hash>
```

> From now, Your cluster should be now ready to deploy applications!

## Deploy the application to Kubernetes cluster
1. Clone Project From [Github Project](https://github.com/openfaas/faas-netes)
2. Goto Directory Of Project
3. Run Command
  ```kubectl apply -f namespace.yml
  kubectl apply -f yaml_armhf/
  ```
4.Check If Deploy Complete
  ``` kubectl get deploy -n openfaas
      kubectl get pods --all-namespaces
  ```



