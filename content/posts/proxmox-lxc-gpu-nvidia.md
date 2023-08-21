---
title: "Run NVIDIA GPU Inside an LXC Container on Proxmox"
description: "Discover how to harness the power of NVIDIA GPU acceleration within a Linux Container (LXC) on Proxmox. This comprehensive guide provides step-by-step instructions for installing the NVIDIA driver, configuring the LXC container, and verifying successful setup."
date: 2023-08-21
tags: [proxmox, gpu, lxc]
---

I recently made the decision to repurpose an old notebook into a server.
To accomplish this, I installed Proxmox on it.
Considering that I need to utilize GPU acceleration for various applications, I opted to perform GPU passthrough within a virtual machine (VM).

However, due to the fact that this notebook employs NVIDIA Optimus technology to minimize battery consumption, there are potential complications associated with GPU passthrough.
Despite attempting a range of solutions, I wasn't able to achieve a satisfactory resolution to the issue.

Consequently, in my situation, the sole viable option is to employ a Linux Container (LXC) with GPU passthrough.
This approach offers me the flexibility I need, similar to that of a VM.
I've already conducted experiments with running games and applications that depend on CUDA core acceleration and GPU functionality even within a Docker container.

If you're interested, another guide will be made available that details the configuration of Docker within an LXC container with NVIDIA GPU passthrough. 
Stay tuned for the upcoming second part of the guide, which will delve into the specifics of this process.

## Specifics
I am using an old ASUS V550X laptop with the following specifications:

* CPU: Intel core i7-6700H
* GPU: NVIDIA GeForce GTX 950M
* Proxmox: pve-manager/8.0.4/d258a813cfa6b390 (running kernel: 6.2.16-6-pve)

With these specifications in mind, we can now proceed to the steps required to configure an LXC container with an NVIDIA GPU.

## Installing NVIDIA Driver on Proxmox Host

To utilize your NVIDIA graphics card within your container on Proxmox, it's necessary to install the NVIDIA driver on both the Proxmox host and the LXC container. Let's begin by following these steps to install the NVIDIA driver on the Proxmox host.

This guide is designed for the `535.86.05` version of the driver, but you can readily adapt it to any driver version you have.

### Blacklist Nouveau Driver

Before we delve into the installation, we must disable the Nouveau kernel module, which can interfere with NVIDIA driver functionality. 
Execute the following commands in your terminal:

```shell
echo -e "blacklist nouveau\noptions nouveau modeset=0" > /etc/modprobe.d/blacklist-nouveau.conf
update-initramfs -u
```

After this, ensure a smooth transition by rebooting your Proxmox server. 
A restart via the Proxmox UI works just as well.

### Install Proxmox VE Headers + Essential Tools

Next, we need to equip your system with the Proxmox VE headers that correspond to your current kernel version, as well as a set of essential development tools and libraries required for compiling the NVIDIA driver.
Open your terminal and execute:

```shell
apt install pve-headers-$(uname -r)
apt install build-essential
```

### Download + Install NVIDIA Driver

Now comes the pivotal step: obtaining and installing the NVIDIA driver. 
Begin by downloading the latest version and making it executable:

```shell
wget https://us.download.nvidia.com/XFree86/Linux-x86_64/535.86.05/NVIDIA-Linux-x86_64-535.86.05.run
chmod +x NVIDIA-Linux-x86_64-535.86.05.run
```

Always remember to customize commands for your driver version if necessary.

> Note: If desired, you can verify the integrity of the run file by utilizing the `--check` option:
> ```shell
> ./NVIDIA-Linux-x86_64-535.86.05.run --check
> ```

Finally, install the driver with the `--dkms` option to ensure kernel updates don't disrupt functionality:

```shell
./NVIDIA-Linux-x86_64-535.86.05.run --dkms
```

### Configure Kernel Module Loading

To enable this functionality, configure the kernel to load the essential NVIDIA modules during boot:

```shell
echo -e '\n# load nvidia modules\nnvidia-drm\nnvidia-uvm' >> /etc/modules-load.d/modules.conf
```

