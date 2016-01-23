# System76 Onyx Pro + Arch Linux

Arch linux installation instructions for a System76 Onyx Pro.

**Partition and format drives**

I'm dual booting Windows and using the SSDs available to me, so
my partition setup is a little strange. Since Windows 10 likes to blow away
anything you setup on the drive first, it's best to install that first and
use the ESP partition it generated.

After installing Windows onto the larger of my 2 SSDs, I shrunk the data partition
down as low as it would allow me to go. For some reason, I was only able to get down
to about 130G, which was okay by me because that's about what I was shooting for anyway.
It seems Windows limits how much you can shrink the partition since it has certain
critical files on disk that prevents shrinking farther. Also, defragging won't help.
Yay Microsoft. Additionally, Windows generated a second recovery partition at the end of
the drive for some reason. I removed it to make way for the linux partitions.

```
/dev/sda # 128GB drive shipped with machine
  * /dev/sda1 -> 4G Linux Swap
  * /dev/sda2 -> 124G ext4 /home

/dev/sdb # 256GB drive I installed
  * /dev/sdb1 -> 300M Windows Recovery
  * /dev/sdb2 -> 99M FAT32 ESP (this is important!)
  * /dev/sdb3 -> 128M Microsoft Reserved (no idea what this is.)
  * /dev/sdb4 -> 134.3G Microsoft basic data (main windows data partition)
  * /dev/sdb5 -> 30G ext4 /
  * /dev/sdb6 -> 68.1 ext4 /var
```

`mkswap /dev/sda1 && swapon`

Create mountpoints and mount up partitions.

**Setup mirrorlist**
```
curl "https://www.archlinux.org/mirrorlist/?country=US" > mirrorlist.b && \
  rankmirrors -n 12 mirrorlist.b > mirrorlist && pacman -Syy
```

**Pacstrap**

`pacstrap -i /mnt base base-devel`

**fstab**

`genfstab -U /mnt > /mnt/etc/fstab`
Verify things look good.

**Change root**

`arch-chroot /mnt /bin/bash`

**Initial personal env setup**

```
pacman -S vim && echo "imap kj <Esc> > ~/.vimrc"
echo "export EDITOR=/usr/bin/vim" > ~/.bashrc && source ~/.bashrc
```

**Locale**

Uncomment `en_US.UTF-8` in `/etc/locale.gen`

```
locale-gen
echo "LANG=en_US.UTF-8" > /etc/locale.conf
export LANG=en_US.UTF-8
```

**Time**

`tzselect` -> Choose correct timezone

`ln -s /usr/share/zoneinfo/America/New_York /etc/localtime`

`hwclock --systohc --utc`

**Microcode**

`pacman -S intel-ucode`

Grub will pick this up and correctly enable the microcode updates,
marking `/boot/intel-ucode.img` as the first initrd in  the bootloader
config file.

**Boot Loader**

Using GRUB for bootloader and chainloading Windows.

`pacman -S grub efibootmger`

`efibootmgr` required for adding EFI boot entries.
