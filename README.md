## This document is not finished, please do not use it yet.

# Single GPU passthrough to a virtual machine

This guide will guide you through setting up a "Gaming VM" without the need for a second GPU.\
Most traditional VM setups require a second graphics card which is isolated at boot. However, it is possible to detach (and reattach, of course) the GPU on a running system and pass it to a virtual machine.

Most information in this guide comes from [Muta's video](https://www.youtube.com/watch?v=BUSrdUoedTo), the Arch Wiki and my personal experience.\
You should have a decent understanding of Linux and virtual machines, and this guide will be targeted at Arch Linux (I use arch btw), but it will most likely also work on other distros.\
I'll try to keep it as simple and easy to follow as possible, and guide you through everything from installing libvirt to bypassing anticheat VM detection. The only game that will probably not work with this method is Valorant, because their ~~rootkit~~ anticheat is a shitty piece of garbage. Fuck you Riot Games.

## Warning!
This guide requires you to run terminal commands. **NEVER** blindly copy/paste commands from the internet, especially when running as root (Which most of these commands will require). Always try to understand what's happening!\
While I am obviously not trying to harm your soft- or hardware, don't hold me accountable if anything goes wrong. You've been warned!

## Hardware requirements
Basically, your processor and mainboard must support virtualization and IOMMU, which applies to almost everything released in the last 10 years. Better CPU, better performance. The same goes for the graphics card, basically any GPU supported by Windows should work.

The limiting factor will most likely be the RAM. You should be fine with 16GB. If you have less than that, you might be able to follow along but you will most likely run into issues.

Another factor to consider is storage space. Since Windows itself is a super bloated operating system and you will likely use it to install and play games, you should have at least 100GB of storage space to spare.

## Enabling IOMMU
[IOMMU](https://en.wikipedia.org/wiki/Input%E2%80%93output_memory_management_unit) groups are basically (to my understanding) groups of physical PCI devices, which can be "redirected" to a virtual machine. One group can contain multiple devices (for example the video and audio components of a grapics card are most likely seperate devices).

To enable IOMMU, you need to add a parameter in your boot loader. As this is most likely GRUB, that is what I'll be showing here. If you use systemd-boot or another bootloader, check [this](https://wiki.archlinux.org/title/Kernel_parameters#systemd-boot).

Edit the file `/etc/default/grub`, and find the line starting with `GRUB_CMDLINE_LINUX_DEFAULT=""`. There's probably already a few parameters in the quotes, so add the following to the end:\
If you're using an Intel CPU, add `intel_iommu=on`.\
If you're using an AMD CPU, add `amd_iomu=on`.

Save the file, and regenerate the grub config file:
```bash
sudo grub-mkconfig -o /boot/grub/grub.cfg
```
Now reboot your system.

To verify that IOMMU grouping works, run the following:
```bash
sudo dmesg | grep -i -e DMAR -e IOMMU
```

If you see a bunch of lines like `pci 0000:00:01.0: Adding to iommu group 0`, everything is fine. If not, something went wrong. Check if your hardware supports IOMMU, maybe you need to enable it somewhere in the BIOS?

## Installing libvirt and virt-manager
First, if not already installed, you will need to install and set up libvirt and virt-manager.

```bash
sudo pacman -Syu libvirt qemu virt-manager edk2-ovmf ebtables dnsmasq
sudo systemctl enable --now libvirtd.service virtlogd.socket

# Start the default libvirt network
# If this throws an error, try again after rebooting your system
sudo virsh net-autostart default
sudo virsh net-start default

# Add yourself to the `libvirt` group, so virt-manager doesn't ask for your password every time
sudo usermod -aG libvirt $YOURUSERNAME
```

## Finding the IOMMU group(s) to pass
Now you need to decide which devices you want to pass. Apart from the GPU you most likely want to pass your audio chip and USB controllers (Passing individual USB devices might cause issues because you're not able to interact with the host).

Download [list_iommu.sh](https://github.com/janderedev/single-gpu-passthrough/blob/master/list_iommu.sh) (taken from the [Arch Wiki](https://wiki.archlinux.org/title/PCI_passthrough_via_OVMF#Ensuring_that_the_groups_are_valid)), make it executable (`sudo chmod +x ./list_iommu.sh`) and run it.\
It will print out all IOMMU groups and the devices in them.

Example output:
```bash
IOMMU Group 0:
        00:01.0 Host bridge [0600]: Advanced Micro Devices, Inc. [AMD] Starship/Matisse PCIe Dummy Host Bridge [1022:1482]
IOMMU Group 1:
        00:01.2 PCI bridge [0604]: Advanced Micro Devices, Inc. [AMD] Starship/Matisse GPP Bridge [1022:1483]
[...]
# USB controller, keep in mind that there's probably more than one of these
IOMMU Group 20:
        21:08.0 PCI bridge [0604]: Advanced Micro Devices, Inc. [AMD] Matisse PCIe GPP Bridge [1022:57a4]
        2a:00.0 Non-Essential Instrumentation [1300]: Advanced Micro Devices, Inc. [AMD] Starship/Matisse Reserved SPP [1022:1485]
        2a:00.1 USB controller [0c03]: Advanced Micro Devices, Inc. [AMD] Matisse USB 3.0 Host Controller [1022:149c]
        2a:00.3 USB controller [0c03]: Advanced Micro Devices, Inc. [AMD] Matisse USB 3.0 Host Controller [1022:149c]
[...]
# GPU
IOMMU Group 27:
        2d:00.0 VGA compatible controller [0300]: NVIDIA Corporation GP107 [GeForce GTX 1050 Ti] [10de:1c82] (rev a1)
        2d:00.1 Audio device [0403]: NVIDIA Corporation GP107GL High Definition Audio Controller [10de:0fb9] (rev a1)
[...]
# Audio chip
IOMMU Group 31:
        2f:00.4 Audio device [0403]: Advanced Micro Devices, Inc. [AMD] Starship/Matisse HD Audio Controller [1022:1487]
```

Find all the groups that contain the devices you want to pass, and copy/paste them into a text file for now, you will need them later. Do not pass network or Wi-Fi controllers, since networking will be handled by the host using NAT or bridged networking.

## Patching the VBIOS
From what I've heard, this step is only required if you're using an NVIDIA graphics card. If you're using an AMD card, skip ahead to the next step.

**Warning!** This will require you to dump your graphics card's VBIOS. Although unlikely, this contains the risk of permanently bricking your GPU! You've been warned, don't blame me if you break your hardware.\
(In case you run the same GPU as me, **AND ONLY THEN**, you can use my patched VBIOS which is found [here](https://files.janderedev.xyz/1050ti_10de_1c82_vbios_patched.bin). Check if the output of the previous command matches `NVIDIA Corporation GP107 [GeForce GTX 1050 Ti] [10de:1c82] (rev a1)`)

### Dumping the VBIOS
Dumping can either be done from Windows (GPU-Z should give you the option) or from a live Arch/Manjaro/Mint/whatever USB. **DO NOT TRY TO DO THIS WITH THE PROPRIETARY NVIDIA DRIVERS RUNNING, YOU'LL DAMAGE YOUR GPU!** You should probably use an Arch ISO without desktop environment.

Download the file from [here](https://www.techpowerup.com/download/nvidia-nvflash/) and extract it to your downloads folder.

Once you're booted into your arch ISO, mount your hard drive and navigate to said downloads folder.
Run `./nvflash --save vbios.bin` to dump the VBIOS to a file.

That's it, you can now go back to your normal OS again. Make sure to back the file up somewhere so you won't have to dump it again.

### Patching the dump
Now you'll need a hex editor, for example bless: `sudo pacman -Syu bless`. Open the dumped VBIOS file in it, press CRTL+F and search for the term `VIDEO` (Select "as Text" next to the search bar). You should find a passage like `....U...K7400.L.w.VIDEO....`. Select everything before, but not including the `U`, and just delete it.\
Yup, it's that easy.

Now save the file, and copy it to `/etc/libvirt/vbios_patched.bin`.

## Setting up the virtual machine, step 1
At this point you should have a Windows 10 ISO downloaded. If not, download it from [here](https://www.microsoft.com/software-download/windows10).

Open the "Virtual Machine Manager" application. First, navigate to Edit > Preferences and check "Enable XML editing".\
Next, click the "Create a new virtual machine" button in the top left.
Select "Local install media (ISO image or CDROM)". On the next screen click "Browse", then "Browse Local" and select your Windows 10 ISO.

After you click forward, you'll have the option to set your VM's RAM allocation. If you have 16GB, you should give around 8GB to the VM. If you have 32GB, you can give it around 20-24GB.\
Don't worry about setting the CPUs yet, we'll set this later.

On the next screen you'll create a hard disk for the VM. Here you have multiple options: Either you pass an entire partition from your hard drive (Best performance) or you use a normal hard disk file (Easiest). Either way, select "Select or create custom storage".\
If you choose to pass a partition, just type the block device into the text field (`/dev/sda4` for example). Just make sure to pass the correct partition or you'll end up nuking your data.\
If you choose to use a virtual hard disk file, click "Manage..." and click [the + button](https://ඞ.janderedev.xyz/sc/1626476789-nicememe.hl8fkywDb1ua5QbH45I.png). Configure it to your liking and make sure to select "raw" as format.

After you click forward, make sure to check "Customize configuration before install" and click "Finish". Leave the name as "win10".

### Configuring the VM hardware

Fow now, we won't pass PCI devices until Windows is installed.

In the "Overview" tab, select "UEFI x86_64" in the "Firmware" dropdown.

Next, switch to the "CPUs" tab, expand the "Topology" category and check "Manually set CPU topology".\
Set "Sockets" to 1, "Cores" to the amount of physical cores in your system and "Threads" to the amount of threads per physical core. For example, my CPU has 8 cores / 16 threads, so I'd need to set "Cores" to 8 and "Threads" to 2.

Leave the other options for now and hit "Begin Installation". Go through the normal Windows 10 installation procedure until you're at a usable desktop, and shut the VM down.
