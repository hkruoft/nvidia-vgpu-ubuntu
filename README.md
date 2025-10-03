# nvidia-vgpu-ubuntu

A guide on how to make NVIDIA vGPU [v15.4](https://docs.nvidia.com/vgpu/15.0/) work on Ubuntu 22.04

## Summary
1) Check the support matrix and see what works best for you. For me it was this version: [v15.4](https://docs.nvidia.com/vgpu/15.0/product-support-matrix/index.html#abstract__ubuntu)
2) If you are still testing or trying things out, you can setup an Enterprise Account that will give you a [FREE 90-day evaluation license](https://www.nvidia.com/en-us/data-center/resources/vgpu-evaluation/)
## First things first!!
Before even getting started, figure out what version of NVIDIA vGPU do you need. There is a [Support Matrix](https://docs.nvidia.com/vgpu/15.0/product-support-matrix/index.html#abstract__ubuntu) page detailing what your Host OS needs to be, and which VM OS(s) are supported. This will be important later

## Setup Enterprise account (**Free** 90-day eval license)
- If you're a new user, go here to start your 90 day eval license to test out the vGPU functionality: https://www.nvidia.com/en-us/data-center/resources/vgpu-evaluation/

- Once that is done, go here: https://ui.licensing.nvidia.com/software

- If you've checked the Support Matrix [above](#first-things-first) you should know what version works best for your GPU and Ubuntu OS version

- Download the zip and extract it. You'll see `Host Drivers` and `Guest Drivers`.
  - On your baremetal server/machine, install the driver that is in the `Host Drivers` folder. The `*.run` file runs better in my experience
  
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
    - Enable SRIOV (I did `ALL` since I only have one GPU) 
    ```
    sudo /usr/lib/nvidia/sriov-manage -e ALL
    ```
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
- Next step is to create the VM. I used `libvirt` with `virt-install` , you can use `qemu` or `virt-manager` , however this guide will only go over `libvirt`
- 
