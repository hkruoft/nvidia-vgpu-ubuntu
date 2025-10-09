# nvidia-vgpu-ubuntu

A guide on how to make NVIDIA vGPU [v17.6](https://docs.nvidia.com/vgpu/17.0/) work on Ubuntu 22.04

## Summary
1) Check the support matrix and see what works best for you. For me it was this version: [v17.6](https://docs.nvidia.com/vgpu/17.0/product-support-matrix/index.html#abstract__ubuntu)
2) If you are still testing or trying things out, you can setup an Enterprise Account that will give you a [FREE 90-day evaluation license](https://www.nvidia.com/en-us/data-center/resources/vgpu-evaluation/)
3) Setup License server & generate token
4) Install vGPU manager and Host drivers on your main Host (baremetal)
5) Enable SRIOV and enable virtual functions on Host GPU
6) Create your vGPU
7) Create your VM & link your vGPU to the VM
8) Install GRIDD drivers (Guest Drivers)
9) Add license token 
10) Cross your fingers and hope everything works ¯\\\_(ツ)_/¯ 

## First things first!!
Before even getting started, figure out what version of NVIDIA vGPU do you need. There is a [Support Matrix](https://docs.nvidia.com/vgpu/15.0/product-support-matrix/index.html#abstract__ubuntu) page detailing what your Host OS needs to be, and which VM OS(s) are supported. This will be important later when you are setting up your VM OS.

## License Server (needed for vGPU driver to work on VM)

Why do I need this? Well, without this the vGPU driver on the VM does not work. The moment you run `nvidia-smi` , syslog on the VM starts spamming errors since NVIDIA blocks the driver from running. So you need the license token to make the VM work.

### 1) Setup Enterprise account (**Free** 90-day eval license)
- If you're a new user, go to the following link to start your 90 day eval license to test out the vGPU functionality: https://www.nvidia.com/en-us/data-center/resources/vgpu-evaluation/

### 2) Creating a License Server on the NVIDIA Licensing Portal
- Log in to the [NVIDIA Application Hub](http://nvid.nvidia.com/dashboard/) and click **NVIDIA LICENSING PORTAL** to go to the NVIDIA Licensing Portal.
- In the left navigation pane of the NVIDIA Licensing Portal dashboard, expand **LICENSE SERVER** and click **CREATE SERVER**. The Create License Server wizard is started.

<img src="https://i.imgur.com/FFe1dQw.png" width="300">

- The Create License Server wizard opens:

![alt text](https://i.imgur.com/5sbcYX5.png)

- On the Create License Server page of the wizard, step through the configuration requirements to provide the details of your license server.
  - **Step 1 – Identification**: In the **Name** field, enter your choice of name for the license server and in the **Description** field, enter a text description of the license server. The description is required and will be displayed on the details page for the license server that you are creating.
  - **Step 2 – Features**: Select one or more available features from your entitlements to allot to this license server. Can select all, I had the following show up: 
      - **NVIDIA RTX Virtual Workstation-5.0** 
      - **NVIDIA Virtual Applications-3.0**
  
  - **Step 2 (Cont'd)**: It will also ask you to pick the the number of "Concurrent Users". Each of the one above has 128 CCUs with the 90-day license. We did 2 each, since we only needed two VMs for the research

  - **Step 3 - Environment**: Select **Cloud (CLS)** to install this license server. To make the selection after the license server has been created, select the **Deferred** option.
  - **Step 4 – Configuration**: From the **Leasing mode** drop-down list, select the following leasing mode (this is what worked for me):
    - **Standard Networked Licensing**

      [*Select this mode to simplify the management of licenses on a license server that supports networked licensing. In this mode, no additional configuration of the licenses on the server is required.*]

  - Click **REVIEW SUMMARY** to review the configuration summary before creating the license server.
- On the Create License Server page, from the **Step 4 – Configuration** menu, click the **CREATE SERVER** option to create this license server. Alternatively, you can click **CREATE SERVER** on the Server Summary page.

### 3) Creating a CLS *Instance* on the NVIDIA Licensing Portal
- In the left navigation pane of the NVIDIA Licensing Portal dashboard, click **SERVICE INSTANCES**.

<img src="https://i.imgur.com/iIcfMoU.png" width="300">

- You should already see a Service Instance that has been auto-created 

- For me it was named something like: `00q8z00001wfh8ouaw-2025-10-03_20-22`

> **But it didn't autocreate for me!!**

If it did **NOT** autocreate, then you can follow the steps here:
 - [Creating a CLS Instance on the NVIDIA Licensing Portal](https://docs.nvidia.com/vgpu/17.0/grid-software-quick-start-guide/index.html#creating-cls-instance-on-portal)
 - [Binding a License Server to a Service Instance](https://docs.nvidia.com/vgpu/17.0/grid-software-quick-start-guide/index.html#binding-license-server-on-portal-to-cls-instance)
- [Installing a License Server on a CLS Instance](https://docs.nvidia.com/vgpu/17.0/grid-software-quick-start-guide/index.html#installing-license-server-on-cls-instance)

### 4) Generating a Client Configuration Token for a CLS Instance

- In the left navigation pane, click **SERVICE INSTANCES**

<img src="https://i.imgur.com/iIcfMoU.png" width="300">

- On the Service Instances page that opens, from the **Actions** menu for the CLS instance for which you want to generate a client configuration token, choose **Generate client configuration token**.
- In the **Generate Client Configuration Token** pop-up window that opens, select the references that you want to include in the client configuration token.
  - From the list of scope references, select the scope references that you want to include

  ![alt text](https://i.imgur.com/81OnQAS.png)

  - In the Expiration section, select an expiration date for the client configuration token. If you do not select a date, the default token expiration time is 12 years if you select "**Token never expires**"

  - Click **DOWNLOAD CLIENT CONFIGURATION TOKEN**.
A file named `client_configuration_token_mm-dd-yyyy-hh-mm-ss.tok` is saved to your default downloads folder.

- Copy this token on your main Host where you will be doing the vGPU setup. This will be used later when we create the VMs

## Installing and configuring vGPU manager and drivers

### Prerequisite:
#### [!!!] Change `displaymode` of GPU on main Host
  (This step is <ins>**important</ins> !!** )
- Download the [`displaymodeselector`](https://developer.nvidia.com/displaymodeselector) tool & change the display mode to "compute"
        
    - Check current mode: 
    ```
    sudo ./displaymodeselector —listgpumodes
    ``` 
    this shows the current mode (graphics or compute)
    - Change to compute for display-less mode: `sudo ./displaymodeselector --gpumode compute` (only needed for A6000, not for server-grade GPUs)
    - Now 
    ```
    sudo ./displaymodeselector —listgpumodes
    ```
     shows `compute`

### Setup the drivers
- In the left navigation pane, click **SOFTWARE DOWNLOADS**
- If you've checked the Support Matrix [above](#first-things-first) you should know what version works best for your GPU and Ubuntu OS version
- Find your download package and click Download. (Mine was `Complete vGPU 17.6 package for Linux KVM including support driver`. [[ *Why Linux and not Ubuntu?* The `Linux` one comes with `*.run` files which I found better for 22.04 Host Driver installation) ]] 
- Download the zip and extract it on your main Host. You'll see `Host Drivers` and `Guest Drivers`.
  - On your baremetal server/machine, install the driver that is in the `Host Drivers` folder. The `*.run` file works better in my experience.

### Enable SR-IOV
[ **IMPORTANT:** Use only the custom script `sriov-manage` provided by NVIDIA vGPU software for this purpose. **Do NOT** try to enable the virtual function for the GPU by any other means]

- Enable SRIOV (I did `ALL` since I only have one GPU on the server) 
    ```
    sudo /usr/lib/nvidia/sriov-manage -e ALL
    ```

### Check if your Virtual Functions are enabled
- Obtain the PCI device bus/device/function (BDF) of the physical GPU
```
$ lspci | grep NVIDIA

51:00.0 VGA compatible controller: NVIDIA Corporation GA102GL [RTX A6000] (rev a1)
``` 
So if we check the following, we will see that Virtual Functions should have been created from the `sriov-manage` step above.

```
$ ls -l /sys/bus/pci/devices/0000:51:00.0 | grep virtfn

lrwxrwxrwx 1 root root           0 Oct  3 14:23 virtfn0 -> ../0000:51:00.4
lrwxrwxrwx 1 root root           0 Oct  3 14:23 virtfn1 -> ../0000:51:00.5
lrwxrwxrwx 1 root root           0 Oct  3 14:23 virtfn10 -> ../0000:51:01.6
.
.
.
```

- If you've got this far and everything has worked, then so far so good.

### Creating an NVIDIA vGPU that Supports SR-IOV on a Linux with KVM Hypervisor

An NVIDIA vGPU that supports SR-IOV resides on a physical GPU that supports SR-IOV, such as a GPU based on the **NVIDIA Ampere architecture**. If your GPU does not support this, then follow the [LEGACY vGPU setup steps](https://docs.nvidia.com/vgpu/17.0/grid-vgpu-user-guide/index.html#creating-legacy-vgpu-device-red-hat-el-kvm)

- Change to the `mdev_supported_types` directory for the virtual function on which you want to create the vGPU.

`# cd /sys/class/mdev_bus/domain\:bus\:vf-slot.v-function/mdev_supported_types/`

For example: 
`cd /sys/class/mdev_bus/0000\:51\:00.4/mdev_supported_types`

- Find out which subdirectory of `mdev_supported_types` contains registration information for the vGPU type that you want to create.

<pre> # grep -l "<i>vgpu-type</i>" nvidia-*/name` </pre>

**vgpu-type**:
The vGPU type, for example, `A6000-24Q`.

This example shows that the registration information for the `A6000-24Q` vGPU type is contained in the nvidia-532 subdirectory of `mdev_supported_types`.

``` 
# grep -l "A6000-24Q" nvidia-*/name
nvidia-532/name 
```

- Confirm that you can create an instance of the vGPU type on the virtual function.
<pre>
# cat <i>subdirectory</i>/available_instances
</pre>

**subdirectory**:
The subdirectory that you found in the previous step, for example, `nvidia-532`.
The number of available instances must be 1. If the number is 0, a vGPU has already been created on the virtual function. Only one instance of any vGPU type can be created on a virtual function.

This example shows that an instance of the A40-2Q vGPU type can be created on the virtual function.

```
# cat nvidia-532/available_instances
1
```

- Generate a correctly formatted universally unique identifier (UUID) for the vGPU.
```
# uuidgen
6240e2b9-61fe-4fca-809c-9a8000c3fa33
```
- Write the UUID that you obtained in the previous step to the create file in the registration information directory for the vGPU type that you want to create.

<pre>
# echo "<i>uuid</i>"> <i>subdirectory</i>/create
</pre>
**uuid**:
The UUID that you generated in the previous step, which will become the UUID of the vGPU that you want to create.

**subdirectory**:
The registration information directory for the vGPU type that you want to create, for example, `nvidia-532`.
This example creates an instance of the `A6000-24Q` vGPU type with the UUID `6240e2b9-61fe-4fca-809c-9a8000c3fa33`.

```
# echo "6240e2b9-61fe-4fca-809c-9a8000c3fa33" > nvidia-532/create
```
An mdev device file for the vGPU is added to the parent virtual function directory of the vGPU. The vGPU is identified by its UUID.

- **Time-sliced vGPUs only**: Make the mdev device file that you created to represent the vGPU persistent.

<pre>
# mdevctl define --auto --uuid <i>uuid</i>
</pre>

**uuid**: 
The UUID that you specified in the previous step for the vGPU that you are creating.

- Confirm that the vGPU was created.
Confirm that the /sys/bus/mdev/devices/ directory contains a symbolic link to the `mdev` device file.

<pre>
# mdevctl list
6240e2b9-61fe-4fca-809c-9a8000c3fa33 0000:51:00.4 nvidia-532 (defined)
</pre>

<pre>
# ls -l /sys/bus/mdev/devices/
total 0
lrwxrwxrwx 1 root root 0 Oct  2 12:26 6240e2b9-61fe-4fca-809c-9a8000c3fa33 -> ../../../devices/pci0000:50/0000:50:03.1/0000:51:00.4/6240e2b9-61fe-4fca-809c-9a8000c3fa33

</pre>

## Virtual Machines
### Initial VM setup
Next step is to create the VM. I used `libvirt` with `virt-install` , you can use `qemu` or `virt-manager` , however this guide will only go over `libvirt`

- Download your desired Ubuntu ISO that will be used for the VM somewhere on your baremetal machine (mine was in `/mnt/ubuntu-iso` so that's what you will see below)
```
$ cd /mnt/ubuntu-iso/
$ wget https://mirror.csclub.uwaterloo.ca/ubuntu-releases/22.04/ubuntu-22.04.5-live-server-amd64.iso
```

(Before the next step, make you SSH with X window [`ssh -X hostname`] or use a VNC client)

Here is what I ran to create the VM, your mileage may vary:

```
virt-install   --name ubuntu2204   --memory 8096   --vcpus 2   --disk size=80   --cdrom /mnt/ubuntu-iso/ubuntu-22.04.5-live-server-amd64.iso   --os-variant ubuntu22.04   --network network=default --graphics vnc
```

- Setup your VM over the VNC connection. You can also attach to it with `virt-viewer` like this 
```
virt-viewer --connect qemu:///system --wait ubuntu2204
```

- After your VM OS installation is done, make sure to "eject" the iso. From your Host (make sure you select the correct type `sda`/`hda`/`cdrom`):

```
virsh change-media ubuntu2004 sda --eject
```

- Shutdown the VM from Host:
```
virsh shutdown ubuntu2004
```
### Link the vGPU to the VM
Edit the VM's XML config:
```
virsh edit ubuntu2204
```
For a hypervisor that uses the mdev VFIO framework, add a device entry that identifies the vGPU through its UUID as follows:

From the step where we figured out what the PCIe BDF was, add the following lines in the VM's XML config:


```
<devices>
.
.
.
  <hostdev mode='subsystem' type='mdev' model='vfio-pci'>
    <source>
      <address uuid='uuid'/>
    </source>
  </hostdev>
</devices>
```

**uuid**: 
The UUID that was assigned to the vGPU when the vGPU was created.
This example adds a device entry for the vGPU with the UUID `6240e2b9-61fe-4fca-809c-9a8000c3fa33`.


### Restart the VM and install Guest Drivers

```
# virsh shutdown ubuntu2204
# virsh start ubuntu2204
```

Connect to the VM and copy the "Guest Drivers" from this [step](#setup-the-drivers) onto somewhere in the VM.

Then, install the Guest drivers (file name should be `NVIDIA-Linux-*-grid.run`)

### Create the gridd.conf file

Inside the VM, copy the `gridd.conf.template` file and create a `gridd.conf` file

```
GuestVM#  cp /etc/nvidia/gridd.conf.template /etc/nvidia/gridd.conf
```

And then add the `FeatureType` configuration parameter to the file `/etc/nvidia/gridd.conf` on a new line as `FeatureType="value"`.
value depends on the type of the GPU assigned to the licensed client that you are configuring.

```
# /etc/nvidia/gridd.conf.template - Configuration file for NVIDIA Grid Daemon
…
# Description: Set Feature to be enabled
# Data type: integer
# Possible values:
# 0 => for unlicensed state
# 1 => for NVIDIA vGPU
# 2 => for NVIDIA RTX Virtual Workstation
# 4 => for NVIDIA Virtual Compute Server
FeatureType=1
...
```

### Add the .tok token 
Inside your VM still, copy the client configuration token from this earlier [step](#4-generating-a-client-configuration-token-for-a-cls-instance) to the `/etc/nvidia/ClientConfigToken` directory.

Ensure that the file access modes of the client configuration token allow the owner to read, write, and execute the token, and the group and others only to read the token.

### Restart the GRIDD service

```
GuestVM# systemctl restart nvidia-gridd.service
```

### Check license

If all goes well (hopefully), you should see the license applied succesfully:

```
GuestVM# nvidia-smi -q | grep Lic
    vGPU Software Licensed Product
        License Status                    : Licensed (Expiry: 2025-10-5 1:57:32 GMT)

```

```
GuestVM# nvidia-smi
Fri Oct  3 01:48:17 2025       
+-----------------------------------------------------------------------------+
| NVIDIA-SMI 550.54.15   Driver Version: 550.54.15   CUDA Version: 12.4       |
|-------------------------------+----------------------+----------------------+
| GPU  Name        Persistence-M| Bus-Id        Disp.A | Volatile Uncorr. ECC |
| Fan  Temp  Perf  Pwr:Usage/Cap|         Memory-Usage | GPU-Util  Compute M. |
|                               |                      |               MIG M. |
|===============================+======================+======================|
|   0  NVIDIA RTXA6000-24Q  On  | 00000000:07:00.0 Off |                    0 |
| N/A   N/A    P8    N/A /  N/A |      0MiB / 24576MiB |      0%      Default |
|                               |                      |             Disabled |
+-------------------------------+----------------------+----------------------+                     
+-----------------------------------------------------------------------------+
| Processes:                                                                  |
|  GPU   GI   CI        PID   Type   Process name                  GPU Memory |
|        ID   ID                                                   Usage      |
|=============================================================================|
|  No running processes found                                                 |
+-----------------------------------------------------------------------------+
