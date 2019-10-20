# macos-kvm-pci-passthrough
A guide to macOS virtualization on Ubuntu Server 17.10 & 18.04 + Debian 10, done completely through the command line, no GUI required.

Warning: There are a few annoying bugs with macOS virtualization and I wouldn't recommend the VM as a desktop replacement. For example, the mouse cursor jumps around when hovering over hyperlinks. Dropdown menus sometimes appear in the bottom left corner. iMovie crashes regularly when importing videos into the timeline. Preview has bugs when using the magnifier. Airplay audio has synchronization bugs with video. The volume control in the menubar keeps glitching. With all this said, however, it is extremely useful as a server, so I recommend the following tutorial to those who want it simply as a VM.

Preface: I wanted to run macOS on my workstation, since macOS is a more friendly OS than Linux. However, I still needed to run a Linux Server, mainly to manage my ZFS harddrive array. Virtualizing Linux on a macOS host, and then passing the VM the harddisks may potentially wreak havoc on the ZFS array. I ended up having to use Linux as the host. Thankfully, this also means we don't have to deal with the problems that Hackintosh users must endure.

Virtualization technology has matured a lot in the past few years. The two biggest features are KVM (Kernel-based Virtual Machine) and PCIe-Passthrough. KVM allows near-native usage of the CPU, while PCIe-Passthrough allows *native* usage of the PCI device by the guest. If you passthrough a graphics card, it will even allow you to do gaming, HDMI/DisplayPort audio, etc at full speed. Furthermore, you can even pass through ethernet cards and USB controllers.

## Prerequisites
You will need a Mac in order to download and create an install image.
You should also use a Mac if you are using Clover Configurator to edit the Clover config.

You need a CPU that supports both KVM and IOMMU. Check https://wiki.archlinux.org/index.php/PCI_passthrough_via_OVMF#Prerequisites

#### My System: 
```
Motherboard: AsRock Rack C236M WS
Chipset: C236
RAM: 8GB Crucial ECC @ 2133MHz
CPU: i3-6100
GPU: Intel HD 530 & AMD Radeon RX 560
Ethernet: Intel I210 and Intel I219-LM
SSD: Samsung SM951 NVMe
HDD: 2 x WD Red 3TB
Host OS: Ubuntu Server 18.04
```

There are two PCIe devices I wish to passthrough:

- Ethernet (Intel I219-LM)
There is poor support in macOS for the Intel I210, so I chose the Intel I219-LM. The I210 will be used for the host.

- Graphics Card (AMD Radeon RX 560)
Allows me to run a display off of macOS, as well as accelerate the rendering of macOS desktop. Furthermore, the RX 560 works out of the box in macOS 10.14.


## Creating the install image
Let's begin with the step that requires a Mac.
* Download the High Sierra installer from the App Store.
* Create a bootable image using a USB Drive: https://support.apple.com/en-us/HT201372 (Use a 8GB flash drive if possible, as the install image we create will be the same size as your flash drive.)
  * A much quicker method is to use Disk Utility to create a blank sparse image file. Format it as HFS+. Then create the boot media using the sparse image. Once completed, continue with the `dd` command below.
* Now, convert the bootable USB Drive back into an `.img` file. Type `diskutil list` to find out the name of your flash drive, and replace `/dev/disk3` in the following command:
  ```
  bash -lic 'sudo dd bs=1m if=/dev/disk3 of=~/Desktop/10.15.0.img'
  ```
* Copy the bootable `.img` file to your Ubuntu Server.

## Installing QEMU
Install qemu (the hypervisor), libvirt (the VM daemon), virtinst (the VM manager) on Ubuntu:
  ```
  sudo apt-get install qemu-kvm libvirt-bin virtinst bridge-utils cpu-checker
  ```

