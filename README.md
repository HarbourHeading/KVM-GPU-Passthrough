# Create KVM with Full GPU and CPU Passthrough

This guide explains how to set up a headless Windows 10/11 VM on QEMU/KVM with full GPU and CPU passthrough on an Ubuntu 24.04 server. I did not find many guides on the topic, so I wanted to share how I accomplished it in a straightforward way. Steps can be followed by Nvidia, AMD and Intel users. Although it has only been tested on Ubuntu Server 24.04, the steps should remain largely the same for older versions, e.g. 22.04 and 20.04.

**Note:** This guide isolates the GPU from the Ubuntu host for security reasons. The GPU isolation step can be skipped, however it is strongly recommended. Guide was originally made for Windows 10, but can be performed with minimal changes (difference being the iso) in Windows 11. For clarity, tools like spice viewer is referred to as a remote viewer, while apps like parsec and looking glass are referred to as remote desktops.

## Table of Contents
- [Prerequisites](#prerequisites)
   - [Is virtualization supported?](#is-virtualization-supported)
- [Installation](#installation)
   - [Install windows ISO](#install-windows-iso)
   - [Install virtio Drivers](#install-virtio-drivers)
   - [Install required packages](#install-required-packages)
- [Getting Started](#getting-started)
   - [Set up the GPU](#set-up-the-gpu)
   - [Isolate the GPU](#isolate-the-gpu)
   - [Launch and configure the KVM](#launch-and-configure-the-kvm)
   - [Install drivers in the KVM](#install-drivers-in-the-kvm)
   - [Set up remote desktop (optional)](#set-up-remote-desktop-optional)
- [Tips](#tips)
   - [Unmount installation disks](unmount-installation-disks)
   - [Set reboot signal to restart](set-reboot-signal-to-restart)
- [Troubleshooting](#troubleshooting)
- [Additional resources](#additional-resources)

## Prerequisites

### Is virtualization supported?
 
Run `lscpu | grep "Virtualization"` to check if virtualization is supported. You should see:

```bash
Virtualization: VT-x   # Intel
Virtualization: AMD-V  # AMD
```

Then, run `egrep -c '(vmx|svm)' /proc/cpuinfo`. If the output is greater than 0, virtualization is supported.

## Installation

### Install windows ISO

Download `Windows.iso` on an installation media (e.g. USB) from the [Windows 10 microsoft website](https://www.microsoft.com/en-us/software-download/windows10) or [Windows 11 microsoft website](https://www.microsoft.com/en-us/software-download/windows11). You can either copy the ISO to your server using scp or download it directly via CLI using [Fido](https://github.com/pbatard/Fido).

### Install virtio drivers  
  
Download the `virtio-win ISO` from the [virtio repository](https://github.com/virtio-win/virtio-win-pkg-scripts/blob/master/README.md). 

### Install required packages

Update your package database, install the required KVM packages then restart to apply changes:

```
sudo apt update
sudo apt install qemu-kvm virt-manager bridge-utils
sudo reboot now
```

Enable the libvirtd service and add your user to the libvirt group:

```
sudo systemctl enable libvirtd.service
sudo systemctl start libvirtd.service
sudo usermod -aG libvirt $USER
```

Relog for the group change to take effect.

## Getting started

### Set up the GPU

Use the applicable command to list your GPU and its associated devices with:

```bash
lspci -nn | grep -E "NVIDIA"
lspci -nn | grep -E "AMD"
lspci -nn | grep -E "INTEL"
```
Write down the ID numbers in square brackets at the end. Example output:
 
```
01:00.0 VGA compatible controller [0300]: NVIDIA Corporation TU106 [GeForce RTX 2070] [10de:1af2] (rev a1)

01:00.1 Audio device [0403]: NVIDIA Corporation TU106 High Definition Audio Controller [10de:1af9] (rev a1)

01:00.2 USB controller [0c03]: NVIDIA Corporation TU106 USB 3.1 Host Controller [10de:1ad8] (rev a1)

01:00.3 Serial bus controller [0c80]: NVIDIA Corporation TU106 USB Type-C UCSI Controller [10de:1af3] (rev a1)
```

In the example above, note down `[10de:1af2]`, `[10de:1af9]`, `[10de:1ad8]` and `[10de:1af3]`. Your IDs will be different.

Edit the GRUB configuration:

```
sudo vim /etc/default/grub
```

Update the GRUB_CMDLINE_LINUX_DEFAULT line to **include** (not replace):

```
GRUB_CMDLINE_LINUX_DEFAULT="intel_iommu=on iommu=pt vfio-pci.ids=10de:1af2,10de:1af9,10de:1ad8,10de:1af3"  # Intel
GRUB_CMDLINE_LINUX_DEFAULT="amd_iommu=on iommu=pt vfio-pci.ids=10de:1af2,10de:1af9,10de:1ad8,10de:1af3"    # AMD
```

Replace the ids with the ones tied to your GPU 2 steps previous.

Save and update the GRUB configuration:

```
sudo grub-mkconfig -o /boot/grub/grub.cfg
sudo reboot now
```

### Isolate the GPU

Edit the vfio.conf file:

```
sudo vim /etc/modprobe.d/vfio.conf
```

Add your GPU IDs:

```
options vfio-pci ids=10de:1af2,10de:1af9,10de:1ad8,10de:1af3
```

Update the initramfs:

```
sudo update-initramfs -c -k $(uname -r)
sudo reboot now
```

Verify the GPU isolation by running the applicable command:

```
lspci -k | grep -E "vfio-pci|NVIDIA"
lspci -k | grep -E "vfio-pci|AMD"
lspci -k | grep -E "vfio-pci|INTEL"
```

Kernel driver in use should now be listed as: `vfio-pci`. If not, you may need to blacklist the other kernel drivers.

### Launch and configure the KVM

We will setup and install the VM with [virt-install](https://manpages.org/virt-install). Tweak the example below as needed:
```
sudo virt-install \
--name=Windows10 \
--vcpus=8 \
--memory=8192 \
--os-variant=win10 \
--features kvm_hidden=on \
--machine q35 \
--cdrom=/home/user-1/kvm/Windows.iso \
--disk path=/var/lib/libvirt/images/win10.qcow2,size=100,bus=virtio,format=qcow2 \
--disk path=/home/user-1/kvm/virtio-win-0.1.262.iso,device=cdrom,bus=sata \
--graphics spice \
--video virtio \
--network network=default,model=virtio \
--hostdev pci_0000_01_00_0 \
--hostdev pci_0000_01_00_1 \
--hostdev pci_0000_01_00_2 \
--hostdev pci_0000_01_00_3 \
--wait -1
```

Use a remote [Spice viewer](https://www.spice-space.org/download.html) with the command:

```
spice://<server-ip>:5900
spice://localhost:5900    # If the server has a graphical interface (desktop), it can connect to the installer from the same machine
```

If you are making a remote connection, you will need to temporarily configure the KVM to listen for connections outside of localhost. run `virsh edit <guestname>` and change:

```
    <graphics type='spice' autoport='yes'>
      <listen type='address'/>
      <image compression='off'/>
    </graphics>
```

to (also updating `passwd` to a unique password that you'll enter when connecting remotely with spice viewer)

```
    <graphics type='spice' autoport='yes' listen='0.0.0.0' passwd='securePassword1!'>
      <listen type='address' address='0.0.0.0'/>
      <image compression='off'/>
    </graphics>
```

After the installation and another Remote Desktop has been set up, it can be changed back to not listen for connections outside of localhost. 

### Install drivers in the KVM

Open the drive with the virtio drivers and open the Guests tools installation (screenshot below) and follow the steps.

![install-virtio-tools](https://github.com/user-attachments/assets/2f55433f-9246-477f-85b3-346e7d840596)


Install your graphics card drivers: [Nvidia](https://www.nvidia.com/en-us/geforce/drivers/), [AMD](https://www.amd.com/en/support/download/drivers.html) or [Intel](https://www.intel.com/content/www/us/en/download-center/home.html).

Verify successful install by using task manager > performance and see your GPU listed. Alternatively check device manager.

### Set up remote desktop (optional)

If you are running a headless system and want low-latency, you can use a remote desktop viewer like [Moonlight QT](https://github.com/moonlight-stream/moonlight-qt), [Parsec](https://parsec.app/) or [Looking Glass](https://looking-glass.io/). Setup guides for each one will not be covered here, but can be found on their respective websites. Applications may need to be installed as machine instead of user to allow connections from the login screen.

Some of the options above provide fallback virtual displays, but if it does not work out of the box I recommend either buying a HDMI dummy plug or installing and setting up a virtual display driver. For installation instructions, refer to [Virtual Display Driver](https://github.com/itsmikethetech/Virtual-Display-Driver) for example.

### Tips

#### Unmount installation disks

After installation, you can unmount the Windows.iso and virtio.iso by editing the VM configuration with `virsh edit <guestname>` and removing:

```xml
    <disk type='file' device='cdrom'>
      <driver name='qemu' type='raw'/>
      <source file='/home/user-1/kvm/Windows.iso'/>
      <target dev='sda' bus='sata'/>
      <readonly/>
      <address type='drive' controller='0' bus='0' target='0' unit='0'/>
    </disk>
    <disk type='file' device='cdrom'>
      <driver name='qemu' type='raw'/>
      <source file='/home/user-1/kvm/virtio-win-0.1.262.iso'/>
      <target dev='sdb' bus='sata'/>
      <readonly/>
      <address type='drive' controller='0' bus='0' target='0' unit='1'/>
    </disk>
```

Do not delete the mounted main disk you'll be using for the VM, only the 2 mounted isos. If you do, mount it again with `virsh attach-disk <guestname> /dev/cdrom /media/cdrom`.

#### Set reboot signal to restart

By default, on restart/reboot of the VM it destroys itself (shuts down until powered up again). To change this, do:  

```
virsh edit <guestname>
```

Change:
 
```xml
<on_reboot>destroy</on_reboot>
```

 to
 
```xml
<on_reboot>restart</on_reboot>
```

#### Autostart VM on boot

By default, the VM does not start on host boot. To change this behaviour, do:

```
virsh autostart <guestname>
```

If you wish to disable this feature at any point, do:

```
virsh autostart --disable <guestname>
```

### Troubleshooting

**Q: VM not running?**<br>
A: Check if libvirtd is running: `sudo systemctl status libvirtd`. Look for any errors in logs if not.

**Q: Nvidia GPU not isolated?**<br>
A: Try adding `softdep nvidia pre: vfio-pci` under the options line in `/etc/modprobe.d/vfio.conf`. If this doesn't work for you either,
Read more on the [Arch wikis guide on GPU isolation](https://wiki.archlinux.org/title/PCI_passthrough_via_OVMF#Isolating_the_GPU).

**Q: Windows VM only showing 1 logical processor being used?**<br>
A: You may need to manually specify the cores in the VM configuration. To do so, do the following:
     Enter your server CLI. Turn off the VM before editing it with `virsh destroy <guestname>`.
     Then, edit it with `virsh edit <guestname>` to include:
     
```xml
  <vcpu placement='static'>8</vcpu>
  <cputune>
    <vcpupin vcpu='0' cpuset='0'/>
    <vcpupin vcpu='1' cpuset='1'/>
    <vcpupin vcpu='2' cpuset='2'/>
    <vcpupin vcpu='3' cpuset='3'/>
    <vcpupin vcpu='4' cpuset='4'/>
    <vcpupin vcpu='5' cpuset='5'/>
    <vcpupin vcpu='6' cpuset='6'/>
    <vcpupin vcpu='7' cpuset='7'/>
  </cputune>
  <cpu mode='host-passthrough' check='none' migratable='on'>
    <topology sockets='1' dies='1' cores='8' threads='1'/>
  </cpu>
```    
 
The above configuration specifies 8 cores. Change it as needed.
 
**Q: How do I fix error: `error: Requested operation is not valid: PCI device 0000:01:00.0 is in use by driver QEMU, domain <guestname>`?**<br>
A: Another KVM is using the GPU already. verify currently running KVMs with `virsh list --all`. To shutdown a KVM, use `virsh destroy <guestname>` and to remove it entirely use `virsh undefine <guestname>`.

**Q: Why is the screen completely black when connecting with *x* remote viewer but not with *x* remote desktop, or vice versa?**<br>
A: Remote viewers (like spice viewer) may not recognise the virtual display, and therefore display a black screen instead. To circumvent this, when connecting to the kvm and seeing the black screen, switch to the project windows menu `Windows key + P` and use arrow key down and press enter until the display appears. On parsec for example, you may want to switch projection type to *second screen only* (see why on question below) even if *PC screen only* is required for spice viewer to display the screen, as spice may not recognise the virtual display driver.

**NOTE:** In windows 10, you can do this from the login screen. However - for completely unfathomable reasons - **changing projection is blocked from the login screen in windows 11**. Therefore, you need to press Enter, type your password and then assume you've logged in and press `Windows key + P` a bunch, or through exact arrow + enter key presses.

**Q: Even with the virtual display installed, the remote desktop refresh rate only appears to be 1 hz?**"<br>
A: You may not have set the projection `Windows key + P` to *second screen only*, making the remote desktop use the basic display instead (default software display). In parsec, even if you have specified another encoding type (e.g. hardware encoding), you get a warning of "host is using software encoding" when the basic display is used.

**Q: On the guest VM it shows I have available disk space, but it goes into pause state and shows full disk usage on the host?**<br>
A: When files are written to and then deleted in the virtual disk, the space does not update on the host. You may need to use the command `virt-sparsify --in-place Windows10.qcow2`. Read more about it [here](https://balau82.wordpress.com/2011/05/08/qemu-raw-images-real-size/).

**Q: VM runs super slow and shows high CPU usage (~70%) as if it only had 1 core?**<br>
A: I found when I turned on **Core isolation -> memory integrity**, the windows 11 VM ran incredibly slow - as if it only had a single core. Turning it off made it go from ~70% average CPU usage to ~2% average CPU usage while idle. A [VMWare article on windows VM performance](https://www.switchfirewall.com/2025/03/vmware-performance-issues-on-windows11.html) mentions this as a valid solution and seems to be a common issue across windows virtualisation.

## Additional resources

- [Libvirt documentation](https://ubuntu.com/server/docs/libvirt)
- [Looking Glass installation](https://looking-glass.io/docs/B6/install/)
- [Arch Wiki: PCI passthrough via OVMF](https://wiki.archlinux.org/title/PCI_passthrough_via_OVMF)
- [KVM installation with Ubuntu 22.04 GUI](https://www.youtube.com/watch?v=vyLNpPY-Je0)

## Contributions

FedericoSlongo (fix broken links).
