# Introduction

This is a setup for dualbooting Windows 11 and encrypted Arch with secure boot.
I'll be using systemd-boot as a bootloader.

## References

- [Systemd-boot](https://wiki.archlinux.org/title/Systemd-boot)
- [dracut](https://wiki.archlinux.org/title/Dracut)
- [Secure boot](https://wiki.archlinux.org/title/Unified_Extensible_Firmware_Interface/Secure_Boot#Using_your_own_keys)
- [Secure ArchLinux Installation Tutorial part 1 - Base System](https://www.youtube.com/watch?v=4xeNL7nJLrM)

## Prerequisites

1. Create Windows 11 and Arch live USBs
2. Backup important data.

**Check this list before starting!**

- Your computer supports SecureBoot/UEFI
- Your computer allows for enrollment of your own secureboot keys
- Your computer does not have manufacturer's backdoors

If you are not interested in SecureBoot, you can just skip last section of this
document.

### Windows install

- Delete all partitions and create a new partition with space for windows (this
  will also create ESP and recovery partitions). I allocated half of the disk to
  windows.
- Disable fast start-up and hibernate (control panel > power options > choose
  what the power button does)
- reboot to check if windows is booting

## Arch install

1. disable secure boot and clear keys
2. _boot into live usb_
3. connect to internet

```sh
$ iwctl
$ station wlan0 get-networks
$ station wlan0 connect SSSID
```

4. ssh for easy copying and pasting

- get machines ip

```sh
$ ip a s
```

- check sshd

```sh
$ systemctl status sshd
```

- set password for root user (for ssh)

```sh
$ passwd
<enter password>
```

- ssh into the new machine

```sh
$ ssh root@<IP>
```

5. Instead of partitioning the fresh disk(like in secure arch), we want to
   create a partition from the remaining space. (confirm the disk with `lsblk`
   first)

```sh
$ cgdisk /dev/nvme0n1
```

Following the dual boot setup, the linux partitions will be `boot 512Mib` and
`root ` in that order.

```
[new]
> Enter (start from the free space)
> 512Mib 
> Enter
> boot
[new]
> Enter (start from free space)
> Enter
> 8308
> root
[write]
```

6. Encrypt `root` partition

```sh
$ cryptsetup luksFormat --type luks2 /dev/nvme0n1p6

$ cryptsetup open --perf-no_read_workqueue --perf-no_write_workqueue --persistent /dev/nvme0n1p6 cryptlvm


$ pvcreate /dev/mapper/cryptlvm
$ vgcreate vg /dev/mapper/cryptlvm
$ lvcreate -l 100%FREE vg -n root
```

7.  Format partitions

```sh
$ mkfs.ext4 /dev/nvme0n1p5
$ mkfs.ext4 /dev/vg/root
```

8. mount drives

```sh
$ mount /dev/vg/root /mnt
$ mkdir -p /mnt/boot
$ mount /dev/nvme0n1p5 /mnt/boot
$ mkdir -p /mnt/boot/efi
$ mount /dev/nvme0n1p1 /mnt/boot/efi
```

9. Modify pacman mirrorlist You can get the suitable mirrors for your region
   from [https://archlinux.org/mirrorlist/].

```sh
$ vim /etc/pacman.d/mirrorlist

## United Kingdom
Server = https://archlinux.uk.mirror.allworldit.com/archlinux/$repo/os/$arch Server = https://mirror.bytemark.co.uk/archlinux/$repo/os/$arch
Server = https://london.mirror.pkgbuild.com/$repo/os/$arch
Server = https://mirrors.gethosted.online/archlinux/$repo/os/$arch
Server = https://mirrors.melbourne.co.uk/archlinux/$repo/os/$arch
Server = https://mirror.infernocomms.net/archlinux/$repo/os/$arch
Server = https://www.mirrorservice.org/sites/ftp.archlinux.org/$repo/os/$arch
Server = https://mirror.netweaver.uk/archlinux/$repo/os/$arch
Server = https://lon.mirror.rackspace.com/archlinux/$repo/os/$arch
Server = https://repo.slithery.uk/$repo/os/$arch
Server = https://mirrors.ukfast.co.uk/sites/archlinux.org/$repo/os/$arch
Server = https://mirror.cov.ukservers.com/archlinux/$repo/os/$arch
Server = https://mirror.vinehost.net/archlinux/$repo/os/$arch
```

10. Install arch

```sh
$ pacstrap /mnt linux linux-firmware base base-devel UCODE_PACKAGE sudo vim lvm2 dracut sbsigntools git efibootmgr binutils networkmanager
```

> NOTE: If using older arch ISO, might get an error like "signature from
> "maintainer is unknown trust''. In that case update the keyring with
> `pacman -Sy archlinux-keyring`

11. Generate fstab

```sh
$ genfstab -U /mnt >> /mnt/etc/fstab
```

12. `chroot` into installed system and basic configuration

```sh
$ arch-chroot /mnt
```

Set root passwd

```sh
$ passwd
```

Set timezone and generate /etc/adjtime:

```sh
$ ln -sf /usr/share/zoneinfo/<Region>/<city> /etc/localtime
hwclock --systohc
```

Set your desired locale:

```sh
$ vim /etc/locale.gen # uncomment locales you want
$ locale-gen

$ echo "LANG=en_GB.UTF-8" >> /etc/locale.conf
```

Set your hostname:

```sh
$ vim /etc/hostname
```

Configure your keyboard layout:

```sh
$ vim /etc/vconsole.conf

	KEYMAP=pl
	FONT=Lat2-Terminus16
	FONT_MAP=8859-2
```

Create your user:

```sh
useradd -m -G wheel YOUR_NAME
passwd YOUR_NAME
```

Add your user to sudo:

```sh
$ visudo

	%wheel	ALL=(ALL) ALL # Uncomment this line
```

13. Create dracut scripts that will hook into pacman

```sh
$ vim /usr/local/bin/dracut-install.sh

	#!/usr/bin/env bash
	mkdir -p /boot/efi/EFI/Linux
	while read -r line; do
		if [[ "$line" == 'usr/lib/modules/'+([^/])'/pkgbase' ]]; then
			kver="${line#'usr/lib/modules/'}"
			kver="${kver%'/pkgbase'}"
			dracut --force --uefi --kver "$kver" /boot/efi/EFI/Linux/arch-linux.efi
		fi
	done
```

_And the removal script_

```sh
$ vim /usr/local/bin/dracut-remove.sh

	#!/usr/bin/env bash
 	rm -f /boot/efi/EFI/Linux/arch-linux.efi
```

Make those scripts executable and create pacman's hook directory:

```sh
$ chmod +x /usr/local/bin/dracut-*
```

14. Now the actual hooks, first for the install and upgrade

```sh
$ mkdir /etc/pacman.d/hooks`
$ vim /etc/pacman.d/hooks/90-dracut-install.hook

	[Trigger]
	Type = Path
	Operation = Install
	Operation = Upgrade
	Target = usr/lib/modules/*/pkgbase

	[Action]
	Description = Updating linux EFI image
	When = PostTransaction
	Exec = /usr/local/bin/dracut-install.sh
	Depends = dracut
	NeedsTargets
```

_And for removal_

```sh
$ vim /etc/pacman.d/hooks/60-dracut-remove.hook

	[Trigger]
	Type = Path
	Operation = Remove
	Target = usr/lib/modules/*/pkgbase

	[Action]
	Description = Removing linux EFI image
	When = PreTransaction
	Exec = /usr/local/bin/dracut-remove.sh
	NeedsTargets
```

Check UUID of your encrypted volume and write it to file you will edit next:

```sh
$ blkid -s UUID -o value /dev/nvme0n1p2 >> /etc/dracut.conf.d/cmdline.conf
```

Edit the file and fill with with kernel arguments:

```sh
$ vim /etc/dracut.conf.d/cmdline.conf

	kernel_cmdline="rd.luks.uuid=luks-YOUR_UUID rd.lvm.lv=vg/root root=/dev/mapper/vg-root rootfstype=ext4 rootflags=rw,relatime"
```

> NOTE: Since linux 6.5 `amd_pstate_epp` is used by default. If using a version
> less than that it is recommended to include `amd_pstate=passive` for better
> cpu freqency management. Check that your machine has CPPC enabled either in
> the BIOS or else by the vendor.

Create file with flags:

```sh
$ vim /etc/dracut.conf.d/flags.conf

	compress="zstd"
	hostonly="no"
```

15. Generate your image by re-installing `linux` package and making sure the
    hooks work properly:

```sh
$ pacman -S linux
```

You should have `arch-linux.efi` within your `/efi/EFI/Linux/`. Now you only
have to add UEFI boot entry and create an order of booting:

16. install systemd-boot

```sh
$ bootctl install
```

17. Enable networkmanager service

```sh
$ systemctl enable NetworkManager
```

18. Exit and unmount partitions

```sh
$ exit
$ umount -R /mnt
```

## Secure Boot

Enable Setup Mode for SecureBoot in your BIOS, and erase your existing keys (it
may spare you setting attributes for efi vars in OS). If your system does not
offer reverting to default keys (useful if you want to install windows later),
you should backup them, though this will not be described here.

19. Configuring SecureBoot is easy with sbctl:

```sh
$ pacman -S sbctl
```

Check your status, setup mode should be enabled (You can do that in BIOS):

```sh
$ sbctl status
	  Installed:      ✘ Sbctl is not installed
	  Setup Mode:     ✘ Enabled
	  Secure Boot:    ✘ Disabled
```

20. Create keys and sign binaries:

```sh
$ sbctl create-keys

$ sbctl sign -s /boot/efi/EFI/Linux/arch-linux.efi #it should be single file with name verying from kernel version
$ sbctl sign -s /usr/lib/systemd/boot/efi/systemd-bootx64.efi
```

21. Automate systemd-boot update.

The package
[systemd-boot-pacman-hook](https://aur.archlinux.org/packages/systemd-boot-pacman-hook/)
AUR adds a pacman hook which is executed every time
[systemd](https://archlinux.org/packages/?name=systemd) is upgraded.

Rather than installing _systemd-boot-pacman-hook_, you may prefer to manually
place the following file in `/etc/pacman.d/hooks/`:

```sh
/etc/pacman.d/hooks/95-systemd-boot.hook

[Trigger]
Type = Package
Operation = Upgrade
Target = systemd

[Action]
Description = Gracefully upgrading systemd-boot...
When = PostTransaction
Exec = /usr/bin/systemctl restart systemd-boot-update.service
```

22. Configure dracut to know where are signing keys:

```sh
$ vim /etc/dracut.conf.d/secureboot.conf
	uefi_secureboot_cert="/usr/share/secureboot/keys/db/db.pem"
	uefi_secureboot_key="/usr/share/secureboot/keys/db/db.key"
```

23. We also need to fix sbctl's pacman hook. Creating the following file will
    overshadow the real one:

```sh
$ vim /etc/pacman.d/hooks/zz-sbctl.hook

	[Trigger]
	Type = Path
	Operation = Install
	Operation = Upgrade
	Operation = Remove
	Target = boot/*
	Target = efi/*
	Target = usr/lib/modules/*/vmlinuz
	Target = usr/lib/initcpio/*
	Target = usr/lib/**/efi/*.efi*

	[Action]
	Description = Signing EFI binaries...
	When = PostTransaction
	Exec = /usr/bin/sbctl sign-all -g
```

24. Enroll previously generated keys (drop microsoft option if you don't want
    their keys):

```sh
$ sbctl enroll-keys -m
```

Reboot the system. Enable only UEFI boot in BIOS and set BIOS password so evil
maid won't simply turn off the setting. If everything went fine you should first
of all, boot into your system, and then verify with sbctl or bootctl:

```sh
sbctl status
  Installed:	✓ sbctl is installed
  Owner GUID:	YOUR_GUID
  Setup Mode:	✓ Disabled
  Secure Boot:	✓ Enabled
```
