
- ##  Steps

___

1. #### Install required packages


	Update packages database:

	```bash
	# switch to root
	sudo -i
	# update package database
	apt update
	```

	Install packages:

	```bash
	apt install neovim gdisk debootstrap arch-install-scripts cryptsetup lvm2 dosfstools
	```

___

2. #### Partition disk

	```bash
	DISK="/dev/sda"
	gdisk "${DISK}"
	```

____

3. #### Prepare encrypted partition and lvm

   Encryption:

   ```bash
   CRYPT="/dev/sda3"
   LVM_NAME="xyz-locked"
  	cryptsetup --type luks2 -v --verify-passphrase --cipher aes-xts-plain64 --key-size 512 --key-slot 0 --hash sha256 --iter-time 11654 --pbkdf argon2id  --pbkdf-memory 524288 --pbkdf-parallel 4 --use-random --label "${LVM_NAME}" --timeout 60 luksFormat "${CRYPT}"
   ```

	Open encrypted container:

	``` bash
	CRYPT="/dev/sda3"
	CRYPT_NAME="xyz-unlocked"
	cryptsetup open "${CRYPT}" "${CRYPT_NAME}"
	```

	LVM:

	```bash
	CRYPT_NAME="xyz-unlocked"
	VG_NAME="vg"
	ROOT_LVM="root"
	SWAP_LVM="swap"
	HOME_LVM="home"
	pvcreate /dev/mapper/"${CRYPT_NAME}"
	vgcreate "${VG_NAME}" /dev/mapper/"${CRYPT_NAME}"
	lvcreate -L 30G "${VG_NAME}"  --name "${ROOT_LVM}"
	lvcreate -L 4G "${VG_NAME}"  --name "${SWAP_LVM}"
	lvcreate -L 30G "${VG_NAME}"  --name "${HOME_LVM}"
	```

___

4. #### Format and mount partitions

   ```bash
   #
   # dir for chroot
   mkdir -p /target
   #
   VG_NAME="vg"
   EFI="/dev/sda1"
   BOOT="/dev/sda2"
   ROOT_LVM="root"
   SWAP_LVM="swap"
   HOME_LVM="home"
   # root partition
   mkfs.ext4 -L RootFs /dev/mapper/"${VG_NAME}-${ROOT_LVM}"
   mount /dev/mapper/"${VG_NAME}-${ROOT_LVM}" /target
   # boot partition
   mkdir -p /target/boot
   mkfs.ext2 -L BootFs "${BOOT}"
   mount "${BOOT}" /target/boot
   # efi partition
   mkdir -p /target/boot/efi
   mkfs.vfat -F 32 -n EFI "${EFI}"
   mount "${EFI}" /target/boot/efi
   # home partition
   mkdir -p /target/home
   mkfs.ext4 -L HomeFs /dev/mapper/"${VG_NAME}-${HOME_LVM}"
   mount /dev/mapper/"${VG_NAME}-${HOME_LVM}" /target/home
   # swap partition
   mkswap -L SWAP /dev/mapper/"${VG_NAME}-${SWAP_LVM}"
   swapon /dev/mapper/"${VG_NAME}-${SWAP_LVM}"
   #
   ```

____

5. #### Install base system with deboostrap

	```bash
	debootstrap --verbose --arch=amd64 --include=ca-certificates,bash-completion trixie /target http://ftp.pl.debian.org/debian/
	```

____

6. #### Prepare apt sources file for new installed system

	```bash
	#
	vim /target/etc/apt/sources.list
	#
	# Content of file:
	########################################################################################################################
	##														      #
	## trixie													      #
	deb http://ftp.pl.debian.org/debian trixie main contrib non-free non-free-firmware				      #
	# deb-src http://ftp.pl.debian.org/debian trixie main contrib non-free non-free-firmware			      #
	#														      #	
	## trixie-backports												      #
	# deb http://ftp.pl.debian.org/debian trixie-backports main contrib non-free non-free-firmware			      #
	# deb http://ftp.pl.debian.org/debian trixie-backports main contrib non-free non-free-firmware			      #
	#														      #
	## trixie-security												      #
	deb https://security.debian.org/debian-security trixie-security main contrib non-free non-free-firmware		      #
	# deb https://security.debian.org/debian-security trixie-security main contrib non-free non-free-firmware 	      #
	#														      #
	## trixie-updates												      #
	deb http://ftp.pl.debian.org/debian trixie-updates main contrib non-free non-free-firmware			      #
	# deb-src http://ftp.pl.debian.org/debian trixie-updates main contrib non-free non-free-firmware		      #
	#														      #
	## trixie-proposed												      #
	# deb http://ftp.pl.debian.org/debian trixie-proposed main contrib non-free non-free-firmware			      #
	# deb-src http://ftp.pl.debian.org/debian trixie-proposed main contrib non-free non-free-firmware		      #
	##														      #
	#######################################################################################################################
	```

