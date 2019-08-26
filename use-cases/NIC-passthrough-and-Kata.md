# How To Pass a Physical NIC Into a Kata Container

* [Before you start (Restrictions, Requirements and Assumptions)](#before-you-start-restrictions-requirements-and-assumptions)
* [Pass a physical NIC into a Kata Container](#Pass-a-physical-NIC-into-a-Kata-Container)
  * [Part 1 – The Host](#part-1--the-host)
  * [Part 2 – The Container](#part-2--the-container)
* [Test Your Setup](#Test-Your-Setup)

![](./images/NIC%20passthrough%20Diagram.png)


In this guide we walk through the process of passing a physical NIC into a KC (Kata Container). The "usual" way that a KC is wired for communication is through a series of bridges and virtual NICs as can be seen in the Image above (marked as default). 

For some use cases, the container demands a direct link to the physical port of the host device, for example in a situation were the container is required to route high BW traffic without having support for acceleration such as SR-IOV.

## Before you start (Restrictions, Requirements and Assumptions)  

The method described in this guide relays on your system [supporting  IOMMU.]( https://en.wikipedia.org/wiki/Input–output_memory_management_unit#Published_specifications) before you start, make sure that your system has this attribute. If it does not, you should look for a different solution then the one presented here. 

If you are uncertain about your system’s support, the first two steps in [Part 1 – The Host](#part-1--the-host) should help you figure this out. 

 This guide was tested using *Intel(R) Xeon(R) CPU D-1548 @ 2.00GHz* and [support for VT-d](https://en.wikipedia.org/wiki/X86_virtualization#Intel-VT-d), Operated by Ubuntu 18.04.3 LTS. The general steps proposed is this guide should match other hardware and OS with some modifications.

This guide’s starting point is a system with an installed and working Docker and Kata Containers.

If that is not the case for you:

- [How to install Kata Containers](https://github.com/kata-containers/documentation/tree/master/install)

- [How to install Docker](https://github.com/kata-containers/documentation/blob/master/install/docker/ubuntu-docker-install.md) 



## Pass a physical NIC into a Kata Container 

### Part 1 – The Host

**Note:** All next commands are run as a super-user. Use `$ sudo -i` to become a super-user.



#### Configure system to load with IOMMU support.

   Edit grub file ( `/etc/default/grub` )

```bash
$ sudo vi /etc/default/grub
```

And add ` intel_iommu=on iommu=pt` To the **GRUB_CMDLINE_LINUX** line 

> GRUB_CMDLINE_LINUX=" … quiet intel_iommu=on iommu=pt"



If using UEFI: (like I do) 

```bash
$ sudo grub2-mkconfig -o /boot/efi/EFI/ubuntu/grub.cfg
```

Else (not using UEFI):

```bash
$ sudo grub2-mkconfig -o /boot/grub2/grub.cfg
```

#### Test that the system loads with IOMMU:

```bash
$ sudo dmesg | grep -ie iommu
```


#### Enable the *hotplug_vfio_on_root_bus* configuration in the `kata configuration.toml` file:

```bash
$ sudo vi /usr/share/defaults/kata-containers/configuration.toml
```
a. Set hotplug_vfio_on_root_bus = true

b. Make sure you are using the PC machine type by verifying machine_type = "pc"



#### Install *lshw* and *pciutils* (Ubuntu)

```bash
$ sudo apt install lshw pciutils
```


#### Find PCI address for the interface you wish to pass to the container and place it in a variable

Say I want to pass **enp0s20f1** to the container (see what interfaces you have by running `$ ip link` ). 

Get address: 

```bash
$ NIC=<interface>
```
For example: 
```bash
$ NIC=enp0s20f1
```
Place in BDF:
```bash
$ BDF=$(sudo lshw -class network -businfo -numeric | grep ${NIC} | awk '{print $1;}' | cut -d@ -f2)
```
Test BDF:
```bash
$ echo $BDF
```

```
Expected results (something like):

0000:00:14.1
```

#### Load *vfio-pci*  kernel module

```bash
$ sudo modprobe -i vfio-pci
```
Test that the module loaded correctly: 

```bash
$ sudo lsmod | grep vfio
```

#### Unbind the Interface from its current driver. 

```bash
$ sudo echo $BDF | sudo tee /sys/bus/pci/devices/$BDF/driver/unbind
```

#### Find vendor & device ID

```bash
$ sudo lspci -n -s $BDF
```

```
Expected results (something like):

00:14.1 0200: 8086:1f40 (rev 03)
```


#### Bind the device to vfio-pci using the result of the previous step.

```bash
$ sudo echo 8086 1f40 | sudo tee /sys/bus/pci/drivers/vfio-pci/new_id
```

> **Note:** The ` xxxx xxxx ` format is of the correct form and not a mistake.

#### Check what the new device name is, now under vfio

```bash
$ sudo ls /dev/vfio
```
```
Expected results (something like):

 39  vfio
```

> **Note:** The requested outcome here is the newly created “39”.

#### All set here. Time to move to the Container’s side of the deal.


### Part 2 – The Container

#### Create the container you wish to pass the NIC into

```bash
docker run -it --runtime=kata-runtime --name <choose a name> --device /dev/vfio/<device name from previous step> -v /dev:/dev <image> <command>
```
   For example:

```bash
docker run -it --runtime=kata-runtime --name vfio_con --device /dev/vfio/39 -v /dev:/dev ubuntu bash
```
If you aren’t sure what just happened back there, here’s a breakdown of the command:
- `docker run` - creates and runs the container. 
- ` -it ` - docker command flags for “interactive” and “tty”.
- ` --runtime=kata-runtime ` - tells docker to run the container as a KC (if you set kata-runtime as the default runtime, this option is redundant).
- ` --name vfio_con ` – name the container. It can be any name you choose. 
- ` --device /dev/vfio/39 ` – this one is the important part. With this command, you tell docker to pass the NIC to the container. 
- `-v /dev:/dev `– here you tell docker to map the devices folder of the host to the devices folder of the container.
- ` ubuntu `– choose the container image you want to run. 
- ` bash ` - the command you what the container to run. 

 

#### Find forwarded device
Update and install pciutils (Ubuntu)

```bash
$ sudo apt update ; apt install pciutils
```

List devices:
```bash
$ sudo lspci –nn -D
```

> **Note:** The [XXXX:XXXX] address you are looking for is the same one you used in the previous steps.

#### Bind the passed device with a driver in the container.
/ TODO

## Test Your Setup
/TODO