Add the following content into the file located at `/etc/udev/rules.d/70-nvidia.rules`.
This action guarantees the creation of essential device files in the `/dev/` directory upon system boot, enabling effective communication between LXC containers and the GPU. 
Here's the content to include:

```
KERNEL=="nvidia", RUN+="/bin/bash -c '/usr/bin/nvidia-smi -L && /bin/chmod 666 /dev/nvidia*'"
KERNEL=="nvidia_uvm", RUN+="/bin/bash -c '/usr/bin/nvidia-modprobe -c0 -u && /bin/chmod 0666 /dev/nvidia-uvm*'"
SUBSYSTEM=="module", ACTION=="add", DEVPATH=="/module/nvidia", RUN+="/usr/bin/nvidia-modprobe -m"
```

### Enable NVIDIA Persistent

As a final step, it is necessary to enable the daemon to maintain a persistent software state in the NVIDIA driver:

```shell
nvidia-persistenced
```

### Verify NVIDIA Driver Installation

Lastly, confirm the successful installation of the NVIDIA driver with a simple command:

```shell
nvidia-smi
```

After executing this command, the output should appear as follows:

```console
root@pve:~# nvidia-smi 
Fri Aug 21 21:07:58 2023       
+---------------------------------------------------------------------------------------+
| NVIDIA-SMI 535.86.05              Driver Version: 535.86.05    CUDA Version: 12.2     |
|-----------------------------------------+----------------------+----------------------+
| GPU  Name                 Persistence-M | Bus-Id        Disp.A | Volatile Uncorr. ECC |
| Fan  Temp   Perf          Pwr:Usage/Cap |         Memory-Usage | GPU-Util  Compute M. |
|                                         |                      |               MIG M. |
|=========================================+======================+======================|
|   0  NVIDIA GeForce GTX 950M        On  | 00000000:01:00.0 Off |                  N/A |
| N/A   42C    P8              N/A / 200W |      0MiB /  2048MiB |      0%      Default |
|                                         |                      |                  N/A |
+-----------------------------------------+----------------------+----------------------+
                                                                                         
+---------------------------------------------------------------------------------------+
| Processes:                                                                            |
|  GPU   GI   CI        PID   Type   Process name                            GPU Memory |
|        ID   ID                                                             Usage      |
|=======================================================================================|
|  No running processes found                                                           |
+---------------------------------------------------------------------------------------+
```

If your output looks like the previous example, it indicates that everything has worked correctly up to this point.
Now we can proceed with the LXC container configuration.

## Configuring LXC Container
In this segment, I have set up an LXC container using Ubuntu 22.04 with the LXC ID set to `100`.
The container is configured as `privileged` and has `nesting=1` and `fuse=1` enabled.

Although I opted for Ubuntu due to its user-friendliness, the following steps are not tied to a specific distribution and are generally applicable to various Debian-based distributions. 

> If your LXC ID varies from `100`, please make the necessary adjustments as you proceed with the upcoming instructions.


### Edit LXC Configuration

To proceed, shut down the LXC container and open the file located at `/etc/pve/lxc/<LXC-ID>.conf`. 
Then, append the provided content, ensuring you adjust the filename according to your specific LXC ID (e.g., `/etc/pve/lxc/100.conf`):

```
lxc.cgroup2.devices.allow: c 195:* rwm
lxc.cgroup2.devices.allow: c 509:* rwm
lxc.cgroup2.devices.allow: c 511:* rwm
lxc.mount.entry: /dev/nvidia0 dev/nvidia0 none bind,optional,create=file
lxc.mount.entry: /dev/nvidiactl dev/nvidiactl none bind,optional,create=file
lxc.mount.entry: /dev/nvidia-modeset dev/nvidia-modeset none bind,optional,create=file
lxc.mount.entry: /dev/nvidia-uvm dev/nvidia-uvm none bind,optional,create=file
lxc.mount.entry: /dev/nvidia-uvm-tools dev/nvidia-uvm-tools none bind,optional,create=file
```

