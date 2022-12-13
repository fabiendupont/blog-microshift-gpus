# blog-microshift-gpus
In this blog we will demonstrate how to enable Nvidia GPUs on an x86 system running RHEL 8.6 and Microshift. 
## Environment (Step 1)
Check your RHEL level and kernel level.
```bash
# cat /etc/redhat-release
Red Hat Enterprise Linux release 8.6 (Ootpa)
# uname -r 
4.18.0-372.32.1.el8_6.x86_64 
```
Nvidia drivers need to be installed for your specific kernel version. 
Precompiled kernel modules are availabe for the Nvidia drivers and have the advantage of being tested and validated against specific kernel versions. 
Look for your kernel version (in this example 4.18.0-372.32.1) in [Nvidia's precompiled kmod driver package table](https://developer.download.nvidia.com/compute/cuda/repos/rhel8/x86_64/precompiled/); locate the matching Nvidia driver version in the table.
The driver version and kernel verison string must match exactly.  If your kernel version is not in the Nvidia precompiled kmod driver package table you can use the Nvidia Dynamic Kernel Module Support (DKMS) package to generate linux kernel modules for the Nvidia driver. To install the Nvidia drivers with RHEL using DKMS follow [Nvidia's instructions](https://docs.nvidia.com/datacenter/tesla/tesla-installation-notes/index.html). 



## Verify the GPU installed on your system (Step 2)
```bash
# lspci -nnv |grep -i nvidia

example output:
17:00.0 3D controller [0302]: NVIDIA Corporation GA100GL [A30 PCIe] [10de:20b7] (rev a1)
	Subsystem: NVIDIA Corporation Device [10de:1532]
```

## Remove the Nouveau kernel Driver module (otherwise the Nvidia driver will not load), then install the Nvidia Driver and reboot (Step 3)
At this point in time the "latest" pre-compiled kernel modules for the Nvidia drivers support kernel version 4.18.0-372.32.1, as shown in [Nvidia's precompiled kmod driver package table](https://developer.download.nvidia.com/compute/cuda/repos/rhel8/x86_64/precompiled/), so "latest" is chosen here.  
```bash
# echo 'blacklist nouveau' >> /etc/modprobe.d/blacklist.conf
# dnf config-manager --add-repo=https://developer.download.nvidia.com/compute/cuda/repos/rhel8/x86_64/cuda-rhel8.repo
# dnf module install nvidia-driver:latest -y
# reboot
```

## Verify Nvidia Drivers are installed property (Step 4) 
```bash
# nvidia-smi 

example output:

# nvidia-smi
Mon Dec 12 20:22:27 2022       
+-----------------------------------------------------------------------------+
| NVIDIA-SMI 520.61.05    Driver Version: 520.61.05    CUDA Version: 11.8     |
|-------------------------------+----------------------+----------------------+
| GPU  Name        Persistence-M| Bus-Id        Disp.A | Volatile Uncorr. ECC |
| Fan  Temp  Perf  Pwr:Usage/Cap|         Memory-Usage | GPU-Util  Compute M. |
|                               |                      |               MIG M. |
|===============================+======================+======================|
|   0  NVIDIA A30          Off  | 00000000:17:00.0 Off |                    0 |
| N/A   27C    P0    29W / 165W |      0MiB / 24576MiB |      0%      Default |
|                               |                      |             Disabled |
+-------------------------------+----------------------+----------------------+
                                                                               
+-----------------------------------------------------------------------------+
| Processes:                                                                  |
|  GPU   GI   CI        PID   Type   Process name                  GPU Memory |
|        ID   ID                                                   Usage      |
|=============================================================================|
|  No running processes found                                                 |
+-----------------------------------------------------------------------------+
```


## Install Podman and crun, and verify that crun is the default OCI runtime (Step 5)
```bash
# dnf install -y crun
# dnf install -y podman
# cp /usr/share/containers/containers.conf /etc/containers/containers.conf
# sed -i 's/^# runtime = "crun"/runtime = "crun"/;' /etc/containers/containers.conf
```

## Install Nvidia-docker (Step 6)
Nvidia-docker supports [these distributions](https://nvidia.github.io/nvidia-docker/). Now we install nvidia-docker for RHEL 8.6 which matches our version (shown in step 1).
```bash
# curl -s -L https://nvidia.github.io/nvidia-docker/rhel8.6/nvidia-docker.repo | tee /etc/yum.repos.d/nvidia-docker.repo
# dnf install nvidia-container-toolkit -y![image](https://user-images.githubusercontent.com/3208719/207197182-7067db89-acfd-4dcc-afc0-ee3b29113daa.png)
# curl -LO https://raw.githubusercontent.com/NVIDIA/dgx-selinux/master/bin/RHEL8/nvidia-container.pp
```

## Add the nvidia-container SELinux policy to our machine (Step 7)
```bash
# curl -LO https://raw.githubusercontent.com/NVIDIA/dgx-selinux/master/bin/RHEL8/nvidia-container.pp
# semodule -i nvidia-container.pp
# nvidia-container-cli -k list | restorecon -v -f –
```



## Verify that the Nvidia Drivers and Podman are configured correctly (Step 8) 
```bash
# podman run --privileged -ti nvidia/cuda:11.0.3-base-ubuntu20.04 nvidia-smi


example output: 

Tue Dec 13 00:57:20 2022       
+-----------------------------------------------------------------------------+
| NVIDIA-SMI 520.61.05    Driver Version: 520.61.05    CUDA Version: 11.8     |
|-------------------------------+----------------------+----------------------+
| GPU  Name        Persistence-M| Bus-Id        Disp.A | Volatile Uncorr. ECC |
| Fan  Temp  Perf  Pwr:Usage/Cap|         Memory-Usage | GPU-Util  Compute M. |
|                               |                      |               MIG M. |
|===============================+======================+======================|
|   0  NVIDIA A30          Off  | 00000000:17:00.0 Off |                    0 |
| N/A   27C    P0    29W / 165W |      0MiB / 24576MiB |      0%      Default |
|                               |                      |             Disabled |
+-------------------------------+----------------------+----------------------+
                                                                               
+-----------------------------------------------------------------------------+
| Processes:                                                                  |
|  GPU   GI   CI        PID   Type   Process name                  GPU Memory |
|        ID   ID                                                   Usage      |
|=============================================================================|
|  No running processes found                                                 |
+-----------------------------------------------------------------------------+
```

## Install MicroShift (Step 9) 
Register your system with subscription manager
```bash

e.g. 

~]# subscription-manager register
Registering to: subscription.rhsm.redhat.com:443/subscription
Username: <your username>
Password: <your password>
```
Then refer to the Microshift Doc and follow steps in the [installation guide](https://microshift.io/docs/getting-started/). 



## Install Nvidia GPU Operand (Step 10)
Note: Nvidia GPU Operator is not yet available for MicroShift. So we will do the part of the Nvidia GPU Operator by installing the operand using helm.

