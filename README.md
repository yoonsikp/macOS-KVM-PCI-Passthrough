# macos-kvm-pci-passthrough
A guide to macOS virtualization on Ubuntu Server 17.10

Preface: I wanted to run macOS on my workstation, since macOS is a more friendly (yet proprietary) OS than Linux. However, I still wanted to run a Linux Server, mainly to manage my ZFS harddrive array. Virtualizing Linux on a macOS host, and then passing the VM the harddisks may potentially wreak havoc on my ZFS array. I ended up having to use Linux as the host. Not bad, since we don't have to deal with the hardware variety that Hackintosh (c) users must endure.

Virtualization technology has changed since I first learned about. The two biggest new features are KVM (Kernel-based Virtual Machine) and PCIe-Passthrough. KVM allows near-native usage of the CPU, while PCIe-Passthrough allows *native* passing of the PCI device to the guest. If you passthrough a graphics card, it will even allow you to do gaming, HDMI/DisplayPort audio, etc. 

You will need a Mac in order to download and create the install image.


My system: 
```
Motherboard: AsRock Rack C236M WS
Chipset: C236
RAM: 8GB Crucial ECC
CPU: i3-6100
GPU: Intel HD 530 & TBA
Ethernet: Intel I210 and I219-LM
SSD: Samsung SM951 NVMe
HDD: 2 x WD Red 3TB
OS: Ubuntu Server 17.10
```
There are two PCIe devices I wish to passthrough:

-Intel I219-LM (Ethernet)
There is poor support in macOS for Intel I210

-Graphics Card
Self Explanatory


First we need to install qemu (the emulator), libvirt (the VM daemon), virtinst (the manager):
```sudo apt install qemu 
sudo apt-get install qemu-kvm libvirt-bin virtinst bridge-utils cpu-checker
```
We also need to install `libguestfs-tools` in order to create a Clover bootloader.
```sudo apt install libguestfs-tools
```


We will download the script for making the bootloader:
```
wget https://www.kraxel.org/cgit/imagefish/tree/scripts/clover-image.sh
```
Open the file for editing:
```
nano clover-image.sh
```
Change in the first line `/bin/sh` to `/bin/bash` (might be fixed in future)

Now download the Clover Bootloader iso from the following webpage:
https://sourceforge.net/projects/cloverefiboot/files/Bootable_ISO/

```
tar --lzma -xvf CloverISO-4289.tar.lzma
```

Lastly, download the config.plist from the git directory:


```
sudo ./clover-image.sh --iso Clover-v2.4k-4289-X64.iso --img fun.raw --cfg config.plist.stripped.qemu
```



