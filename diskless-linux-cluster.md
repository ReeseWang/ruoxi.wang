# Step-by-step Guide to Building a Diskless Linux Cluster

## Intro

A diskless computer is a computer, as the name suggests, without disks. Typically it will grab the programs it needs to run over a network instead of a dedicated storage device. A diskless cluster consists of multiple diskless computers, often share the same hardware configuration, and a "disked" server computer who provide data needed by diskless ones over a network. 

In this guide, we are going to set up a diskless cluster where all computers have x86-64 architecture and run Arch Linux. All client computers share the same root file system but each client has its own version of `/etc` directory so that they can have different configurations: one can run in headless mode while another boots into a desktop environment, for example.

### Why Diskless Cluster?

As is said in [this webpage](https://web.mst.edu/~vojtat/pegasus/administration.htm), diskless clusters have these advantages:

* Reduced cost due to many disks no longer needed.
* Fewer disk failures because the number of disks is reduced.
* Less power consumption, thus less heat and noise.
* Configuration and management in a central place.

Let's say if you want to place several IoT devices across your apartment, they all need to run Linux, and their file systems can't fit into NOR flash chips. Providing each device an SD card or disk could be a pain in the butt:

* They cost money.
* They could fail, and replacing them costs more time and money.
* The write performance of cheap SD cards could be poor.
* SD cards are easy to lose.
* Writing the OS image into SD cards one by one is time-consuming.
* Syncing software versions across them is hard, especially when they are installed at different times, or some of them are put off-line for a period of time.
* Making the same configuration change for each one of them requires extensive scripting and costs time.
* Deploying programs requires uploading the programs first. Or you could write your program on the devices. But are you going to set up your development environment on every one of them?
* And so on.

So unless you need some 100MB/s R/W speed on your computers, diskless is a better option. Our diskless cluster share the same root file system to make them more identical, and batch processing becomes easy as the data are all on the same machine.

### How it Works?

After a client machine's BIOS or UEFI finished executing, it will load and execute a small piece of code  stored in the client machine's network card. This program is called the [Preboot eXecution Environment (PXE)](https://en.wikipedia.org/wiki/Preboot_Execution_Environment). PXE asks the server machine to allocate an IP address for it using the [DHCP](https://en.wikipedia.org/wiki/Dynamic_Host_Configuration_Protocol) protocol. The server offers the client an IP address, and give it a file path. PXE will then accept the offered IP, contact the server with [TFTP](https://en.wikipedia.org/wiki/Trivial_File_Transfer_Protocol) protocol and get a file using the file path.

The file is another larger piece of code called [boot loader](https://wiki.archlinux.org/index.php/Arch_boot_process#Boot_loader). You may have already known a boot loader called [GRUB](https://wiki.archlinux.org/index.php/GRUB). PXE executes the boot loader, the boot loader first gets its configuration file from the TFTP server. The configuration file stores the path of the [Linux kernel](https://wiki.archlinux.org/index.php/Arch_boot_process#Kernel) and [initramfs](https://wiki.archlinux.org/index.php/Arch_boot_process#initramfs). The boot loader then gets the kernel and initramfs through TFTP, loads them into RAM and executes the kernel with the command line options in the configuration file (imagine the kernel to be a command-line program you run in your terminal). 

The kernel is a much larger piece of code. It uses the initramfs as a temporary root file system and runs the `/init` program in it (as a process instead of letting it take over the execution). `/init` then mounts an [Network File System (NFS)](https://en.wikipedia.org/wiki/Network_File_System), provided by the server, to `/new_root` and mounts another NFS to `/new_root/etc`. Finally the kernel use `/new_root` as the root file system and run `/init` in it. `/init` will then start many background processes and show a login prompt to the user. This concludes the boot process of a diskless client machine.

## Let's Do It!

### Set up a DHCP/TFTP/DNS server with dnsmasq

Suppose the server machine has the static IP address 192.168.78.1/24 on its interface dedicated to this cluster. You may use a network manager to assign this static IP. Say if you use `systemd-networkd`, create `/etc/systemd/network/00-eth0.network` as follows should do the job:

```toml
[Match]
Name=eth0

[Link]
RequiredForOnline=no

[Network]
Address=192.168.78.1/24
ConfigureWithoutCarrier=yes
```

You must not have another DHCP server running in this network other than your server machine (or maybe you could, didn't try that though).

Install `dnsmasq` package and edit `/etc/dnsmasq.conf`:

```bash
listen-address=192.168.78.1
dhcp-range=192.168.78.50,192.168.78.150,12h

# Change 8.8.8.8 to DNS servers of your choice
dhcp-option=option:dns-server,192.168.78.1,8.8.8.8

# Identify client machines by their MAC addresses.
# Assign each client a fixed host name and IP, and tag it with 'diskless'
dhcp-host=00:e0:66:59:89:84,client0,192.168.78.50,set:diskless
dhcp-host=00:e0:66:59:88:8b,client1,192.168.78.51,set:diskless
dhcp-host=00:e0:66:59:8c:50,client2,192.168.78.52,set:diskless
dhcp-host=00:e0:66:59:89:72,client3,192.168.78.53,set:diskless
dhcp-host=00:e0:66:59:8c:8d,client4,192.168.78.54,set:diskless
dhcp-host=00:e0:66:59:8a:2d,client5,192.168.78.55,set:diskless

enable-tftp
# We will create this folder later
tftp-root=/srv/root/boot

# Boot loader file path relative to tftp-root.
# Only clients tagged 'diskless' will be told this path.
dhcp-boot=tag:diskless,pxelinux.0
```

Add these lines to `/etc/hosts` to save yourself from typing client IPs every time. These entries are also used by `dnsmasq` and thus will be known by clients.

```
192.168.78.50	client0
192.168.78.51	client1
192.168.78.52	client2
192.168.78.53	client3
192.168.78.54	client4
192.168.78.55	client5
```

Enable and start `dnsmasq.service`:

```
# systemctl enable --now dnsmasq
```

### Prepare the Root File System

On the server machine, create a directory as clients' root file system:

```
# mkdir /srv/root
```

Create another directory as a mount point to bind mount `/srv/root`. This is for stopping `pacstrap` and `arch-chroot` from complaining `/srv/root` is not a mount point:

```
# mkdir /mnt/root
# mount --bind /srv/root /mnt/root
```

Add the following line to `/etc/fstab` to make this mount persistent across server reboots:

```
/srv/root	/mnt/root	none	bind
```

Now infect this file system with Arch Linux:

```
# pacstrap /mnt/root base linux linux-firmware syslinux sudo vi vim etckeeper
```

You can replace `vi` and `vim` with your favorite editor and add other packages such as `intel-ucode`. Our client system is still configurable after this so you don't have to come up with a comprehensive package list here.

Change root into the new system:

```
# arch-chroot /mnt/root
```

Now set the time zone, localization, root password and other things described in [Arch Wiki](https://wiki.archlinux.org/index.php/Installation_guide#Configure_the_system). But don't touch `/etc/hostname`, `/etc/hosts` and initramfs for the time being.

In the chroot environment, install `openssh` and enable `sshd.service` if you need it:

```
chroot# pacman -S openssh
chroot# systemctl enable sshd
```

Create a user with sudo permission, and set its password. You can make this user's GID and UID identical to the user you use on the server machine, for more convenience later on.

```
chroot# useradd --create-home --groups wheel --gid 1001 --uid 1001 wangruoxi
chroot# passwd wangruoxi
```

Edit `/etc/sudoers` with `visudo` command:

```
chroot# visudo
```

Create `(chroot)/etc/systemd/network/00-wired.network`:

```toml
openssh[Match]
Name=eth0

[Network]
DHCP=yes
KeepConfiguration=yes
IgnoreCarrierLoss=yes
```

Use `Ctrl+D` to exit the chroot environment.

Now we export this file system as an NFS. Make sure `nfs-utils` is installed and Edit `/etc/exports` to add the following line:

```
/srv/root    192.168.78.0/24(ro,no_root_squash,subtree_check)
```

Enable NFS server and re-export NFS shares:

```
# systemctl enable --now nfs-server
# exportfs -arv
```

### Prepare the files needed in the client boot process

#### Boot loader executables

We installed `syslinux` in the `pacstrap` step because we need a boot loader provided by it. The boot loader is `pxelinux`. Copy some files used by `pxelinux` to `/srv/root/boot` folder as symlinks. Thes files will be transferred to client machines via TFTP during their boot process. We use symlinks here so that these files will be automatically updated whenever `syslinux` gets an update. These symlinks have relative target paths in case `/srv/root` need to move to another location.

```
# cd /srv/root/boot
# cp -s ../usr/lib/syslinux/bios/{pxelinux.0,ldlinux.c32} ./
```

#### Boot loader configuration file

Create `/srv/root/boot/pxelinux.cfg/default`:

```
DEFAULT linux

LABEL linux
	KERNEL vmlinuz-linux
	APPEND root=/dev/nfs nfsroot=192.168.78.1:/srv/root ip=dhcp ro elevator=deadline rootwait
	INITRD intel-ucode.img,initramfs-linux-fallback.img
```

You can choose whether to use `intel-ucode.img`. You can also use `amd-ucode.img` according to client machines' CPU manufacturer. Just make sure the related package is installed and the file exists in `/srv/root/boot`. 

We use `initramfs-linux-fallback.img` for now because the non-fall-back initramfs generated by the server machine may not have kernel modules and firmwares required by client machines' network card.

#### initramfs

By default, Arch Linux initramfs don't have functionalities to mount NFS. So we need 3 packages from AUR to add this functionality.

Fetch PKGBUILDs of `mkinitcpio-nfs4-hooks`, `mkinitcpio-overlayfs` and `mkinitcpio-etc` from AUR, build packages and install them to clients' root file system. Just copy this 'for' loop to your terminal and hit Enter. Error messages from `pacman` hooks are expected since we aren't installing packages to the running system.

```bash
#!/bin/bash

for pkg in mkinitcpio-nfs4-hooks mkinitcpio-overlayfs mkinitcpio-etc; do
	git clone https://aur.archlinux.org/$pkg.git
	cd $pkg
	makepkg
	sudo pacman --root /srv/root --dbpath /srv/root/var/lib/pacman -U *.pkg.tar.xz
	cd ..
done
```

Edit `/srv/root/etc/mkinitcpio.conf` and modify the line begins with `HOOKS=`:

```bash
...
HOOKS=(base udev autodetect modconf net_nfs4 filesystems keyboard overlayfs etc)
...
```

Change root into the client root file system and update initramfs images:

```
# arch-chroot /srv/root mkinitcpio -p linux
```

