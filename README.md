# rpi-zfs-root
Guide on running a raspberry pi with a ZFS root filesystem (ubuntu based)

This guide is heavily inspired and derived from the work done by the zfsonlinux folks. I'm sharing my experience on May 12, 2020, having little success with building zfs on the 32 bit OS that is raspbian.

A friend suggested using the latest 20.04 Ubuntu Raspberry Pi edition, since it is 64 bits and the main x86 branch has zfs support. I downloaded the Pi4 tar.xz here: https://ubuntu.com/download/raspberry-pi and loaded it onto a 32GB SD card.

NOTE: This Ubuntu install will (somewhere I can't find) invoke a resize2fs to fill your SD card after every boot. I circumvented this by making an additonal dummy ext4 partition that fills my SD card, leaving me with the standard fat32 /boot parition, and two ext4s, one blank and one with my standard ubuntu OS. I then took the card and plugged it into the Pi4. SSH was already enabled, so I could do ssh ubuntu@ubuntu, password ubuntu. You might have to find the IP of the host if your router doesn't support reverse dns hostname lookup through DHCP.

Next, I kept my eye on this guide from the zfsonlinux github wiki: https://github.com/zfsonlinux/pkg-zfs/wiki/HOWTO-install-Raspbian-to-a-Native-ZFS-Root-Filesystem,-or,-How-I-Learned-to-Love-Data-Integrity. It goes through how to do this on Raspbian, but I hit little luck with this. (I compiled zfs right out of github and copying my root fs was hanging somewhere in the internals of ZFS [kernelspace] - not fun).

As most projects go, I did some googling to start, and found this reddit post: https://www.reddit.com/r/zfs/comments/ekl4e1/ubuntu_with_zfs_on_raspberry_pi_4/

A commenter suggests running `sudo apt install zfs-dkms`. So I did. It seemed to successfully install zfs (I could do zfs and zpool commands), but I took it one step further and install two additional packages after this completed building - `zfsutils-linux` and `zfs-initramfs` (installing these didn't hurt, and we likely need something from zfs-initramfs for this project).

I skipped all the building parts / paritioning of the above wiki tutorial, since we just did with two `apt get` commands (the hard part is done)

We're actually ready to make our pool on top of the dummy 3rd parition we made (make sure it looks right in `parted`)

You should be able to run this (as recommended in the zfsonlinux wiki) 
```
sudo zpool create -o ashift=12 -O canmount=off -O compression=lz4 -O normalization=formD -O mountpoint=/mnt/rastank -O xattr=sa -O relatime=on -O recordsize=1M -R /mnt/rastank rastank /dev/mmcblk0p3

sudo zfs create -o mountpoint=/ rastank/ubuntu
```

^^^ you'll need to put in -f somewhere in the zpool command to force overwriting that dummy ext4 partition. I'll let you put that in because the above command is indeed destructive with -f. Make sure you're on the right Pi, and mmcblk0p3 is really the extra parition.

The second command is creating our root dataset, hence we tell it to mount at /.






