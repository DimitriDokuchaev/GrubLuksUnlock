# GrubLuksUnlock
How to Unlock Luks containers using Grub


This is a tutorial on how to unlock luks1 containers using grub bootloader.

Why are you not unlocking it normally?
This issue surfaced from the necessity of unlocking the luks container on Macbook Pro 16,2 that unfortunately doesn't have the kernel module loaded at the time you are prompted for the password.

Then why GRUB?
Because on GRUB Macbook Pro's internal keyboard is still working correctly.

What does this tutorial assume?
- you are running ubuntu;
- you are using a macbook pro with a T2 chip;
- you want to setup encryption on your ubuntu install;

The theory behind this:
The basic principle behind this is that the /boot partition must be inside an encrypted partition, if that happens, then GRUB will obviously ask you for the password in order to be able to decrypt the /boot partition and then be able to read the kernel and initramfs.
It is up to you if you want to have the whole system inisde a partition or have multiple partitions, as long as the contents of /boot are encrypted GRUB will ask for a password.

How i set up this for myself?
I like to use and have LVMs and obviously encrypt everything, I generally leave /boot unencrypted, however as stated above we need to encrypt boot to have grub prompting us for a password.
Here is the setup i do on my macbook:
- Efi
- boot (Encrypted with luks1)
- UbuntuLVM (encrypted with luks2)
  - root
  - home
  - swap
  - var
  - tmp
  - var_log
  - var_tmp
  
Feel free to use whatever partition layout you prefer, but as i have said before make sure boot is encrypted with luks1, for the tutorial i am going to follow the steps for the layout above, feel free to addapt to your needs.

Let's begin

Once you boot the ubuntu iso, make sure to select your keyboard layout in ubuntu settings (lets avoid issues with password inputs because you are using a different layout).
Connect to the internet and apt install gparted
open gparted and select empty space and create a new partition of about 1gb for boot;
still on gparted select the rest of the empty space and create another partition for the operating system;
close gparted;

open the terminal, type:
cryptsetup luksFormat --type=luks1 -v /dev/nvme0n1p5 (my boot partition is on partition 5)
cryptsetup luksFormat --type=luks2 -v /dev/nvme0n1p6 (my Ubuntu partition is on partition 6)

now lets open the partitions:
cryptsetup open --type=luks1 -v /dev/nvme0n1p5 EncryptedUbuntuBoot
cryptsetup open --type=luks2 -v /dev/nvme0n1p6 EncryptedUbuntuLVM

now that the partitions are open, let's create the lvm on the main partition:
pvcreate /dev/mapper/EncryptedUbuntuLVM
vgcreate UbuntuVG /dev/mapper/EncryptedUbuntuLVM
lvcreate -L16G UbuntuVG -n swap
lvcreate -L60G UbuntuVG -n root
lvcreate -L4G UbuntuVG -n tmp
lvcreate -L4G UbuntuVG -n var
lvcreate -L4G UbuntuVG -n var_log
lvcreate -L4G UbuntuVG -n var_tmp
lvcreate -l 100%FREE UbuntuVG -n home

Now lets format the partitions, i always go for ext4, feel free to chose what you like for your partitions:
mkfs.ext4 /dev/mapper/UbuntuVG-root -L "root"
mkfs.ext4 /dev/mapper/UbuntuVG-home -L "home"
mkfs.ext4 /dev/mapper/UbuntuVG-tmp -L "tmp"
mkfs.ext4 /dev/mapper/UbuntuVG-var -L "var"
mkfs.ext4 /dev/mapper/UbuntuVG-var_log -L "var_log"
mkfs.ext4 /dev/mapper/UbuntuVG-var_tmp -L "var_tmp"
mkfs.ext4 /dev/mapper/EncryptedUbuntuBoot -L "boot"

now all the partitions are according to what we want, let's install ubuntu, in terminal type:
ubiquity --no-bootloader

This will open ubuntu install, but it won't install the bootloader, it will be installed manually later, so on the installer select whatever you feel appropriate and when you reach the partitioning stage select "Something Else":
select each partition created and its mount point

/dev/mapper/EncryptedUbuntuBoot format as ext4 mount to /boot
/dev/mapper/UbuntuVG-root format as ext4 mount to /
/dev/mapper/UbuntuVG-home format as ext4 mount to /home
/dev/mapper/UbuntuVG-tmp format as ext4 mount to /tmp
/dev/mapper/UbuntuVG-var format as ext4 mount to /var
/dev/mapper/UbuntuVG-var_log format as ext4 mount to /var_log
/dev/mapper/UbuntuVG-var_tmp format as ext4 mount to /var_tmp

