# TM-F2-420-Linux

This is a short, informal guide to set up Arch Linux on a TerraMaster F2-420 NAS.

The [TerraMaster F2-420](http://www.terra-master.com/html/en/article_read_1513.html) is a surprisingly capable machine containing:
- Small, passively cooled Single-Board Computer
- Intel Celeron J1900 4-core 2 GHz (64bit) CPU
- Intel HD Graphics (Ribbon cable VGA connector on-board)
- 4GB RAM + slot expandable additional 4GB DDR3L 1333
- 2 Network and 2 external + 2 internal USB plugs

This differentiates it a bit from many other NAS out there that are often based
on ARM chips and run some specially crafted, proprietary Linux distribution.
The F2-420 on the other hand is based on standard x86_64 and can therefore
run just about any modern OS out-of-the-box.

I got myself one of these with the intent to set it up with my own fully
customized NAS server with all the bells and whistles I want and none of the
bloat.


## Prep

Disassemble the case by unscrewing the back, taking out the board and unscrewing
the drives bracket to get to the boot USB stick.
It might be hot-glued in place but that should peel off with care.

You'll probably want to just pull the "TerraMaster OS" USB out so the machine
doesn't boot into that anymore.
Unfortunately the TOS USB that came in my machine was a tiny 64 MB (who knew
they made them that small?) so I replaced it with my own 16GB USB stick.

Of course you'll want to install your own OS on your own USB stick first.
The easiest way to do this, I found, is to plug the USB into a computer, connect
it to a VirtualBox virtual machine and boot a Live image of the OS you want to
install.


## Install Arch Linux

Installing Arch Linux for the F2-420 is pretty much the same as any other
machine so if you know how all this works you can skip ahead.

Mount the partition you intend to install on, most likely `sda1`:
```
mount /dev/sda1 /mnt
```

Install the base system in the mounted directory, generate an `fstab`, and
chroot into it:
```
pacstrap /mnt base sudo
genfstab -U /mnt >> /mnt/etc/fstab
arch-chroot /mnt
```

Now that we're in the new system, configure timezone and system hardware clock:
```
ln -sf /usr/share/zoneinfo/Asia/Tokyo /etc/localtime
hwclock --systohc --utc
```

And locales:
```
vim /etc/locale.gen # uncomment your desired locale
echo "LANG=en_US.UTF-8" > /etc/locale.conf
locale-gen
```

And hostname:
```
echo nas > /etc/hostname
```

Set the root password, add a normal user, 'nasuser', set password, and finally
configure sudo to allow 'wheel' users:
```
passwd

useradd -m -g users -G wheel nasuser
passwd nasuser

visudo # uncomment wheel with password
```

The base system has now been set up, so generate the boot images:
```
mkinitcpio -p linux
```

Now you can install any other software you'll want on the system.
You'll probably want to install some basic services like SSH, NTP, NTFS, DLNA:
```
pacman -S openssh ntp udisks2 ntfs-3g net-tools ifplugd minidlna
```
And I like to install my set of standard tools:
```
pacman -S vim fish beep
```

If you want SSH, be sure to enable the service:
```
sudo systemctl enable sshd
```
If you're configuring your SSH to only allow keys (ie. not password), remember
to copy the public key from your client (ie. not your NAS) before disabling
password login:
```
ssh-copy-id nasuser@192.168.1.10
```


Now comes the hardest part. You'll have to install a bootloader like GRUB to
make the whole system boot.
When I was setting up my F2-420 I mucked about with this for a long time not
getting anywhere. The system just wouldn't boot and, it being headless, I
couldn't see why.

What I eventually got working was to bootstrap from the TOS image and use the
GRUB from there. But I figure you probably don't have to go through all that
trouble yourself.
In fact if you can make your own grub work, great! But if you can't, try copy
out the `/boot/` directory from the TOS USB (or the image included here).

To make the TOS grub boot Arch, add the following to the end of
`/boot/grub2/grub.cfg`:
```
set default="arch"

menuentry "arch" --id "arch" {
  insmod part_gpt
  insmod ext2
  insmod gzio
  echo "Arch Linux boot option .."
  set root='hd0,gpt1'
  linux /boot/vmlinuz-linux root=UUID=12ab34cd-56ef-78ab-90cd-12ef34ab56cd rw
  initrd /boot/initramfs-linux.img
}
```

NOTE: You'll probably want to use UUIDs instead of /dev/sdXX to avoid boot
issues when adding and removing HDDs and USB drives.
And remember to set `rw` to getting the 'The root device is not configured to be
mounted read-write.' error message.


Lastly, I found that since this system is running headless it can be hard to
tell if it's even booting if we're not able to connect to it.
To give at least some indication I set up another system service that does
exactly one thing: beep once booted.

To set this service up create the following service file:

/etc/systemd/system/beep.service:
```
[Unit]
After=systemd-user-sessions.service

[Service]
Type=oneshot
ExecStart=/usr/bin/beep

[Install]
WantedBy=multi-user.target
```

And don't forget to enable the service:
```
sudo systemctl enable beep
```

The system will now play a single beep with the on-board beeper once the system
has been successfully brought up.


## Additional tips

If NTP seems to be out of sync you can try some combination of:
```
ntpq -p # query
sudo ntpd -qg # force sync
hwclock --systohc # write sys to hw clock
```


If you're planning on running a Windows share, configure SAMBA something like
this:

/etc/samba/smb.conf:
```
[global]
  load printers = No
  printcap name = /dev/null
  disable spoolss = Yes
  follow symlinks = yes
  wide links = yes
  unix extensions = no

[shares]
  path = /shares/
  available = yes
  browsable = yes
  public = yes
  writable = no
```


If you're having trouble booting the boot images you might want to try edit
`/etc/mkinitcpio.conf` to say something like:
```
HOOKS="base udev block autodetect modconf filesystems keyboard fsck"
```
After which you'll of course have to run:
```
sudo mkinitcpio -p linux
```
If you're still having trouble, you can also try edit it to include:
```
MODULES="crc32_generic crc32-pclmul libcrc32c crc32c_generic crc32c-intel"
```
And run the command again..


Since the TOS USB was so small, I decided to dump the image to have a backup
in case I ever wanted to go back.
I've included the image as the 'UTOSBOOT-X86-S64.img'.