> Note: This guide is also applicable for configuring a GPU for Docker within an LXC container. 
> Therefore, if you need to perform this setup, make sure to append the following lines to the same configuration file:
> ```
> lxc.apparmor.profile: unconfined
> lxc.cgroup.devices.allow: a
> lxc.cap.drop:
> ```

### Download + Install NVIDIA Driver Inside LXC

{{< callout emoji="⚠️" text="Important: Ensure that the NVIDIA driver version of the LXC container matches the host driver version. Consistency is crucial for proper functionality." >}}

Just as with the host system, the LXC container requires the NVIDIA driver. 
Download the identical version of the driver installed on the host (for instance, version `535.86.05`) and then make it executable:

```shell
wget https://us.download.nvidia.com/XFree86/Linux-x86_64/535.86.05/NVIDIA-Linux-x86_64-535.86.05.run
chmod +x NVIDIA-Linux-x86_64-535.86.05.run
```

> Note: Similar to the host, if you wish, you can ensure the integrity of the downloaded run file by using the `--check` option:
> ```shell
> ./NVIDIA-Linux-x86_64-535.86.05.run --check
> ```

The steps thus far are largely identical, with one notable distinction. 
In contrast to the host, the kernel module doesn't necessitate installation within the container, as it leverages the module from the host. 
To accomplish this, employ the `--no-kernel-module` option:

```shell
./NVIDIA-Linux-x86_64-535.86.05.run --no-kernel-module
```

### Enable NVIDIA Persistent Inside LXC

To ensure the persistence of software state, similar to what was done for the host, it is essential to enable the daemon for maintaining this persistence within the NVIDIA driver:

```shell
nvidia-persistenced
```

### Verify NVIDIA Driver Installation

In conclusion, validate the installation of the NVIDIA driver within the LXC container.
If all processes have proceeded smoothly, the final step involves confirming the functionality of the NVIDIA driver.
As previously done, utilize the same command to inspect the output:

```shell
nvidia-smi
```

In this case as well, the output of this command should appear like the following:

```console
root@docker:~# nvidia-smi 
Fri Aug 21 21:34:09 2023       
+---------------------------------------------------------------------------------------+
| NVIDIA-SMI 535.86.05              Driver Version: 535.86.05    CUDA Version: 12.2     |
|-----------------------------------------+----------------------+----------------------+
| GPU  Name                 Persistence-M | Bus-Id        Disp.A | Volatile Uncorr. ECC |
| Fan  Temp   Perf          Pwr:Usage/Cap |         Memory-Usage | GPU-Util  Compute M. |
|                                         |                      |               MIG M. |
|=========================================+======================+======================|
|   0  NVIDIA GeForce GTX 950M        Off | 00000000:01:00.0 Off |                  N/A |
| N/A   42C    P8              N/A / 200W |      0MiB /  2048MiB |      0%      Default |
|                                         |                      |                  N/A |
+-----------------------------------------+----------------------+----------------------+
                                                                                         
+---------------------------------------------------------------------------------------+
| Processes:                                                                            |
|  GPU   GI   CI        PID   Type   Process name                            GPU Memory |
|        ID   ID                                                             Usage      |
|=======================================================================================|
|  No running processes found                                                           |
+---------------------------------------------------------------------------------------+
```

Congratulations, you've done it!
Now you have the ability to utilize an NVIDIA GPU within any LXC container of your choice.
This opens up a world of possibilities for GPU-accelerated tasks and applications, enhancing the capabilities of your Proxmox node.

## References

1. Jocke. (2022, February 23). [Plex GPU transcoding in Docker on LXC on Proxmox](https://jocke.no/2022/02/23/plex-gpu-transcoding-in-docker-on-lxc-on-proxmox/).

2. kuanghan. (2019, January 3). [Proxmox Plex Nvidia HW transcoding with LXC containers](https://gist.github.com/kuanghan/9aa5dfea243ed109c0878267e2d80b13#edit-the-config-file-for-this-container).

3. NVIDIA/nvidia-docker. (2023). [Issue #1648: Issue with Nvidia driver installations inside LXC containers](https://github.com/NVIDIA/nvidia-docker/issues/1648#issuecomment-1161813687).
