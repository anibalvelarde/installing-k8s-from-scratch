# Setting K8s from scratch on Linux OS (Ubuntu 22.4.x):
These notes assume you have installed Ubuntu 22.4.x.
Additionally, in my specific case, I am using `ProxMox` as a type 1 Hypervisor. I'm using an old gaming PC as my virtualization environment.  If this is a route you may be interested in, check the [YouTube video](#video-virtual-machines-pt-2-proxmox-install-w-kali-linux) that helped me with this setup.

## Step 1:  REQUIRED SOFTWARE ON ALL THE VMs

### Perform these steps after installing your OS
#### Update and upgrade packages on all your servers
```shell
sudo apt update
sudo apt upgrade -y
sudo apt install -y apt-transport-https ca-certificates curl software-properties-common
```

 #### Install Docker on all nodes
```shell
sudo apt-get remove docker docker-engine docker.io containerd runc
sudo apt-get update
sudo apt-get install -y docker.io
sudo systemctl start docker
sudo systemctl enable docker
```
 - After installing docker, `conteinerd` service should have been installed as a dependency. You may need to come back to the configuration of that service. It should be a file located at `/etc/containerd/config.toml`.
 - See [blog-post](#blog-post-registryk8siopause38-of-the-container-runtime-is-inconsistent-with-that-used-by-kubeadm) entry in troubleshooting section below.

#### Install Kubernetes components on all nodes
 
 You will [need to install](https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/install-kubeadm/#installing-kubeadm-kubelet-and-kubectl):
 - kubeadm
 - kubelet
 - kubectl
 - kubernetes-cni  <- - - This should install automagically as a dependency

 1. Update the `apt` package and install packages needed to user Kubernetes `apt` repository as a source
 ```shell
# update apt
sudo apt-get update
# apt-transport-https may be a dummy package; if so, you can skip that package
sudo apt-get install -y apt-transport-https ca-certificates curl gpg
```
2. Dowonload public signing key
```shell
# Download the K8S package repository signing key
curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.28/deb/Release.key | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg
```
3. Add the appropriate K8S repo (remember I'm using v1.28.4)
```shell
# This overwrites any existing configuration in /etc/apt/sources.list.d/kubernetes.list
echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.28/deb/ /' | sudo tee /etc/apt/sources.list.d/kubernetes.list
```
4. Update the `apt` package index, install apps and pin their versions
```shell
sudo apt-get update
sudo apt-get install -y kubelet kubeadm kubectl
sudo apt-mark hold kubelet kubeadm kubectl
```


## Step 2:  INITIALIZE Kubernetes - Master Node Only!
Perform these steps only on the server that will play the role of `Master Node`. More info can be found with K8S Dox on Cluster Networking, [here](https://kubernetes.io/docs/concepts/cluster-administration/networking/).

> **BEFORE YOU CONTINUE:**    It is probably a GREAT idea to take a snapshot of your master node at this point .

### Initialize the Cluster
Make sure you pick a CIDR range that will not conflict or overlap with your local network.
```shell
sudo kubeadm init
```

If all goes well, you should see output that looks like this:
```shell
Your Kubernetes control-plane has initialized successfully!

To start using your cluster, you need to run the following as a regular user:

  mkdir -p $HOME/.kube
  sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
  sudo chown $(id -u):$(id -g) $HOME/.kube/config

Alternatively, if you are the root user, you can run:

  export KUBECONFIG=/etc/kubernetes/admin.conf

You should now deploy a pod network to the cluster.
Run "kubectl apply -f [podnetwork].yaml" with one of the options listed at:
  https://kubernetes.io/docs/concepts/cluster-administration/addons/

Then you can join any number of worker nodes by running the following on each as root:

kubeadm join <your-k8s-master-node-IP-address>:6443 --token <some-token> \
        --discovery-token-ca-cert-hash sha256:<some-sha256-hash>
```

## Step 3: Join the Worker Node to the K8S Cluster (for each Worker Node)
The last section of the final [output from Step 2](#initialize-the-cluster) is what you will use in this step. Feel free to copy the section that reads:
```shell
kubeadm join <your-k8s-master-node-IP-address>:6443 --token <some-token> \
        --discovery-token-ca-cert-hash sha256:<some-sha256-hash>
```
Then go to each of your worker nodes and paste that command there.  Do not forget to run with elevatd privileges (e.g., `sudo`) to properly execute this command on the worker node.

You should see some output resembling this:
```shell
[preflight] Running pre-flight checks
[preflight] Reading configuration from the cluster...
[preflight] FYI: You can look at this config file with 'kubectl -n kube-system get cm kubeadm-config -o yaml'
[kubelet-start] Writing kubelet configuration to file "/var/lib/kubelet/config.yaml"
[kubelet-start] Writing kubelet environment file with flags to file "/var/lib/kubelet/kubeadm-flags.env"
[kubelet-start] Starting the kubelet
[kubelet-start] Waiting for the kubelet to perform the TLS Bootstrap...

This node has joined the cluster:
* Certificate signing request was sent to apiserver and a response was received.
* The Kubelet was informed of the new secure connection details.

Run 'kubectl get nodes' on the control-plane to see this node join the cluster.
```

## Step 4: Tweak the Control Plane (a.k.a. Master Node)
Go back to your master node and make some final adjustments. But first, let's see why these are needed.

If, on the master-node, you run the command `kubectl get nodes`, you will get an output that looks like this:
```shell
user@k8smaster:~$ kubectl get nodes

# Output:
The connection to the server localhost:8080 was refused - did you specify the right host or port?
```
That is because the `kubectl` command does not know the configuration it should use. That can be fix by following the instructions that were displayed on the output received after a successful `kubeadm init`:
```shell
# You can perform these commands on your Master K8S Node...
To start using your cluster, you need to run the following as a regular user:

  mkdir -p $HOME/.kube
  sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
  sudo chown $(id -u):$(id -g) $HOME/.kube/config
```
After setting up the current user's `$HOME/.kube` directory, wher you run the `kubectl` command the connection is not refused any more:
```shell
user@k8smaster:~$ kubectl get nodes

# utput:
NAME          STATUS     ROLES           AGE     VERSION
k8s-master    NotReady   control-plane   112m    v1.28.4
k8s-worker1   NotReady   <none>          5m42s   v1.28.4
k8s-worker2   NotReady   <none>          5m5s    v1.28.4
k8s-worker3   NotReady   <none>          4m57s   v1.28.4
```
Ahh... that looks better, but wait...  Why are they showing with `STATUS` of "_not ready_"?
The reason for that is that we have yet to deply container networking configurations. Is the last step to take. For my home lab, I chose to deply `calico`. To do that we have to
1. Download the `YAML` manifest for `calico` (from the Internet)
2. That's it!

```bash
# Command to download the YAML manifest:
kubectl apply -f https://docs.projectcalico.org/manifests/calico.yaml
```
Now, after a few moments and letting K8S catch-up, you can re-issue the `kubectl get nodes` command and you should see this:
```shell
user@k8smaster:~$ kubectl get nodes

# Output
NAME          STATUS   ROLES           AGE    VERSION
k8s-master    Ready    control-plane   146m   v1.28.4
k8s-worker1   Ready    <none>          39m    v1.28.4
k8s-worker2   Ready    <none>          38m    v1.28.4
k8s-worker3   Ready    <none>          38m    v1.28.4
```


## Troubleshooting Notes - In General
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

If that is the case, maybe the troubleshooting stories listed below may help you out.

## Troubleshooting Notes - `cgroup` configuration issues

you may need to research how to adjust `cgroup` configuration for your Linux distribution.  In my case, I had to edit the "grub" file (located at: `/etc/default/grub`) and make sure that the line with `GRUB_CMDLINE_LINUX` was set to:
```script
...
   GRUB_CMDLINE_LINUX="... systemd.unified_cgroup_hierarchy=1"
...
```
After updating that file and saving it, you'll have to regenerate the GRUB configuration, by running this command:
```script
sudo update-grub
```
Reboot your system (`sudo reboot`).

After loging back in, try resetting K8S:
```
sudo kubeadm reset
```
and re-try initializing it:
```
sudo kubeadm init
```


## Troubleshooting Notes - "no" `swap` memory is allowed!
After this changes, when I tried issuing the `kubeadm init` command, my `kubelet` service was still failing to start.  I then saw this error:

> "... failed to run Kubelet: running with swap "on" is not supported, please disable swap!"

so, I did... it was a load off my mind.  Here are the commands I used to accomplish that:
- first verify that `swap` is active by running this command:
```script
  free -h
```

You will see an output that looks like this:
```
               total        used        free      shared  buff/cache   available
Mem:           3.8Gi       501Mi       2.6Gi       1.0Mi       735Mi       3.1Gi
Swap:          3.8Gi         0Mi       3.8Gi
```
In that example, the `swap` is `on` because we see a total of 3.8 GB of RAM available for swap.

Next run this command to turn `off` swap:
```
sudo swapoff -a
```

Next, make sure that `swap` is removed from the boot sequence by commenting out the line that is setting the swap memory amounts. Run this command to open the file for editing:
```
    sudo nano /etc/fstab
```

Find the line that looks (kind of) like this:
```
UUID=<swap_UUID>  swap  swap  defaults  0  0
```
and comment it out with a leading `#` symbol:
```
# UUID=<swap_UUID>  swap  swap  defaults  0  0
```
Then, save and exit the file.

Lastly, confirm that swap is turned off by running `free -h` again. It should now that zero bytes are used for swap:
```
               total        used        free      shared  buff/cache   available
Mem:           3.8Gi       501Mi       2.6Gi       1.0Mi       735Mi       3.1Gi
Swap:            0B          0B          0B
```

After this, try resetting K8S:
```
sudo kubeadm reset
```
and re-try initializing it:
```
sudo kubeadm init
```

# References & Resources
The following resources and references were great points of information and helped me along the way.

## Kubernetes Docs
- Documentation about installing [Container Runtimes](https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/install-kubeadm/#installing-runtime)
- Documentation about [installing Kubernetes Apps](https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/install-kubeadm/#installing-kubeadm-kubelet-and-kubectl): `kubelet`, `kubectl`, and `kubeadm`.
- Documentation about Kubernetes [Production Environments](https://kubernetes.io/docs/setup/production-environment/)

## YouTube
### Video: Buld a Kubernetes Home Lab from Scratch step-by-step!
by [VirtualizationHowto](https://www.youtube.com/@VirtualizationHowto) YouTube Channel
#### Description
Building a Kubernetes Home Lab environment from scratch might seem like a daunting task. However, it is actually pretty straightforward with the right tools in place. In this video, we are going to use Kubeadm to initialize a Kubernetes cluster from scratch in our home lab environment.
#### Link
Follow [this link](https://www.youtube.com/watch?v=_WW16Sp8-Jw), to watch the video.

### Video: Virtual Machines Pt. 2 (Proxmox install w/ Kali Linux)
by [NetworkChuck](https://www.youtube.com/@NetworkChuck) YouTube Channel
#### Description
This is Virtual Machines Pt. 2. This time, we're going to dive deeper into the world of virtualization and start messing with Type 1 Hypervisors like Proxmox, ESXi..etc. You can take any old laptop or PC and convert it into a Virtual Machine power house!! We'll go through a complete Proxmox install, and touch on why we didn't use vmware.
#### Link
Follow [this link](https://www.youtube.com/watch?v=_u8qTN3cCnQ), to watch the video.

## Internet Blogs and Posts
### Blog Post: "registry.k8s.io/pause:3.8" of the container runtime is inconsistent with that used by kubeadm
#### Author: Urgensherpa
#### Excerpt
Recently while deploying 3 node kubernetes cluster( v 1.28.2-1.1) I came across this issue in the pre-flight checks stage
```shell
kubeadm init
```
the "warning" that came up read:
```shell
W1001 10:56:31.090889 28295 checks.go:835] detected that the sandbox image "registry.k8s.io/pause:3.8" of the container runtime is inconsistent with that used by kubeadm. It is recommended that using "registry.k8s.io/pause:3.9" as the CRI sandbox image.
```
#### Link
Follow [this link](https://dev.to/sherpaurgen/detected-that-the-sandbox-image-registryk8siopause38-of-the-container-runtime-is-inconsistent-with-that-used-by-kubeadm-1glc), to read the full post.