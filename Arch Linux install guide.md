____

## Installation steps

___

- ### Check internet connections

	```bash
	#
	ping -c 5 archlinux.org
	#
	```

___

- ### Partition disk

	```bash
	#
	gdisk /dev/sda
	#
	```

___

- ### Prepare encrypted partition

	```bash
	#
	# create encrypted partition
	cryptsetup --type luks2 -v --verify-passphrase --cipher aes-xts-plain64 --key-size 512 --key-slot 0 --hash sha256 --iter-time 11654 --pbkdf argon2id  --pbkdf-memory 524288 --pbkdf-parallel 4 --use-random --label xyz-locked --timeout 60 luksFormat /dev/sda3
	# open encrypted partition
	cryptsetup open /dev/sda3 xyz-unlocked
	#
	```

___

- ### Configure LVM

	```bash
	#
	pvcreate /dev/mapper/xyz-unlocked
	vgcreate vg /dev/mapper/xyz-unlocked
	lvcreate -L 30G vg --name root
	lvcreate -L 4G vg --name swap
	lvcreate -L 40G vg --name home
	#
	```

___

- ### Prepare file systems

	```bash
	#
   # dir for chroot
   mkdir -p /target
   # root partition
   mkfs.ext4 -L RootFs /dev/mapper/vg-root
   mount /dev/mapper/vg-root /target
   # boot partition
   mkdir -p /target/boot
   mkfs.ext2 -L BootFs /dev/sda2
   mount /dev/sda2 /target/boot
   # efi partition
   mkdir -p /target/boot/efi
   mkfs.vfat -F 32 -n EFI /dev/sda1
   mount /dev/sda1 /target/boot/efi
   # home partition
   mkdir -p /target/home
   mkfs.ext4 -L HomeFs /dev/mapper/vg-home
   mount /dev/mapper/vg-home /target/home
   # swap partition
   mkswap -L SWAP /dev/mapper/vg-swap
   swapon /dev/mapper/vg-swap
	#
	```

___

- ### Configure mirrorlist

	```bash
	#
	# Backup existing configuration
	cp /etc/pacman.d/mirrorlist /etc/pacman.d/mirrorlist.bak
	# finding mirrors around PL
	sudo reflector --protocol http,https --country "PL,DE,CZ,SK,FR" --latest 20 --sort rate --save /etc/pacman.d/mirrorlist
	#
	```

___

- ## Install base system

	```bash
	#
	pacstrap -K /target base base-devel linux-lts linux-lts-headers linux-firmware bash bash-completion sudo man-db man-pages reflector vi neovim
	#
	```
	
___

- ## Configure fstab

	```bash
	#
	genfstab -pU /target >> /target/etc/fstab
	#
	```


___

- ## Chroot

	```bash
	#
	arch-chroot /target
	export PS1="|chroot| ${PS1}"
	ln -sf /usr/bin/nvim /usr/bin/vim
	#
	```

___

- ## Configure timezone

	```bash
	#
	ln -sf /usr/share/zoneinfo/Europe/Warsaw /etc/localtime
	hwclock --systohc
	#
	```

___

- ## Configure language and keyboard map

	```bash
	# uncoment needed locale in /etc/locale.gen
	vim /etc/locale.gen
	locale-gen
	# select locale
	LANG="en_US.UTF-8"
	touch /etc/locale.conf
	vim /etc/locale.conf
	export LANG="en_US.UTF-8"
	# Configure vconsole.conf
	#
	pacman -S terminus fontconfig
	fc-cache
	# CONFIG
	#
	KEYMAP=us
	FONT=ter-v24b 
	#
	vim /etc/vconsole.conf
	#
	```

___

- ## Hostname and Host configuration

	```bash
	#
	vim /etc/hostname
	vim /etc/hosts
	#
	```

___

- ## Configure users

	```bash
	# create new user 
	useradd -m -G users,wheel -s /bin/bash -c "Janush" janush
	# set password for new user
	passwd janush
	# lock root account
	passwd -l root
	# enable wheel group in sudoers file
	visudo
	#
	```

___

- ## Configure initramfs

	```bash
	# install needed packages
	pacman -S cryptsetup lvm2 device-mapper dhcpcd
	# configure required things in /etc/mkinitcpio.conf
	vim /etc/mkinitcpio.conf
	#
	HOOKS=(base udev autodetect microcode modconf kms keyboard keymap consolefont block encrypt lvm2 filesystems fsck)
	#
	mkinitcpio -P
	#
	```


___

- ## Bootloader

	```bash
	# Install package
	pacman -S grub efibootmgr
	#
	# configure /etc/default/grub
	blkid -s UUID -o value /dev/sda3
	# LINE TO change: 
	GRUB_CMDLINE_LINUX_DEFAULT="cryptdevice=UUID=643ef53c-361e-4813-8521-6b7cbd49781a:xyz-unlocked:allow-discards root=/dev/mapper/vg0-root loglevel=3 quiet"
	vim /etc/default/grub
	# update grub config
	grub-mkconfig -o /boot/grub/grub.cfg
	#
	```

___

- ## Finish things

	```bash
	# exit chroot
	exit
	umount -R /target
	swapoff /dev/mapper/vg-swap
	vgchange -a n vg
	cryptsetup close xyz-unlocked
	poweroff
	
	```
___ 
## Reference

- a) [https://wiki.archlinux.org/title/Installation_guide](https://wiki.archlinux.org/title/Installation_guide)
- b) [https://gist.github.com/mjkstra/96ce7a5689d753e7a6bdd92cdc169bae](https://gist.github.com/mjkstra/96ce7a5689d753e7a6bdd92cdc169bae)
___