# nvidia-vgpu-ubuntu

A guide on how to make NVIDIA vGPU [v15.4](https://docs.nvidia.com/vgpu/15.0/) work on Ubuntu 22.04

## Summary
1) Check the support matrix and see what works best for you. For me it was this version: [v15.4](https://docs.nvidia.com/vgpu/15.0/product-support-matrix/index.html#abstract__ubuntu)
2) If you are still testing or trying things out, you can setup an Enterprise Account that will give you a [FREE 90-day evaluation license](https://www.nvidia.com/en-us/data-center/resources/vgpu-evaluation/)
## First things first!!
Before even getting started, figure out what version of NVIDIA vGPU do you need. There is a [Support Matrix](https://docs.nvidia.com/vgpu/15.0/product-support-matrix/index.html#abstract__ubuntu) page detailing what your Host OS needs to be, and which VM OS(s) are supported. This will be important later

## Setup Enterprise account (**Free** 90-day eval license)
- If you're a new user, go here to start your 90 day eval license to test out the vGPU functionality: https://www.nvidia.com/en-us/data-center/resources/vgpu-evaluation/

## Download the drivers zip file
- Once that is done, go here: https://ui.licensing.nvidia.com/software

- If you've checked the Support Matrix [above](#first-things-first) you should know what version works best for your GPU and Ubuntu OS version

## Setup the drivers
- Download the zip and extract it. You'll see `Host Drivers` and `Guest Drivers`.
  - On your baremetal server/machine, install the driver that is in the `Host Drivers` folder. The `*.run` file runs better in my experience
  
  ## [!!] Change `displaymode` of GPU
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
     shows “compute”

## Enable SR-IOV
[Use only the custom script `sriov-manage` provided by NVIDIA vGPU software for this purpose. **Do NOT** try to enable the virtual function for the GPU by any other means]
- Enable SRIOV (I did `ALL` since I only have one GPU) 
    ```
    sudo /usr/lib/nvidia/sriov-manage -e ALL
    ```

## Check if your Virtual Functions are enabled
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
$ wget https://mirror.csclub.uwaterloo.ca/ubuntu-releases/20.04/ubuntu-20.04.6-live-server-amd64.iso
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
