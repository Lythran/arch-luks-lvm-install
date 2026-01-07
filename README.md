# Installing Arch Linux with LVM on LUKS
This guide only works on x86_64 bit computers that have UEFI/EFI enabled. This follows the arch wiki [installation guide](https://wiki.archlinux.org/title/Installation_guide#Installation) and the [LVM on LUKS guide](https://wiki.archlinux.org/title/Dm-crypt/Encrypting_an_entire_system#LVM_on_LUKS). This also uses [Plymouth](https://wiki.archlinux.org/title/Plymouth) and the ```bgrt``` theme using the ```rhgb``` (Red Hat Graphical Boot) to look fancy.There is no swap partition because I am too lazy to encrypt and decrypt it (for now), but you can take a look at it [here](https://wiki.archlinux.org/title/Dm-crypt/Swap_encryption).

**THIS IS A BASE INSTALLATION, YOU CAN INSTALL MORE THINGS LATER!**

![](/luks-prompt.png)
![](/luks-unlocked.png)

## INSTRUCTIONS
Ensure that you can connect to the interweb first and formost
```
ping ping.archlinux.org
```

Sychronize the repositories, you don't want any errors when you pacstrap!
```
pacman -Sy
```

See the disk
```
fdisk -l
```

Select the disk, for this we will use /dev/sda
```
fdisk /dev/sda
```

Create a new GPT label, press lowercase g then hit enter
```
g
```

Make a new partition for BOOT, this will be sda1, press 1 then hit enter
```
1
```

The first sector can be left as default, you can press enter.
The second sector is recommended to set it to 512MB for only 1 kernel, else set it to 1GB
```
+512MiB
```

Make it an EFI type, press lowercase t, then 1
```
t
1
```

Now we make the root directory, this will be sda2. Type n, press enter, then press 2 then hit enter
```
n
2
```
Leave first sector at default
Since this is the root partition, you can let it populate the rest of the disk, press enter
Leave it at the default Linux Filesystem
To see the actions, press lowercase p, it should look something like this:
![](/fdisk.png)

Press lowercase ```w``` then hit enter. Keep in mind which partition is the boot and root, for this, sda1 will be boot and sda2 will be root.

Format the Efi partition like so:
```
mkfs.fat -F 32 /dev/sda1
```

Now we encrypt! Make sure that you have a strong password for this.
```
cryptsetup luksFormat /dev/sda2
```

Now we open it and give it a name, for this, let's call it root to keep it simple. You can name it anything you want.
```
cryptsetup open /dev/sda2 root
```

Now lets format it to ext4. You can use any file system as long as you follow the arch wiki when it comes to disk encryption with LVM on LUKS.
```
mkfs.ext4 /dev/mapper/root
```

Then we mount the mapper root to ```/mnt```
```
mount /dev/mapper/root /mnt
```

Likewise for the boot partition. Remember, in order to even access the boot partition like GRUB or others, it must remain unencrypted. There are work arounds but for this guide, it will remain unencrypted. 
```
mount --mkdir /dev/sda1 /mnt/boot
```

From this point on, we can follow through the normal arch wiki [install guide](https://wiki.archlinux.org/title/Installation_guide#Installation) until we hit initramfs and the boot loader steps.

You can select the mirrors and the rest, but what's most important is the packages you have to install during pacstrap.
```
pacstrap -K /mnt base linux linux-firmware git curl vi vim sudo networkmanager lvm2 plymouth grub efibootmgr
```
These are the most essential packages, else all of this won't work. You can later install other packages afer you reboot.

Generate the UUID of ```/mnt``` and send it over to the new fstab
```
genfstab -U /mnt >> /mnt/etc/fstab
```

Then, chroot into the system
```
arch-chroot /mnt
```

Configure the time zone in the format of ../Country/City. For example
```
ln -sf /usr/share/zoneinfo/America/New_York /etc/localtime
```

After that, we generate the adjusted time
```
hwclock --systohc
```

Now we edit the [localization](https://wiki.archlinux.org/title/Installation_guide#Localization)
```
vim /etc/locale.gen
```
Find the locale of you language. For American English in vim, hit slash once ```/```, then type ```en_US```. The format should be ```UTF-8```. Then we generate it using the following command
```
locale-gen
```

The output should be similar to this

![](/locale.png)

Create the locale.conf file, and set the LANG variable accordingly:
```
vim /etc/locale.conf
```
while editing it, press i to insert and type out the LANG variable that suits you. For example:
```
LANG=en_US.UTF-8
```
Press ```escape```, press ```:```, then ```wq``` to write and quit vim

Another nice to have is setting up the [console keyboard layout](https://wiki.archlinux.org/title/Installation_guide#Set_the_console_keyboard_layout_and_font)
```
vim /etc/vconsole.conf
# for english
KEYMAP=en
```
Save and exit vim ```:wq```

Now lets set up the hostname, same process as before
```
vim /etc/hostname
arch
```
Save and exit vim ```:wq```

Before we generate a ```initramfs```, we must add the necessary hooks for ```mkinitcpio.conf```.
To do so, we must edit ```/etc/mkinitcpio.conf```
```
vim /etc/mkinitcpio.conf
```
Add ```keyboard```, ```sd-encrypt```, ```lvm2``` to the ```HOOKS``` line. It should look something similar to this
```
HOOKS=(base systemd autodetect microcode modconf kms keyboard keymap sd-vconsole block plymouth sd-encrypt lvm2 filesystems fsck)
```
And yes, the order matters...

NOW we can regenerate the ```initramfs```
```
mkinitcpio -P
```

Set a strong root password
```
passwd
```

Give users in group ```wheel``` superuser privilages
```
sudo visudo
```
Uncomment the first wheel (remvoe the hashtags)

![](/wheel.png)

Make a user with a (bash) shell and group wheel
```
useradd -m -G wheel -s /bin/bash me
```
Give the user a super strong password
```
passwd me
```

Now we [configure the bootloader](https://wiki.archlinux.org/title/Dm-crypt/Encrypting_an_entire_system#Configuring_the_boot_loader_2), in this case, GRUB
We append the UUID of the encrypted root to ```GRUB_CMDLINE_DEFAULT``` in the format of
```
rd.luks.name=device-UUID=cryptlvm root=/dev/mapper/root
```
Where device-UUID is the UUID of the encrypted root
To get that we run
```
blkid -s UUID -o value /dev/sda2 >> /etd/default grub
```
Then we move it around in vim or your favourite text editor to make it look more like this
```
GRUB_CMDLINE_LINUX_DEFAULT="loglevel=3 quiet rd.luks.name=device-UUID=root root=/dev/mapper/root
```

While we're still in GRUB, we must add ```rhgb``` to ```GRUB_CMDLINE_LINUX``` for the Plymouth theme and add ```lvm``` to ```GRUB_PRELOAD_MODULES``` according to [this](https://wiki.archlinux.org/title/GRUB#LVM), though, it soon may change :)

It needs to look something like this
```
GRUB_CMDLINE_LINUX="rhgb"
...
GRUB_PRELOAD_MODULES="part_gpt part_msdos lvm"
```

Finally, we can install grub to ```/boot``` using the following command
```
grub-install --target=x86_64-efi --efi-directory=/boot/ --bootloader-id=GRUB
grub-mkconfig -o /boot/grub/grub.cfg
```

And set the Plymouth theme accordingly (you can change it to whatever you want)
```
plymouth-set-default-theme -R bgrt
```

FINALLY, ```exit``` chroot, ```umount -R /mnt```, and reboot. Congratulations, you have a fully encrypted root with LVM on LUKS!