## Enabling Kernel Support for Passthrough
These are the two kernel flags required: `intel_iommu=on` (allows PCIe passthrough), and `iommu=pt` (speeds up the PCIe passthrough, optional - remove if something doesn't work)
* Open the grub configuration:
  ```
  sudo nano /etc/default/grub
  ```
* Change the following line:
  ```
  GRUB_CMDLINE_LINUX_DEFAULT="intel_iommu=on iommu=pt"
  ```
* Update the host bootloader:
  ```
  sudo update-grub  
  ```
* Restart your system
  ```
  sudo reboot
  ```

## Creating the macOS Bootloader
* We need to install `libguestfs-tools` in order to create a Clover bootloader.
  ```
  sudo apt install libguestfs-tools
  ```
* Download the script for making the bootloader:
  ```
  wget https://github.com/yoonsikp/macos-kvm-pci-passthrough/raw/master/create-clover.sh
  ```
  
* Download the latest Clover Bootloader with filename CloverISO-XXXX.tar.lzma from the following webpage: 
  https://github.com/Dids/clover-builder/releases

* Extract it
  ```
  tar --lzma -xvf CloverISO-5093.tar.lzma
  ```

* Lastly, download `config.plist` from this repository. (Later, in order to get iMessage/iCloud working, we will have to edit this config.plist using Clover Configurator)
  ```
  wget https://raw.githubusercontent.com/yoonsikp/macos-kvm-pci-passthrough/master/config.plist
  ```

* Now we can run the script, which results in a file called `clover.raw` being created in your current directory:
  ```
  chmod 777 ./create-clover.sh
  sudo ./create-clover.sh --iso Clover-v2.5k-5093-X64.iso --img clover.raw --cfg config.plist
  ```

## Creating a virtual disk for installation
We need to use `qemu-img` to create a virtual disk to install macOS to.
* Run the command, and change `90G`, to however big or small you want your virtual machine's drive size to be.
  ```
  cd /where/you/want/the/disk/to/be
  qemu-img create -f qcow2 hd.qcow2 90G
  ```
  (`-f qcow2` compresses the disk image. If you wanted, you could always create a raw file using the option `-f raw` instead, and you would have a 90GB file on your disk. After that, don't forget to modify your macos.xml file by deleting the entire line that says `qcow2` in the macos.xml file.)

## Configuring the virtual machine
Download the `macos.xml` file from the git directory. This file defines the virtual machine.
Edit the file to fit your needs
#### RAM
`<memory unit='GB'>4</memory>`
#### CPU Cores
`<vcpu>2</vcpu>`
#### Disks and Install Media
Change all the file paths in the following section to match your system. Make sure to use full paths.
```
    <disk type='file' device='disk'>
      <source file='/rust/storage/hackintosh/clover.raw'/>
      <target dev='sda' bus='sata'/>
    </disk>
    <disk type='file' device='disk'>
      <driver name='qemu' type='qcow2' cache='none' io='native'/>
      <source file='/home/yoonsik/hd.qcow2'/>
      <target dev='sdb' bus='sata'/>
    </disk>
    <disk type='file' device='disk'>
      <source file='/rust/storage/hackintosh/10.15.0.img'/>
      <target dev='sdc' bus='sata'/>
    </disk>
```
Note that the Clover bootloader occupies the `sda` slot, i.e the first boot device.

Later, we will delete the lines for the `10.15.0.img` install media. 

Also, delete the line `<driver name='qemu' type='qcow2' cache='none' io='native'/>` if you used a `-f raw` image from earlier.

#### VNC
`<graphics type='vnc' port='-1' listen='0.0.0.0'/>`
For those who are connecting to this VM outside of their home network, you can change listen to '127.0.0.1' and use a SSH tunnel to connect to it.
* To create a SSH tunnel run:
  ```
  ssh -L 5900:127.0.0.1:5900 remote_server
  ```
* Point your own VNC client towards `localhost`.

## Configuring UEFI (OVMF)
Next we need to install the UEFI (a successor to BIOS) for QEMU.

* Simply download the two OVMF files from the repository and place them in the same folder as your VM. Then change the `macos.xml` file such that the following two paths point to the full paths of the corresponding files.

  ```
  <loader>OVMF_CODE.fd</loader>
          ^
          | 
          \--- Add the path here
  
  <nvram template='OVMF_VARS.fd'>/var/lib/libvirt/qemu/nvram/macos-test-org-base_VARS.fd</nvram>
                  ^
                  |
                  \--- And here
  ```
#### SKIP, NOT WORKING
Warning: Recent versions of this file have problems booting macOS ...
* Download the .rpm file that contains `*ovmf-x64*` from the following page: https://www.kraxel.org/repos/jenkins/edk2/
* Install `rpm2cpio` and extract the UEFI firmware to your root directory:
  ```
  sudo apt install rpm2cpio
  cd /
  rpm2cpio /rust/storage/hackintosh/edk2.git-ovmf-x64-0-20171030.b3082.g710d9e69fa.noarch.rpm  | sudo cpio -idmv
  ```

## Configuring libvirt
* First add yourself as a user of libvirt and/or kvm:
  ```
  sudo usermod -aG libvirt $USER
  sudo usermod -aG kvm $USER
  ```
* libvirt accepts the configurations of virtual machines using xml files.
  ```
  sudo virsh define macos.xml
  ```
* Next, we need to disable AppArmor, since it didn't seem to work with it enabled. (Warning, this may be insecure ...)
  ```
  sudo nano /etc/libvirt/qemu.conf
  ```
  Find the line `# security_driver = [...]`, uncomment it, and change it to `security_driver = "none"`.

## Connecting to the virtual machine

* Start the virtual machine:
  ```
  sudo virsh start macos
  ```
  
* If you get an error about how network `default` is not active, then run:
  ```
  sudo virsh net-start default
  ```
* Download a VNC viewer on another computer, such as RealVNC Viewer (https://www.realvnc.com/en/connect/download/viewer/) or gvncviewer, and connect to the server. (In order to fix the Left Command Key not working in RealVNC Viewer, go to `Preferences -> Expert -> LeftCmdKey` and set it to `Super_L`)

* Quickly press Esc to enter the setup screen. If you missed it you can stop the virtual machine and try again:
  ```
  sudo virsh destroy macos
  ```

* It's important to enter the setup screen so we can change the resolution of the UEFI to match that of macOS, since we are using a QEMU display. Go to `Device Manager -> OVMF Platform Configuration -> Change Preferred` and select `1024x768`.
* Hit `ESC`, `Y`, `ESC`. Finally, select `Reset`, otherwise the settings will not be applied to the boot. 

## Installing macOS

* Once the Clover bootloader is displayed, hit enter on the Install image. The Installer will take several minutes to boot up, and may look frozen most of the time. I would say give it ~10 minutes before giving up.
* Select your language, go to Disk Utility, and click "Show All Devices" in the View menu.
* Find your QEMU HARDDISK in the left, make sure it is the correct size (~90 GB), and click Erase. Name your drive `Macintosh HD`. Use the options `Mac OS Extended (Journaled)` and `GUID Partition Map`. 
* Quit Disk Utility and install macOS on Macintosh HD. It should reboot and boot to Macintosh HD, and finish the installation.

## Setting Up macOS for the first time and Networking (Important!)

Clover should automatically boot up macOS from now on. While setting up your macOS installation in the initial bootup, definitely do not login to iCloud/iMessage/iAnything yet. Logging in now may break things. Only set up user accounts, time zone, etc. While configuring the network, it may fail. That is fine. 

Once completed, we can fix the networking. macOS has a bug where it believes that the network cable is unplugged. Run the following set of commands in Ubuntu Server to fix it (on every boot!):
```
sudo virsh domif-setlink macos vnet0 down
sudo virsh domif-setlink macos vnet0 up
```
## Cleaning Up / Modifying the macos.xml configuration file
We are almost done.

* Shutdown the macOS machine safely
* Edit the macos.xml file and remove the following block to get rid of the installation media:
  ```
    <disk type='file' device='disk'>
      <source file='/rust/storage/hackintosh/10.15.0.img'/>
      <target dev='sdc' bus='sata'/>
    </disk>
  ```
* Finally, redefine the virtual machine:
  ```
  sudo virsh define macos.xml
  ```
## iCloud/iMessage
This section requires macOS.

* Download Clover Configurator, and open the `config.plist` file
* Edit it using the following methods: http://www.fitzweekly.com/2016/02/hackintosh-imessage-tutorial.html
* Copy the file back to your server. Go to the "Troubleshooting" section of this guide and recreate your Clover bootloader.
* Start the VM, open a Terminal window, and run `sudo nvram -c`, then `sudo reboot`.

## PCI-Passthrough for Networking
The networking bug above annoyed me so much, and because I was too lazy to set up tap networking, I ended up spending multiple hours setting up the PCI passthrough of one of my ethernet jacks :).

Follow this section of ensuring the PCI-passthrough groups are valid from ArchWiki: https://wiki.archlinux.org/index.php/PCI_passthrough_via_OVMF#Ensuring_that_the_groups_are_valid

After running the script, you should get something like this:
```
IOMMU Group 0 00:00.0 Host bridge [0600]: Intel Corporation Skylake Host Bridge/DRAM Registers [8086:190f] (rev 07)
IOMMU Group 10 03:00.0 Ethernet controller [0200]: Intel Corporation I210 Gigabit Network Connection [8086:1533] (rev 03)
...
IOMMU Group 8 00:1f.4 SMBus [0c05]: Intel Corporation Sunrise Point-H SMBus [8086:a123] (rev 31)
IOMMU Group 9 00:1f.6 Ethernet controller [0200]: Intel Corporation Ethernet Connection (2) I219-LM [8086:15b7] (rev 31)
```

I chose to passthrough the I219-LM ethernet controller, and thankfully there are no other devices in IOMMU Group 9.

We need to load the kernel modules/drivers that will attach to our PCI devices during the boot process. We will modify the kernel image that is loaded into the RAM on bootup. Note that `vfio-pci` is an alias for `vfio_pci`, and that `vfio_pci` depends on `vfio` and `vfio_virqfd`. 

```
# sudo nano /etc/initramfs-tools/modules 
vfio
vfio_iommu_type1
vfio_virqfd
vfio_pci ids=8086:15b7 disable_vga=1
```

Stop the host (Linux) from loading the ethernet driver. You can find the name of the currently loaded driver by running the command `lspci -v`. The filename should start with the name of the driver you want to blacklist.
```
# sudo nano /etc/modprobe.d/e1000e.conf
blacklist e1000e
```
Update your boot image
```
sudo depmod -ae
sudo update-initramfs -u
```

Now, within the macOS VM, install KextBeast, and then install the ethernet driver called `MausiEthernet.kext`. Shutdown the VM.

Add the ethernet PCIe device to the `macos.xml` file. You can find the PCIe address of the device by running `lspci -v`. The following is the XML definition: 

```
    <hostdev mode='subsystem' type='pci' managed='yes'>
      <source>
        <address domain='0x0000' bus='0x00' slot='0x1f' function='0x06'/>
      </source>
    </hostdev>
```
Replace the `<address>` with the proper PCIe device address. Multiple PCIe devices require multiple `<hostdev>` definitions.
Next, we can delete the whole `<interface type='network'>` block in `macos.xml` and redefine the VM.
```
sudo virsh define macos.xml
```
Lastly, start the VM. If any network interface names are numbered oddly on the macOS VM, open `/Library/Preferences/SystemConfiguration` in Finder and delete `preferences.plist` and `NetworkInterfaces.plist`. Reboot the VM.

## PCI-Passthrough for Graphics Card

Same as above, except we need to attach the `vfio_pci` driver to multiple PCI-e addresses. Below, I also passthrough my entire USB 3.0 controller! 

```
# sudo nano /etc/initramfs-tools/modules 
vfio
vfio_iommu_type1
vfio_virqfd
vfio_pci ids=8086:15b7,8086:a12f,1002:67ff,1002:aae0 disable_vga=1
```

```
# sudo nano /etc/modprobe.d/amdgpu.conf
blacklist amdgpu
```
```
sudo depmod -ae
sudo update-initramfs -u
```
For the graphics card definition, check the commented-out block in `macos.xml`.

Note, you cannot passthrough a `PCI bridge` to a VM.
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

Please star this guide if it is useful!

## Resources

https://www.contrib.andrew.cmu.edu/~somlo/OSXKVM/

https://www.kraxel.org/blog/2017/09/running-macos-as-guest-in-kvm/

## Installing QEMU for Debian 10:
``` 
sudo apt-get install qemu-kvm libvirt-clients virtinst bridge-utils libvirt-daemon libvirt-daemon-system
```
