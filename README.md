# Setting K8s from scratch on Linux OS (Ubuntu 22.4.x):
These notes assume you have installed Ubuntu 22.4.x.
Additionally, in my specific case, I am using `ProxMox` as a type 1 Hypervisor. I'm using an old gaming PC as my virtualization environment.  If this is a route you may be interested in, [this video](https://www.youtube.com/watch?v=_u8qTN3cCnQ) (from [Network Chuck YT Channel](https://www.youtube.com/@NetworkChuck)) has everything you need to know setup `ProxMox`.

## Step 1:  REQUIRED SOFTWARE ON ALL THE VMs

### Perform these steps after installing your OS
 - Update and upgrade packages on all your servers
```shell

sudo apt update
sudo apt upgrade -y
sudo apt install -y apt-transport-https ca-certificates curl software-properties-common

```

 - Install Docker on all nodes
```shell

sudo apt-get remove docker docker-engine docker.io containerd runc
sudo apt-get update
sudo apt-get install -y docker.io
sudo systemctl start docker
sudo systemctl enable docker


```

 - Install Kubernetes components on all nodes
 ```shell

curl -s https://packages.cloud.google.com/apt/doc/apt-key.gpg | sudo apt-key add -
sudo add-apt-repository "deb http://apt.kubernetes.io/ kubernetes-xenial main"
sudo apt-get update
sudo apt-get install -y kubelet kubeadm kubectl

 ```

## Step 2:  INITIALIZE the Kubernetes Master Node
Perform these steps only on the server that will play the role of `Master Node`. More info can be found with K8S Dox on Cluster Networking, [here](https://kubernetes.io/docs/concepts/cluster-administration/networking/).

- Initialize the Cluster. Make sure you pick a CIDR range that will not conflict or overlap with your local network.
```shell

sudo kubeadm init --pod-network-cidr=10.244.0.0/16

```

### Troubleshooting Notes:
It is possible that in this step you get a problem stating that:
> Unfortunately, an error has occurred:
> - timed out waiting for the condition
>
> This error is likely caused by:
> - The kubelet is not running
> - The kubelet is unhealthy due to a misconfiguration of the node in some way (required _cgroups_ disabled)
>
>If you are on a systemd-powered system, you can try to troubleshoot the error with the following commands:
> - 'systemctl status kubelet'
> - 'journalctl -xeu kubelet'
> 

If that is the case, you may need to research how to adjust `cgroup` configuration for your Linux distribution.  In my case, I had to edit the "grub" file (located at: `/etc/default/grub`) and make sure that the line with `GRUB_CMDLINE_LINUX` was set to:
```script
...
   GRUB_CMDLINE_LINUX="... systemd.unified_cgroup_hierarchy=1"
...
```
After updating that file and saving it, you'll have to regenerate the GRUB configuration, by running this command:
```script
sudo update-grub
```
Lastly, reboot your system (`sudo reboot`). Log back in and re try `kubeadm init` as instructed earlier.
 