___

7. #### Chroot

	```bash
	# mounting additional filesystems
	mount --bind /dev /target/dev
	mount --bind /dev/pts /target/dev/pts
	mount --bind /proc /target/proc
	mount --bind /sys /target/sys
	mount --bind /run /target/run
	mount --bind /sys/firmware/efi/efivars /target/sys/firmware/efi/efivars
	# chroot
	chroot /target /bin/bash -l
	# commands in chroot
	export PS1="|chroot| ${PS1}"
	alias vim="vim.tiny"
	alias cls="clear"
	#
	```

___

8. #### Update new system

	```bash
	# update package database
	apt update
	apt modernize-sources
	#
	```

___

9. #### Configure locale,keyboard,timezone,language

	```bash
	# Install packages
	apt install tzdata locales keyboard-configuration console-setup fonts-terminus fonts-terminus-otb fbset
	# configure locales
	dpkg-reconfigure locales
	# configure keyboard
	# model -> Polish -> Right Alt (AltGr) -> Right Control
	dpkg-reconfigure keyboard-configuration
	# configure console-setup
	dpkg-reconfigure console-setup
	# configure timezone
	dpkg-reconfigure tzdata
	#
	source /etc/profile
	export PS1="|chroot| ${PS1}"
	alias vim="vim.tiny"
	alias cls="clear"
	#
	```

____

10. #### Kernel and required tools

	```bash
	# install kernel and firmware
	apt install linux-image-amd64 linux-headers-amd64 dkms firmware-linux firmware-linux-free firmware-linux-nonfree
	# install other packages needed for base system
	apt install initramfs-tools cryptsetup cryptsetup-initramfs lvm2 man-db manpages manpages-pl file locate
	
	```

____

11. #### Set hostname and configure hosts file

	```bash
	# set hostname:
	vim.tiny /etc/hostname
	# edit hosts file
	vim.tiny /etc/hosts
	# example of hosts
	127.0.0.1       localhost
	::1             localhost ip6-localhost ip6-loopback
	ff02::1         ip6-allnodes
	ff02::2         ip6-allrouters
	127.0.0.1       dawcek-vm.localdomian   dawcek-vm
	```

___

12. #### Configure user

	```bash
	# create new user 
	useradd -m -G users,sudo -s /bin/bash -c "Charlie" charlie
	# set password for new user
	passwd charlie
	# lock root account
	passwd -l root
	#
	```

____

13. #### Configure fstab

	```bash
	# comand for getting UUID of partition
	blkid -s UUID -o value $DEV
	#
	# File content:
	#################################################################################################################################
	##                                                                                                                              #
	#<file system>                                  <mount point>   <type>  <options>                       <dump>  <pass>          #
	## root LABEL=RootFs                                                                                                            #
	UUID=0875236a-0afe-4b46-8df3-d9eceae911b2       /               ext4    errors=remount-ro               0       1               #
	#                                                                                                                               #
	## boot LABEL=BootFs                                                                                                            #
	UUID=beb2e793-2e70-4fac-a898-fdbc0171913d       /boot           ext2    defaults                        0       2               #
	#                                                                                                                               #
	## efi LABEL=EFI                                                                                                                #
	UUID=7DA2-F47E                                  /boot/efi       vfat    umask=0077                      0       1               #
	#                                                                                                                               #
	## home LABEL=HomeFs                                                                                                            #
	UUID=797f3a26-9f24-4ae1-b7f0-0f4881c17b4e       /home           ext4    defaults                        0       2               #
	#                                                                                                                               #
	## swap LABEL=SWAP                                                                                                              #
	UUID=c026a602-bbf4-4fa1-9d15-4b353330c969       none            swap    sw                              0       0               #
	#                                                                                                                               #
	##                                                                                                                              #
	#################################################################################################################################
	```

____

14. #### configure cryptab

	```bash
	# get UUID for lukscrypt partition
	CRYPT_NAME="xyz-unlocked"
	CRYPT="/dev/sda3"
	CRYPT_UUID=$(blkid -s UUID -o value ${CRYPT})
	echo ${CRYPT_NAME} >> /etc/crypttab
	echo "UUID=${CRYPT_UUID}" >> /etc/crypttab
	vim /etc/crypttab
	#
	# Example crypttab file entry:
	#
	# <target name> <source device>                                 <key file>      <options>
	xyz-unlocked   UUID=7f1bcc62-a1c2-4567-93d5-d8183e2da7c1      none            luks,discard
	#
	# prepare resume file for initramfs (OPTIONAL)
	SWAP="/dev/mapper/vg-swap"
	SWAP_UUID=$(blkid -s UUID -o value $SWAP)
	echo "RESUME=UUID=${SWAP_UUID}" >> /etc/initramfs-tools/conf.d/resume
	vim /etc/initramfs-tools/conf.d/resume
	# Content
	RESUME=UUID=c026a602-bbf4-4fa1-9d15-4b353330c969
	#
	```

