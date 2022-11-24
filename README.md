# k8s
Deploying a basic cluster from Linux

# Kubernete cluster description:

Master  : 192.168.32.xx   k8smaster.example.net k8smaster <br>
Worker1 : 192.168.32.xx   k8sworker1.example.net k8sworker1 <br>
Worker2 : 192.168.32.xx   k8sworker2.example.net k8sworker2

# Kubernete cluster architecture:

![image](https://user-images.githubusercontent.com/86851766/203845104-440706e3-95b5-435d-917b-09ddf5e74eb2.png)

# Steps to cluster deployment:

# Step 1)  Set hostname and add entries in hosts file.

Login to master node and set hostname using hostnamectl command

```
$ sudo hostnamectl set-hostname "k8smaster.example.net"
$ exec bash
```

On the worker nodes, run(*) :

```
$ sudo hostnamectl set-hostname "k8sworker1.example.net"   // 1st worker node
$ sudo hostnamectl set-hostname "k8sworker2.example.net"   // 2nd worker node
$ exec bash
```
(*) - We just did one worker node deployment

Add the following entries in /etc/hosts file on each node (master and worker)

```
192.168.32.xx   k8smaster.example.net k8smaster
192.168.32.xx   k8sworker1.example.net k8sworker1
192.168.32.xx   k8sworker2.example.net k8sworker2
```

# Step 2) Disable swap & add Kernel settings

Execute beneath swapoff and sed command to disable swap. Make sure to run the following commands on all the nodes.

```
$ sudo swapoff -a
$ sudo sed -i '/ swap / s/^\(.*\)$/#\1/g' /etc/fstab
```

Load the following kernel modules on all the nodes,

```
$ sudo tee /etc/modules-load.d/containerd.conf <<EOF
overlay
br_netfilter
EOF
$ sudo modprobe overlay
$ sudo modprobe br_netfilter
```

Set the following Kernel parameters for Kubernetes, run beneath tee command

```
$ sudo tee /etc/sysctl.d/kubernetes.conf <<EOF
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
net.ipv4.ip_forward = 1
EOF 
```

Reload the above changes, run

```
$ sudo sysctl --system
```

# Step 3) Install containerd run time

In this guide, we are using containerd run time for our Kubernetes cluster. So, to install containerd, first install its dependencies.

$ sudo apt install -y curl gnupg2 software-properties-common apt-transport-https ca-certificates

```
$ sudo curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmour -o /etc/apt/trusted.gpg.d/docker.gpg
$ sudo add-apt-repository "deb [arch=amd64] https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable"
```

Now, run following apt command to install containerd

```
$ sudo apt update
$ sudo apt install -y containerd.io
```

Configure containerd so that it starts using systemd as cgroup.

```
$ containerd config default | sudo tee /etc/containerd/config.toml >/dev/null 2>&1
$ sudo sed -i 's/SystemdCgroup \= false/SystemdCgroup \= true/g' /etc/containerd/config.toml
```

Restart and enable containerd service

```
$ sudo systemctl restart containerd
$ sudo systemctl enable containerd
```

# Step 4) Add apt repository for Kubernetes

Execute following commands to add apt repository for Kubernetes

```
$ curl -s https://packages.cloud.google.com/apt/doc/apt-key.gpg | sudo apt-key add -
$ sudo apt-add-repository "deb http://apt.kubernetes.io/ kubernetes-xenial main"
```

```
Note: At time of writing this guide, Xenial is the latest Kubernetes repository but when repository is available for Ubuntu 22.04 (Jammy Jellyfish) then you need replace xenial word with ‘jammy’ in ‘apt-add-repository’ command.
```

# Step 5) Install Kubernetes components Kubectl, kubeadm & kubelet

Install Kubernetes components like kubectl, kubelet and Kubeadm utility on all the nodes. Run following set of commands,

```
$ sudo apt update
$ sudo apt install -y kubelet kubeadm kubectl
$ sudo apt-mark hold kubelet kubeadm kubectl
```

# Step 6) Initialize Kubernetes cluster with Kubeadm command

Now, we are all set to initialize Kubernetes cluster. Run the following Kubeadm command from the master node only.

```
$ sudo kubeadm init --control-plane-endpoint=k8smaster.example.net
```

Output of above command,

![image](https://user-images.githubusercontent.com/86851766/203847346-55bb2b9b-08c7-4977-bba4-449de3a8e54a.png)

As the output above confirms that control-plane has been initialize successfully. In output also we are getting set of commands for interacting the cluster and also the command for worker node to join the cluster.

So, to start interacting with cluster, run following commands from the master node,

```
$ mkdir -p $HOME/.kube
$ sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
$ sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

Now, try to run following kubectl commands to view cluster and node status

```
$ kubectl cluster-info
$ kubectl get nodes
```

Output,

![image](https://user-images.githubusercontent.com/86851766/203847715-5c7dd7cd-d044-4208-b321-2c132aca6c55.png)

Join both the worker nodes to the cluster, command is already there is output, just copy paste on the worker nodes,

![image](https://user-images.githubusercontent.com/86851766/203847868-00c783d9-0787-46ba-bead-f496c0993a76.png)
```
Note: Do the same with other node
```

Check the nodes status from master node using kubectl command,

```
kubectl get nodes
```

![image](https://user-images.githubusercontent.com/86851766/203848036-127c5a12-f0ae-4abc-86de-894d0b55802b.png)

As we can see nodes status is ‘NotReady’, so to make it active. We must install CNI (Container Network Interface) or network add-on plugins like Calico, Flannel and Weave-net.

# Step 6) Install Calico Pod Network Add-on

Run following curl and kubectl command to install Calico network plugin from the master node,

```
$ curl https://projectcalico.docs.tigera.io/manifests/calico.yaml -O
$ kubectl apply -f calico.yaml
```

Output of above commands would look like below,

![image](https://user-images.githubusercontent.com/86851766/203848252-f541995e-9b57-45dd-b069-0ffd823e2466.png)

Verify the status of pods in kube-system namespace,

Output,

![image](https://user-images.githubusercontent.com/86851766/203848321-621a7af4-96dc-4ec7-a4b5-7d85589722ff.png)

Perfect, check the nodes status as well.

```
$ kubectl get nodes
```

![image](https://user-images.githubusercontent.com/86851766/203848779-4cadbc7f-423f-477a-ac00-6153b0024715.png)

Step 7) Test Kubernetes Installation

```
$ kubectl create deployment nginx-app --image=nginx --replicas=2
```

Check the status of nginx-app deployment

```
$ kubectl get deployment nginx-app
NAME        READY   UP-TO-DATE   AVAILABLE   AGE
nginx-app   2/2     2            2           68s
```

Expose the deployment as NodePort,

```
$ kubectl expose deployment nginx-app --type=NodePort --port=80
service/nginx-app exposed
```

Run following commands to view service status

```
$ kubectl get svc nginx-app
$ kubectl describe svc nginx-app
```

Output of above commands,

![image](https://user-images.githubusercontent.com/86851766/203849036-6ed192ed-7daa-45a5-b67a-b3cfa6bb1f8a.png)

Use following command to access nginx based application,

```
$ curl http://192.168.1.174:31246
```

![image](https://user-images.githubusercontent.com/86851766/203849113-0b05061e-87b7-42ea-9421-8f8753dead15.png)



```
Fonte: https://www.linuxtechi.com/install-kubernetes-on-ubuntu-22-04/
```
