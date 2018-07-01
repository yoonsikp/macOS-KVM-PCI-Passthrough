# macos-kvm-pci-passthrough
A guide to macOS virtualization on Ubuntu Server 17.10, without needing to start a GUI/desktop on the server.

Warning: After using it for about two weeks, I realize there are bugs that are quite annoying, and I cannot recommend the following as a desktop replacement. For example, the mouse cursor jumps around when hovering over hyperlinks. Dropdown menus sometimes appear in the bottom left corner. iMovie crashes regularly when importing videos into the timeline. Preview has bugs when using the magnifier. Airplay audio has synchrnoization bugs with video. The volume control in the menubar keeps glitching. With all this said, however, it is extremely useful as a server, for example building Xcode projects. So I recommend the following tutorial to those who want to run a macOS VM, but do not necessarily want to use it is a daily driver.

Preface: I wanted to run macOS on my workstation, since macOS is a more friendly OS (although proprietary) than Linux. However, I still wanted to run a Linux Server, mainly to manage my ZFS harddrive array. Virtualizing Linux on a macOS host, and then passing the VM the harddisks may potentially wreak havoc on the ZFS array. I ended up having to use Linux as the host. Thankfully, this also means we also don't have to deal with the problems that Hackintosh users must endure.

Virtualization technology has matured a lot in the past few years. The two biggest features are KVM (Kernel-based Virtual Machine) and PCIe-Passthrough. KVM allows near-native usage of the CPU, while PCIe-Passthrough allows *native* usage of the PCI device by the guest. If you passthrough a graphics card, it will even allow you to do gaming, HDMI/DisplayPort audio, etc at full speed. Furthermore, this features allows you to pass through ethernet cards and USB controllers.

## Don't Forget
Don't forget to edit osk.cfg to change the secret key. I can't post it on Github for legal reasons.

## Prerequisites
You will need a Mac in order to download and create the install image.
You should probably also use the Mac if you are using Clover configurator.

You need a CPU that supports both KVM and IOMMU. Check https://wiki.archlinux.org/index.php/PCI_passthrough_via_OVMF#Prerequisites

#### My system: 
```
Motherboard: AsRock Rack C236M WS
Chipset: C236
RAM: 8GB Crucial ECC @ 2133MHz
CPU: i3-6100
GPU: Intel HD 530 & AMD Radeon RX 560
Ethernet: Intel I210 and Intel I219-LM
SSD: Samsung SM951 NVMe
HDD: 2 x WD Red 3TB
Host OS: Ubuntu Server 17.10
```

There are two PCIe devices I wish to passthrough:

-Ethernet (Intel I219-LM)
There is poor support in macOS/Hackintosh community for the Intel I210, so I chose the other ethernet port. The I210 will be used for Ubuntu

-Graphics Card (AMD Radeon RX 560)
Allows me to run a display off of macOS, as well as accelerate the rendering of macOS desktop. Furthermore, the RX 560 works out of box in macOS.


## Creating the install image
Let's begin with the step that requires a Mac.
Download the High Sierra install from the App Store.
Click here to help create the boot image on a USB Drive: https://support.apple.com/en-us/HT201372
Use an 8GB flash drive if possible, since the install image we create will be the size of your flash drive.

Now we need to convert the USB Drive back into an `.img` file.
Type `diskutil list` to find out the name of your flash drive, and replace it in the following command:
```
sudo dd bs=1m if=/dev/disk3 of=~/Desktop/10.13.1.img
```
Copy over the `.img` file to your Ubuntu Server.


## Installing QEMU
First we need to install qemu (the emulator), libvirt (the VM daemon), virtinst (the manager) on Ubuntu:
```sudo apt install qemu 
sudo apt-get install qemu-kvm libvirt-bin virtinst bridge-utils cpu-checker
```

## Enabling Passthrough
```
sudo nano /etc/default/grub
```
```
GRUB_CMDLINE_LINUX_DEFAULT="intel_iommu=on iommu=pt"
```
```
sudo update-grub  
```

## Creating the Bootloader
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


## Configuring UEFI (OVMF)


Next we need to install the UEFI (a successor to BIOS) for QEMU.

