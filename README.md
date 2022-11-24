# k8s - Deploying a basic cluster from Linux

We did this from Ubuntu 22.04 distro

# Kubernete cluster description:

Master  : 192.168.32.xx   k8smaster.example.net k8smaster <br>
Worker1 : 192.168.32.xx   k8sworker1.example.net k8sworker1 <br>
Worker2 : 192.168.32.xx   k8sworker2.example.net k8sworker2

# Kubernete cluster architecture:

![image](https://user-images.githubusercontent.com/86851766/203845104-440706e3-95b5-435d-917b-09ddf5e74eb2.png)

# Steps to cluster deployment:

Step 1)  Set hostname and add entries in hosts file.

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

Step 2) Disable swap & add Kernel settings

Execute beneath swapoff and sed command to disable swap. Make sure to run the following commands on all the nodes.

```
$ sudo swapoff -a
$ sudo sed -i '/ swap / s/^\(.*\)$/#\1/g' /etc/fstab
```
