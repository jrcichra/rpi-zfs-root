# rpi-zfs-root
Guide on running a Raspberry Pi with a ZFS root filesystem (Ubuntu based)

This guide is heavily inspired and derived from the work done by the zfsonlinux folks. I'm sharing my experience on May 12, 2020, having little success with building zfs on the 32 bit OS that is raspbian.

#### Note - the only downside of this approach as of writing is the Pi takes a few minutes to boot. It's getting caught up in some systemd service that eventually times out. 

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

#### If you're a ZFS veteran, you should be able to customize your dataset and mounts (i.e different dataset for var, home, usr, etc). I'll likely try this in a later deployment.

The second command is creating our root dataset, hence we tell it to mount at /.

Continuing with the guide, I ran this apt install: `sudo apt-get install pv di dialog` to get some insight into the next tar copy.

We're ready to copy our filesystem off of the ext4 parition onto our zfs dataset, which should be mounted at /mnt/rastank. Run this to copy it:

```
(cd /; sudo tar cf - --one-file-system . ) | pv -p -bs $( sudo du -sxm --apparent-size / | cut -f1 )m | (sudo tar -xp -C /mnt/rastank)
```
You might get an error about snapcraft, just ignore it. I haven't tested snapcraft but I'm sure a reinstall would do the trick.

Remove the fstab entry for the ext4 / partition (it actually works with this in but it's good practice): `vi /mnt/rastank/etc/fstab`

Make sure zfs is in our initramfs (should be from our apt install earlier): `lsinitramfs /boot/initrd.img-5.4.0-1008-raspi | grep bin/z`

I added this in /boot/firmware/usercfg.txt. Not sure if it's required/matters:
`initramfs initrd.img-5.4.0-1008-raspi followkernel`

Your kernel might be different than mine. Do `uname -r` and replace (or just try without this and make a PR if it works!)

Next we need to tell the initramfs what to do. This is done in cmdline.txt. Here's mine:
`net.ifnames=0 dwc_otg.lpm_enable=0 console=serial0,115200 console=tty1 root=ZFS=rastank/ubuntu rootfstype=zfs elevator=noop rootwait fixrtc plymouth.enable=0 zfs.zfs_arc_max=67108864`

You can copy this over top of your existing cmdline.txt. We're telling initramfs that our root volume is of type zfs and is the dataset rastank/ubuntu (or whatever you called it)

`reboot`. After a few minutes of it complaining about a missing file in /tmp, which has few hits on Google, it should continue the boot process and give a prompt + ssh access. If you figure this out please open an issue or PR this guide.

If you do `df -h` you should see your new rootfs as the zfs dataset name:
```
ubuntu@ubuntu:/boot/firmware$ df -h
Filesystem      Size  Used Avail Use% Mounted on
udev            1.9G     0  1.9G   0% /dev
tmpfs           380M  3.9M  376M   2% /run
rastank/ubuntu   28G  1.4G   27G   6% /  <--------------------------------
tmpfs           1.9G     0  1.9G   0% /dev/shm
tmpfs           5.0M     0  5.0M   0% /run/lock
tmpfs           1.9G     0  1.9G   0% /sys/fs/cgroup
/dev/loop0       49M   49M     0 100% /snap/core18/1708
/dev/loop1       62M   62M     0 100% /snap/lxd/14808
/dev/loop4       49M   49M     0 100% /snap/core18/1753
/dev/loop2       62M   62M     0 100% /snap/lxd/14958
/dev/loop3       24M   24M     0 100% /snap/snapd/7267
/dev/mmcblk0p1  253M  101M  152M  40% /boot/firmware
tmpfs           380M     0  380M   0% /run/user/1000
ubuntu@ubuntu:/boot/firmware$ 
```

Awesome! You're running a zfs rooted raspberry pi (high five!). There's one more optimization I made before calling it a successful night. We still have that old ext4 partition with our old root. I went ahead and added that partition in the pool by doing this:
`zpool add rastank /dev/mmcblk0p2`

Again, this needs -f to work - make sure mmcblk0p2 is the ext4 partition that was our old root fs and you're on the proper Pi.

Cool! Now you have 3 partitions, one for loading the initramfs (that fat32 parition) and two zfs partitions.

Do a `zpool status` to see your pool:

```
root@ubuntu:~# zpool status
  pool: rastank
 state: ONLINE
  scan: none requested
config:

	NAME         STATE     READ WRITE CKSUM
	rastank      ONLINE       0     0     0
	  mmcblk0p3  ONLINE       0     0     0
	  mmcblk0p2  ONLINE       0     0     0

errors: No known data errors
root@ubuntu:~# 
```

And `zpool list` to see how much space you have:

```
root@ubuntu:~# zpool list
NAME      SIZE  ALLOC   FREE  CKPOINT  EXPANDSZ   FRAG    CAP  DEDUP    HEALTH  ALTROOT
rastank  28.8G  1.40G  27.3G        -         -     0%     4%  1.00x    ONLINE  -
root@ubuntu:~# 
```

Awesome! We turned on lz4 compression in our zpool create, so check on your compression ratio with a fresh install:

```
root@ubuntu:~# zfs get compressratio,used,logicalused /
NAME            PROPERTY       VALUE  SOURCE
rastank/ubuntu  compressratio  1.80x  -
rastank/ubuntu  used           1.40G  -
rastank/ubuntu  logicalused    2.30G  -
root@ubuntu:~# 
```

Check that out! We're saving 1.1Gigs of SD card storage with lz4 compression. It's very low overhead and a Pi4 should be able to handle it.

If you're wondering where some of your storage went (28.8G in zpool list vs 26.5G in zfs list) - read up on `spa_slop_shift`. TLDR zfs is reserving some space for things like `snapshot`, so you can still manage the pool on a full filesystem: https://utcc.utoronto.ca/~cks/space/blog/solaris/ZFSSpaceReportDifference

You can modify the value by echoing into this /sys/ file: `/sys/module/zfs/parameters/spa_slop_shift` if you want to reclaim some space, but just know the risks you are taking. See this question I asked the openzfs community for details on what the number means: https://github.com/openzfs/zfs/issues/10260#issuecomment-620332829

That's it! Feel free to do whatever you want with this Pi! It's a working Linux system with ZFS storage underneath.

If this article helped you, please leave a comment in the open issue and let me know! I'd love to hear your use cases/feedback.