Download the .rpm file that contains `*ovmf-x64*` from the following page:
https://www.kraxel.org/repos/jenkins/edk2/

Install rpm2cpio and extract it to your root directory:

```
sudo apt install rpm2cpio
cd /
rpm2cpio /rust/storage/hackintosh/edk2.git-ovmf-x64-0-20171030.b3082.g710d9e69fa.noarch.rpm  | sudo cpio -idmv
```

## Creating a virtual disk for installation

We need to use qemu-img to create a disk file to install macOS to. Change 90G, to however big or small you want your virtual machine's drive size to be.

```
cd /where/you/want/the/disk/to/be
qemu-img create -f qcow2 macoshd.img 90G
```
(`-f qcow2` compresses the disk image. If you wanted, you could always create a raw file using the option `-f raw` instead, and you would have a 90GB file on your disk. After that, don't forget to modify your macos.xml file. Delete the whole line that says `qcow2` in the macos.xml file.)

## Configuring the virtual machine

Download the macos.xml file from the git directory.

Edit the macos.xml to fit your needs
#### RAM
`<memory unit='GB'>4</memory>`
#### CPU Cores
` <vcpu>2</vcpu>`
#### Disks and Install Media
Change all the file paths in the following section to match your system. Make sure to use full paths.
```
    <disk type='file' device='disk'>
      <source file='/rust/storage/hackintosh/clover.raw'/>
      <target dev='sda' bus='sata'/>
    </disk>
    <disk type='file' device='disk'>
      <driver name='qemu' type='qcow2' cache='none' io='native'/>
      <source file='/home/yoonsik/macoshd.img'/>
      <target dev='sdb' bus='sata'/>
    </disk>
    <disk type='file' device='disk'>
      <source file='/rust/storage/hackintosh/10.13.1.img'/>
      <target dev='sdc' bus='sata'/>
    </disk>
```
Note that the Clover bootloader occupies the `sda` slot, i.e the first boot device.

Later, we will delete the lines for the 10.13.1.img install media. 

Also, delete the line `<driver name='qemu' type='qcow2' cache='none' io='native'/>` if you used a `-f raw` image from earlier.

#### VNC
`<graphics type='vnc' port='-1' listen='0.0.0.0'/>`
For those who are connecting to this VM outside of their home network, you can change listen to '127.0.0.1' and use a SSH tunnel to connect to it.

## Configuring libvirt
First add yourself as a user of libvirt:
```
sudo usermod -a -G libvirt username_here
```
Libvirt can accept the configurations of virtual machines using xml files.

```
sudo virsh define macos.xml
```
Next, we need to disable AppArmor, since it didn't seem to work with it enabled. (this may be a security hole ...)
```
sudo nano /etc/libvirt/qemu.conf
```
Find the line `# security_driver = [...]`, uncomment it, and change it to `security_driver = "none"`.


## Connecting to the virtual machine
Start the virtual machine:
```
sudo virsh start macos
```

Download a VNC viewer on another computer, such as RealVNC Viewer (https://www.realvnc.com/en/connect/download/viewer/) and connect to the server. (In order to fix the Left Command Key not working in RealVNC Viewer, go to `Preferences -> Expert -> LeftCmdKey` and set it to `Super_L`)

Quickly press F2 (fn+F2 on Mac) to enter the setup screen. If you missed it you can stop the virtual machine and try again:

```
sudo virsh destroy macos
```
(Try not to use this after having booted into your installation of macOS, use the Shutdown inside the VM)

It's important to enter the setup screen so we can change the resolution of the UEFI to match that of macOS, since we are using a QEMU display. 

In the setup go to `Device Manager -> OVMF Platform Configuration -> Change Preferred` and select `1024x768`.
Hit `ESC`, `Y`, `ESC`. Finally, you must select `Reset`, or else the settings will not be applied to the next boot. 

## Installing macOS

Once the Clover bootloader is displayed, hit enter on the Install image.
The Installer will take several minutes to boot up, and will look frozen most of the time.
I would say give it at least 15 minutes before giving up.

Select your language, go to Disk Utility, and click "Show All Devices" in the View menu.
Find your QEMU HARDDISK in the left, make sure it is the correct size (~90 GB), and click Erase. Name your drive `Macintosh HD`. Use the options `Mac OS Extended (Journaled)` and `GUID Partition Map`. 

Quit Disk Utility and install macOS on Macintosh HD. It should reboot and boot to Macintosh HD, and finish the installation.

## Setting Up macOS for the first time and Networking (Important!)

Clover should automatically boot up macOS from now on. While setting up your macOS installation in the initial bootup, definitely do not login to iCloud/iMessage/iAnything yet. Logging in now may break things. Only set up user accounts, time zone, etc. While configuring the network, it may fail. That is fine. 

Once completed, we can fix the networking. macOS has a bug where it believes that the network cable is unplugged. Run the following set of commands in Ubuntu Server to fix it (on every boot!):
```
sudo virsh domif-setlink macos vnet0 down
sudo virsh domif-setlink macos vnet0 up
```

## Cleaning Up / Modifying the macos.xml configuration file

After shutting down the macOS machine safely, we can edit macos.xml to remove the following block to get rid of the installation media:
```
    <disk type='file' device='disk'>
      <source file='/rust/storage/hackintosh/10.13.1.img'/>
      <target dev='sdc' bus='sata'/>
    </disk>
```
Finally, run:
```
sudo virsh define macos.xml
```

## iCloud/iMessage
Using your mac , Download Clover Configurator , and open the config.plist from your server.
Edit it using the following methods
http://www.fitzweekly.com/2016/02/hackintosh-imessage-tutorial.html

Copy the file back to your server.
Edit the file and add -s

In the config.plist change
```
<key>Boot</key>
	<dict>
		<key>Arguments</key>
		<string></string>
```
to
```
<key>Boot</key>
	<dict>
		<key>Arguments</key>
		<string>-v -s</string>
```

At the prompt, type nvram -c, then halt.



## PCI-Passthrough for Networking
TBD

The networking bug annoyed me so much, and because I was too lazy to set up tap networking, I ended up spending multiple hours setting up the PCI passthrough of one of my ethernet jacks :)

```
sudo nano /etc/initramfs-tools/modules 
```
```
vfio
vfio_iommu_type1
vfio_virqfd
vfio_pci ids=8086:1533 disable_vga=1
```

```
sudo nano /etc/modprobe.d/e1000e.conf
```
```
blacklist e1000e
```
```
sudo depmod -ae
sudo update-initramfs -u
```

Delete the block

```<interface>```
Use kextbeast to install the MausiEthernet.kext
Then, delete library preferences en0
Reboot.


## PCI-Passthrough for Graphics Card

TBD
```
sudo nano /etc/initramfs-tools/modules 
```
```
vfio
vfio_iommu_type1
vfio_virqfd
vfio_pci ids=8086:15b7,8086:1901,1002:67ff,1002:aae0 disable_vga=1
```

```
sudo nano /etc/modprobe.d/amdgpu.conf
```
```
blacklist amdgpu
```
```
sudo depmod -ae
sudo update-initramfs -u
```

## Autostart the VM
Enable autostart
```
sudo virsh autostart macos
```
Disable autostart
```
sudo virsh autostart macos --disable
```
## Troubleshooting

A reminder that Clover frequently freezes, so you should always edit the config.plist file instead.

If your macOS install is hanging on boot:
In the config.plist change
```
<key>Boot</key>
	<dict>
		<key>Arguments</key>
		<string></string>
```
to
```
<key>Boot</key>
	<dict>
		<key>Arguments</key>
		<string>-v -s</string>
```
Delete clover.raw and recreate the bootloader
`sudo ./clover-image.sh --iso Clover-v2.4k-4289-X64.iso --img clover.raw --cfg config.plist`

If your mouse/keyboard isn't working:
Change ```<controller type='usb' model='ehci'/>``` to ```<controller type='usb' model='piix4-usb-uhci'/>```

If only your mouse isn't working:
Use ```<input type='mouse' bus='usb'/>``` instead of ```<input type='tablet' bus='usb'/> ```

If you ever need to delete your VM:
Type ```sudo virsh undefine --nvram macos```. Keep in mind if you add it back, you must go through the process of changing the resolution in the UEFI again.




