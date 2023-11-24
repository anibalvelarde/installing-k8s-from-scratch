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

 - Install Keyring & Kubernetes components on all nodes
 ```shell
# Download and Install the GPG Key Ring...
wget -qO - https://packages.cloud.google.com/apt/doc/apt-key.gpg | sudo gpg --dearmor -o /usr/share/keyrings/kubernetes-archive-keyring.gpg

# Add Kubernetes APT Repository...
echo "deb [signed-by=/usr/share/keyrings/kubernetes-archive-keyring.gpg] https://apt.
kubernetes.io/ kubernetes-xenial main" | sudo tee /etc/apt/sources.list.d/kubernetes.list

# update APT to ensure your server recognizes the new repository...
sudo apt-get update

# install K8S components...
sudo apt-get install -y kubelet kubeadm kubectl

 ```

## Step 2:  INITIALIZE Kubernetes - Master Node Only!
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

After this try resetting K8S:
```
sudo kubeadm reset
```
and re-try initializing it:
```
sudo kubeadm init --pod-network-cidr=10.244.0.0/16
```

It appears that I was not still out of the woods.  It looks like IPv6 (tcp6) was listening on some ports that were required by the install:
- 10250, 10259, 10257
- 22
- 6443
- 2379, 2380

I was able to learn of those port usage patterns with this command:
```shell
    # check to see if you have net-tools installed - you'll get a blank result if it is not installed
    dpkg -l | grep net-tools

    # To install net-tools:
    sudo apt update
    sudo apt install net-tools    

    # get the list of ports in use <pipe-to> find the string "LISTEN" and return those entries
    sudo netstat -tuln | grep LISTEN
```

I know I wasn't planning on using IPv6 at all, so I went ahead and deactivated IPv6. To disable, you'll need to open for edit the `/etc/sysctl.conf` file:

```shell
  sudo nano /etc/sysctl.conf

```

And add the following lines to it:
```shell
# Disable IPv6
net.ipv6.conf.all.disable_ipv6 = 1
net.ipv6.conf.default.disable_ipv6 = 1
```
After that, you need to apply the changes:
```shell
  sudo sysctl -p
```
Verify that `IPv6` is disabled:
```shell
cat /proc/sys/net/ipv6/conf/all/disable_ipv6

# if the previous command returns:
#   "1", the IPv6 is disabled
#   "0", the IPV6 is still enabled
```