Notice we have been ignoring swap, this is on porpose as swap seems to give an error on the installer we will fix swap after.

After setting all the mountpoints continue, set the rest of the options and wait for the installation to finish, once its finished DO NOT REBOOT, select continue testing.

again in the terminal we need to chroot into our install, let's setup the chroot environment:
mount /dev/mapper/UbuntuVG-root /mnt
mount /dev/mapper/EncryptedUbuntuBoot /mnt/boot
mount /dev/mapper/UbuntuVG-home /mnt/home
mount /dev/mapper/UbuntuVG-tmp /mnt/tmp
mount /dev/mapper/UbuntuVG-var /mnt/var
mount /dev/mapper/UbuntuVG-var_log /mnt/var/log
mount /dev/mapper/UbuntuVG-var_tmp /mnt/var/tmp
for n in proc sys dev etc/resolv.conf; do mount --rbind /$n /mnt/$n; done
chroot /mnt /bin/bash

We are now inside our Ubuntu install, now we are going to create keys, why keys? because if we don't create keys in this setup we will have to input the password 3 times two of those where the keyboard doesn't work in the first place, so in order to bypass the fact of the keyboard not working we use keys, I choose to create two different keys, one for the /boot partition and another for the main os partition, let's go
mkdir /etc/luks
dd if=/dev/urandom of=/etc/luks/Bootkey.keyfile bs=4096 count=1 (this is the key for the /boot partition)
dd if=/dev/urandom of=/etc/luks/OSkey.keyfile bs=4096 count=1 (this is the key for the main OS partition)
let's set propper secure permissions:
chmod u=rx,go-rwx /etc/luks
chmod u=r,go-rwx /etc/luks/BootKey.keyfile
chmod u=r,go-rwx /etc/luks/OSKey.keyfile

now let's add both keyfiles to each encrypted partitions:
cryptsetup luksAddKey /dev/nvme0n1p5 /etc/luks/BootKey.keyfile (the Bootkey to the boot partition)
use the password you set up for the partition
cryptsetup luksAddKey /dev/nvme0n1p6 /etc/luks/OSKey.keyfile (the OSkey for the main OS partition)
use the password you set up for the partition

you can check the key was added successfully to each partition:
cryptsetup luksDump /dev/nvme0n1p5 (you will see to keyslots occupied slot0 and slot1, slot0 is the password, slot1 is the key)
cryptsetup luksDump /dev/nvme0n1p6 (you will see to keyslots occupied slot0 and slot1, slot0 is the password, slot1 is the key)

now let's add the keys to crypsetup conf-hooks so they get included in the initramfs.
vi /etc/cryptsetup-initramfs/conf-hook 
at the end of the file uncomment KEYFILE_PATTERN and add:
KEYFILE_PATTERN=/etc/luks/*.keyfile

now let's set the permissions on the initramfs so keys don't leak, that would be a shame wouldn't it? 
vi /etc/initramfs-tools/initramfs.conf
inside type UMASK=0077

lets create crypttab:
vi /etc/crypttab
inside type the following:
EncryptedUbuntuBoot /dev/nvme0n1p5  /etc/luks/BootKey.keyfile luks
EncryptedUbuntuLVM  /dev/nvme0n1p6  /etc/luks/OSKey.keyfile luks

let's download grub bootloader:
apt install grub-efi-amd64 mtools os-prober efibootmgr

now let's edit the settings before installing the bootloader:
vi /etc/default/grub

inside the file add the following (this is what we need to add to have GRUB asking for a password and it shall be the only password you will have to input because then the keyfiles kick in):

GRUB_CMDLINE_LINUX="efi=noruntime pcie_ports=compat acpi=force"
GRUB_ENABLE_CRYPTODISK=y

also remove the quiet and splash on the GRUB_CMDLINE_LINUX_DEFAULT=""
GRUB_HIDDEN_TIMEOUT=5
GRUB_TIMEOUT_STYLE=menu

once all changes are done exit the file

Let's re-install the same packages above:
apt install -y --reinstall grub-efi-amd64 mtools os-prober efibootmgr

let's update initramfs
update-initramfs -c -k all

let's install grub
grub-install /dev/nvme0n1

let's update grub
update-grub

exit out the chroot and reboot, cross your fingers, you should be prompted for a password while GRUB is booting, once you input the password you should be able to select the kernel and boot the OS, that's it.





Thanks to Redecorating for the idea of using GRUB 
