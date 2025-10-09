# nvidia-vgpu-ubuntu

A guide on how to make NVIDIA vGPU [v17.6](https://docs.nvidia.com/vgpu/17.0/) work on Ubuntu 22.04

## Summary
1) Check the support matrix and see what works best for you. For me it was this version: [v17.6](https://docs.nvidia.com/vgpu/17.0/product-support-matrix/index.html#abstract__ubuntu)
2) If you are still testing or trying things out, you can setup an Enterprise Account that will give you a [FREE 90-day evaluation license](https://www.nvidia.com/en-us/data-center/resources/vgpu-evaluation/)

## First things first!!
Before even getting started, figure out what version of NVIDIA vGPU do you need. There is a [Support Matrix](https://docs.nvidia.com/vgpu/15.0/product-support-matrix/index.html#abstract__ubuntu) page detailing what your Host OS needs to be, and which VM OS(s) are supported. This will be important later when you are setting up your VM OS.

## License Server (needed for vGPU driver to work on VM)
### 1) Setup Enterprise account (**Free** 90-day eval license)
- If you're a new user, go to the following link to start your 90 day eval license to test out the vGPU functionality: https://www.nvidia.com/en-us/data-center/resources/vgpu-evaluation/

### 2) Creating a License Server on the NVIDIA Licensing Portal
- Log in to the [NVIDIA Application Hub](http://nvid.nvidia.com/dashboard/) and click **NVIDIA LICENSING PORTAL** to go to the NVIDIA Licensing Portal.
- In the left navigation pane of the NVIDIA Licensing Portal dashboard, expand **LICENSE SERVER** and click **CREATE SERVER**. The Create License Server wizard is started.

![alt text](https://i.imgur.com/FFe1dQw.png)

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

![alt text](https://i.imgur.com/iIcfMoU.png)

- You should already see a Service Instance that has been auto-created 

- For me it was named something like: `00q8z00001wfh8ouaw-2025-10-03_20-22`

> **But it didn't autocreate for me!!**

If it did **NOT** autocreate, then you can follow the steps here:
 - [Creating a CLS Instance on the NVIDIA Licensing Portal](https://docs.nvidia.com/vgpu/17.0/grid-software-quick-start-guide/index.html#creating-cls-instance-on-portal)
 - [Binding a License Server to a Service Instance](https://docs.nvidia.com/vgpu/17.0/grid-software-quick-start-guide/index.html#binding-license-server-on-portal-to-cls-instance)
- [Installing a License Server on a CLS Instance](https://docs.nvidia.com/vgpu/17.0/grid-software-quick-start-guide/index.html#installing-license-server-on-cls-instance)

### 4) Generating a Client Configuration Token for a CLS Instance

- In the left navigation pane, click **SERVICE INSTANCES**

![alt text](https://i.imgur.com/iIcfMoU.png)

- On the Service Instances page that opens, from the **Actions** menu for the CLS instance for which you want to generate a client configuration token, choose **Generate client configuration token**.
- In the **Generate Client Configuration Token** pop-up window that opens, select the references that you want to include in the client configuration token.
  - From the list of scope references, select the scope references that you want to include

  ![alt text](https://i.imgur.com/f6M3aVS.png)

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

## Virtual Machines
### Initial VM setup
Next step is to create the VM. I used `libvirt` with `virt-install` , you can use `qemu` or `virt-manager` , however this guide will only go over `libvirt`

- Download your desired Ubuntu ISO that will be used for the VM somewhere on your baremetal machine (mine was in `/mnt/ubuntu-iso` so that's what you will see below)
```
$ cd /mnt/ubuntu-iso/
$ wget https://mirror.csclub.uwaterloo.ca/ubuntu-releases/22.04/ubuntu-22.04.6-live-server-amd64.iso
```
Here is what I ran to create the VM, your mileage may vary (I am using `ubuntu 20.04` because vGPU v15.4 doesn't seem to work with `ubuntu 22.04`, gives the known [GPL error](https://www.cs.toronto.edu/~jhancock/wlog/?p=328) ):

```
virt-install   --name ubuntu2004   --memory 8192   --vcpus 4   --disk path=/var/lib/libvirt/images/ubuntu2004_vgpu.qcow2,size=40,bus=virtio   --cdrom /mnt/ubuntu-iso/ubuntu-20.04.6-live-server-amd64.iso   --os-variant ubuntu20.04   --network bridge=virbr0,model=virtio,driver.name=vhost,driver.queues=4   --graphics vnc
```

- Setup your VM over the VNC connection. You can also attach to it with `virt-viewer` like this 
```
virt-viewer --connect qemu:///system --wait ubuntu2004
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
virsh edit ubuntu2004
```
From the step where we figured out what the PCIe BDF was, add the following lines in the VM's XML config:
```

```

