## Setting up K3S cluster on Raspberry Pis

#### Hardware
- 1 Raspberry Pi 4 8Gb
- 2 Raspberry Pi 4 4Gb
- 3 SD Cards
- 3 USB-C power cables for the Raspberry Pis

1. Install Ubuntu on all the Pis with username `ubuntu` and hostname to the role of the node .i.e. `k8s-master` or `k8s-worker-1`, etc
2. Give the Pis static IP addresses from your router if possible. It will make your life easier

#### Node Set Up
1. First, prepare the [`/etc/hosts`](hosts) file on the computer you're using. (Not any of the nodes)
2. Install ansible on the device and create an ansible hosts [file](ansible/hosts)
3. Copy the ssh key of this machine to each node `ssh-copy-id -i ~/.ssh/id_rsa.pub ubuntu@k8s-master`
4. To confirm that Ansible is working fine, and we can connect to all nodes, run `ansible kube -m ping`

#### Installing K3S
1.  On the master node, download and install the latest version of K3S. Disable servicelb because we shall use metallb instead
`curl -sfL https://get.k3s.io | sh -s - --write-kubeconfig-mode 644 --disable servicelb --token some_random_password --node-taint CriticalAddonsOnly=true:NoExecute --bind-address 192.168.0.200 --disable-cloud-controller --disable local-storage`
2. On the worker nodes, install K3s as well `ansible workers -b -m shell -a "curl -sfL https://get.k3s.io | K3S_URL=https://192.168.0.200:6443 K3S_TOKEN=some_random_password sh -"`
3. Label and set roles for all the nodes `kubectl label nodes k8s-master kubernetes.io/role=master` and `kubectl label nodes k8s-worker-1 node-type=worker`
4. Lastly, copy the kube config into the environment `ansible kube -b -m lineinfile -a "path='/etc/environment' line='KUBECONFIG=/etc/rancher/k3s/k3s.yaml'"`

#### Installing MetalLB
1. Follow instructions from https://metallb.universe.tf/installation/ to install
2. Create a secret key for the MetalLB pods to encrypt speaker communications: `kubectl create secret generic -n metallb-system memberlist --from-literal=secretkey="$(openssl rand -base64 128)"`
3. Create a [config.yaml](metallb/config.yaml) for MetalLB to specify what range of IPs to use and apply the config

#### Install openfaas
1. Install arkade
2. Run `arkade install openfaas`
3. Give the openfaas gateway its own IP address accessible from outside by creating a custom MetalLB service
4. Give the openfaas IP a nice DNS name `ansible kube -b -m lineinfile -a "path='/etc/hosts' line='192.168.0.230 openfaas openfaas.local'"`