____

15. #### Configure network

	```bash
	# install network-manager
	apt install network-manager
	# check network interfaces
	ip addr
	# add following configuration to /etc/network/interfaces file
	vim /etc/network/interfaces
	#
	# Content to add:
	## loopback
	#
	auto lo
	iface lo inet loopback
	#
	## ens33
	auto ens33
	allow-hotplug ens33
	iface ens33 inet dhcp
	#
	```

____

16. #### Install grub bootloader

	```bash
	# install packages
	apt install grub-efi-amd64 efibootmgr os-prober
	# install grub bootloader
	grub-install --target=x86_64-efi --efi-directory=/boot/efi --bootloader-id=DCDebian
	# add resume=UUID=xxxx... to /etc/default/grub 
	# example:
	GRUB_CMDLINE_LINUX_DEFAULT="resume=UUID=c026a602-bbf4-4fa1-9d15-4b353330c969 quiet"
	# update grub config and recreate initramfs
	update-initramfs -ck all;update-grub2
	#
	```

____

17. #### finish thing

	```bash
	# exit from chroot
	exit
	# deactivate swap
	swapoff /dev/dm-2
	# umount all filesystems mounted at /target
	#
	umount /target/sys/firmware/efi/efivars
	umount /target/run
	umount /target/sys
	umount /target/proc
	umount /target/dev/pts
	umount /target/dev
	umount /target/home
	umount /target/boot/efi
	umount /target/boot
	umount /target
	vgchange -a n vg
	cryptsetup close xyz-unlocked
	reboot
	#
	```

____

18. #### Install basic packages after restart to newly installed system.

	```bash
	  apt install neovim htop tmux build-essential git gdb cmake diffutils wget curl rsync tcpdump nftables ethtool net-tools xclip xsel tree cryfs age mc command-not-found aptitude netselect-apt bind9-dnsutils lsof less logrotate texinfo lsb-release bc pwgen xkcdpass xxd trash-cli tasksel needrestart extrepo gdisk dosfstools mtools ntfs-3g btrfs-progs xfsprogs jfsutils exfatprogs f2fs-tools nfs-common squashfs-tools usbutils sysfsutils mdadm 7zip rar unrar zip unzip gzip bzip2 xz-utils zstd pigz intel-microcode iftop iotop wget2 fzf pv jq yq xq speedtest-cli links inetutils-tools btop bat glances duf ripgrep iwd wpasupplicant xdg-user-dirs xdg-utils  extra-xdg-menus extra-xdg-menus xdg-terminal-exec bubblewrap flatpak flatseal efitools smartmontools nvme-cli inxi hwinfo gvfs-backends mtp-tools smbclient valgrind wireguard wireguard-tools openvpn openconnect entr fsarchiver signify-openbsd doas tshark aria2 testdisk partclone partimage ngrep
	 # install standrad task with tasksel
	 tasksel install standard
	```

_____

19. #### Configure silent boot
	```bash
	# configure to /etc/default/grub
	GRUB_CMDLINE_LINUX_DEFAULT="quiet splash loglevel=3 systemd.show_status=auto rd.udev.log_level=3 vt.global_cursor_default=0"
	```

____

20. #### Install some GUI

	```bash
	# balanced kde plasma desktop
	apt install kde-standard
	# set default target to newly installed GUI enviroment
	systemctl set-default graphical.target
	#
	```

____


21. #### Configure static DNS servers

	```bash
	# add configuration to /etc/dhcpcd.conf
	vim /etc/dhcpcd.conf
	# add following line with selected dns servers IPs,f.g:
	static domain_name_servers=9.9.9.9 149.112.112.112
	#
	```

___

- ##  Reference

- a) [https://morfikov.github.io/post/instalacja-debiana-z-wykorzystaniem-debootstrap/](https://morfikov.github.io/post/instalacja-debiana-z-wykorzystaniem-debootstrap/)
- b) [https://gist.github.com/varqox/42e213b6b2dde2b636ef](https://gist.github.com/varqox/42e213b6b2dde2b636ef)
- c) [https://gist.github.com/starquake/856b05dc88d68e7509e23f8995f7ac5e](https://gist.github.com/starquake/856b05dc88d68e7509e23f8995f7ac5e)
- d) [https://linuxcapable.com/how-to-install-kde-plasma-on-debian-linux/](https://linuxcapable.com/how-to-install-kde-plasma-on-debian-linux/)
- e) [https://zihad.com.bd/posts/debian-minimal-kde-plasma-post-install/](https://zihad.com.bd/posts/debian-minimal-kde-plasma-post-install/)
- f) [https://morfikov.github.io/post/przestrzen-wymiany-swap-tylko-na-potrzebny-hibernacji-w-debian-linux/](https://morfikov.github.io/post/przestrzen-wymiany-swap-tylko-na-potrzebny-hibernacji-w-debian-linux/)

________