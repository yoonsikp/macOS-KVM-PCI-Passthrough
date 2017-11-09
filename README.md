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

##Creating the install image
Let's begin with the step that requires a Mac.
Download the High Sierra install from the App Store.
Click here to create the boot image: https://support.apple.com/en-us/HT201372

Now we need to convert that back into an `.img` file.
Type `diskutil list` to find out the name of your flash drive, and replace it in the following:
```
sudo dd if=/dev/disk3 of=~/10.13.1.img bs=1m
```
Copy over the `.img` file to your Ubuntu Server.


##Installing QEMU
First we need to install qemu (the emulator), libvirt (the VM daemon), virtinst (the manager) on Ubuntu:
```sudo apt install qemu 
sudo apt-get install qemu-kvm libvirt-bin virtinst bridge-utils cpu-checker
```

##Creating the Bootloader
We also need to install `libguestfs-tools` in order to create a Clover bootloader.
```
sudo apt install libguestfs-tools
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

Now download the latest Clover Bootloader iso from the following webpage:
https://sourceforge.net/projects/cloverefiboot/files/Bootable_ISO/

Extract it
```
tar --lzma -xvf CloverISO-4289.tar.lzma
```

Lastly, download the config.plist required by Clover from the git root directory:

```
wget https://raw.githubusercontent.com/yoonsikp/macos-kvm-pci-passthrough/master/config.plist
```
(Later, in order to get iMessage/iCloud working, we will have to edit this config.plist using Clover Configurator)

Now we can run the script:
```
chmod 777 ./clover-image.sh
sudo ./clover-image.sh --iso Clover-v2.4k-4289-X64.iso --img clover.raw --cfg config.plist
```
This results in a file called `clover.raw` being created in your current directory.


##Configuring UEFI (OVMF)


Next we need to install the UEFI (a successor to BIOS) for QEMU.

Download the .rpm file that contains `*ovmf-x64*` from the following page:
https://www.kraxel.org/repos/jenkins/edk2/

Install rpm2cpio and extract it to your root directory:

```
sudo apt install rpm2cpio
cd /
rpm2cpio /rust/storage/hackintosh/edk2.git-ovmf-x64-0-20171030.b3082.g710d9e69fa.noarch.rpm  | sudo cpio -idmv
```

##Creating a virtual disk for installation

We need to use qemu-img to create a disk file to install macOS to. Change 90G, to however big or small you like.

```
cd /where/you/want/the/disk/to/be
qemu-img create -f qcow2 macoshd.img 90G
```


##Configuring libvirt
First add yourself as a user of libvirt:
```
sudo usermod -a -G libvirt username_here
```
Libvirt can accept the configurations of virtual machines using xml fiiles.

Download the xml file from the git directory:



