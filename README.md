# System76 Onyx Pro + Arch Linux

Arch linux installation instructions for a System76 Onyx Pro.

**DISCLAIMER:** This tutorial requires pre-release software out of the testing
repos in order to work. You should be aware that going forwards, you will
likely need to manually manage your installation setup to return to a standard,
non-testing environment. Also, this is Arch Linux and things move fast. If you
find something to be inaccurate or incorrect here, please file an issue. I would
like to know!

#### Partition and format drives

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

#### Setup mirrorlist

```
curl "https://www.archlinux.org/mirrorlist/?country=US" > mirrorlist.b && \
  rankmirrors -n 12 mirrorlist.b > mirrorlist && pacman -Syy
```

#### Pacstrap

`pacstrap -i /mnt base base-devel`

#### fstab

`genfstab -U /mnt > /mnt/etc/fstab`
Verify things look good.

#### Change root

`arch-chroot /mnt /bin/bash`

#### Initial personal env setup

```
pacman -S vim && echo "imap kj <Esc> > ~/.vimrc"
echo "export EDITOR=/usr/bin/vim" > ~/.bashrc && source ~/.bashrc
```

#### Locale

Uncomment `en_US.UTF-8` in `/etc/locale.gen`

```
locale-gen
echo "LANG=en_US.UTF-8" > /etc/locale.conf
export LANG=en_US.UTF-8
```

#### Time

```
hwclock --systohc --utc
pacman -S ntp && systemctl enable ntpd
ln -s /usr/share/zoneinfo/America/New_York /etc/localtime
```

#### Microcode

`pacman -S intel-ucode`

Grub will pick this up and correctly enable the microcode updates,
marking `/boot/intel-ucode.img` as the first initrd in  the bootloader
config file.

#### Boot Loader

Using GRUB for bootloader and chainloading Windows.

`pacman -S grub efibootmger`

`efibootmgr` required for adding EFI boot entries.

Install grub:

```
esp=/boot/efi grub-install --target=x86_64-efi --efi-directory=$esp \
  --bootloader-id=grub --recheck`
```

NOTE: The following is particular to my dual-boot setup. It is not required
unless you are attempting something similar.

Custom boot options for Windows chainloading:

Copy over `40_custom` to `/etc/grub.d/40_custom`, ensure 755 permissions.

If you are dual booting and you have a different partition scheme, substitute
the appropriate values on the search line. You'll assuradly need to edit the
UUID of the ESP partition.

Gen grub config: `grub-mkconfig -o /boot/grub/grub.cfg`

> NOTE: i915.preliminary_hw_support=1 may be a required kernel param.
  If so, add to `GRUB_CMDLINE_LINUX_DEFAULT` in `/etc/default/grub`
  In my testing, I haven't found it to be required since I'm running
  a kernel out of testing, addressed later. YMMV.

#### User config

Enable passwordless sudo if preferred: `visudo`

`useradd -m -G wheel -s /bin/bash $NEW_USER && passwd $NEW_USER`

#### Touchpad drivers

`pacman -S xf86-input-synaptics`

#### Display Setup

This is the difficult part of getting Arch to run on an Onyx Pro.

The Onyx Pro is an Nvidia Optimus enabled laptop. To leverage this, we're going
to use Bumblebee, an effort to get Optimus to run correctly on Linux.
It uses Intel graphics unless specified to use the discrete graphics via `optirun`.

NOTE: I'm choosing to run bumblebee+nvidia rather than nouveau for performance
reasons. The proprietary drivers tend to run a little faster. It is also possible
to use nouveau with bumblebee, in which case, obviously do not blacklist it
in the grub config.

> IMPORTANT: The Onyx Pro ships with it's hybird graphics chipset configured for
  discrete graphics, NOT hybird graphics. Hybrid graphics can be switch on in the
  BIOS settings via advanced chipset settings. Hold F7 > advanced chipset settings >
  switch from DISCRETE to MSHYBRID. Do not do so yet, since the arch iso will not
  boot with this switched on. (More specifically, it will boot, but you will be
  greeted with a blank screen.)

Install packages and setup:

```
pacman -S bumblebee mesa xf86-video-intel nvidia xorg-server
systemctl enable bumblebeed # Enable the bumblebee daemon
gpasswd -a $MY_USER bumblebee # Add main user to bumblebee group
```

  Things are going to vary here depending on your preferred setup. I boot to tty
  login and manually launch Gnome with startx scripts. I will say I've had
  difficulties with some DMs, particularly GDM.

`pacman -S xorg-xinit gnome && echo "exec gnome-session" > /home/$MY_USER/.xinitrc`

### !! Critical Tweaks !!

**Install newest kernel and display drivers**

After much experimentation, I've found this to be the only way to consistently
boot. Again, do so at your own risk. To quote the wiki:

> WARNING: Be careful when enabling the testing repository. Your system may break
  after performing an update. Only experienced users who know how to deal with potential
  system breakage should use it.

> NOTE: testing is not for the "newest of the new" package versions. Part of its purpose
  is to hold package updates that have the potential to break the system,
  either by being part of the core set of packages, or by being critical in other ways.
  As such, users of testing are strongly encouraged to subscribe to the
  [arch-dev-public](https://mailman.archlinux.org/mailman/listinfo/arch-dev-public)
  mailing list, watch the [testing repository forum](https://bbs.archlinux.org/viewforum.php?id=49),
  and to [report all bugs](https://wiki.archlinux.org/index.php/Reporting_bug_guidelines).

1) Enable testing repos. Uncomment `testing` and `community-testing`
   in `/etc/pacman.conf`.

2) Sync repos and upgrade: `pacman -Syyu`. This is important because
   [Arch Linux does not support parial upgrades](https://wiki.archlinux.org/index.php/System_maintenance#Partial_upgrades_are_unsupported).
   You should not cherry pick packages from testing, it's all or nothing.
   We're not in kansas anymore!

3) Install the latest nvidia proprietary drivers: `pacman -S nvidia`. The previous
   upgrade should have pulled in linux 4.4. If not, nvidia will as a dependency.

**Enable early KMS**

We're going to enable early KMS by baking `intel_agp` and `i915` into the intial
RAM image.

Add to MODULES, space separated, in `/etc/mkinitcpio.conf`.

Rebuild intiram img: `mkinitcpio -p linux`

#### Network Setup

```
echo $MY_HOSTNAME > /etc/hostname
pacman -S networkmanager networkmanager-openvpn network-manager-applet
systemctl enable NetworkManager
```
#### Finalization

```
exit # Leave chroot
umount -R /mnt
reboot
```

Hold F7 and enter setup, switch from DISCRETE to MSHYBRID to enable
hybrid graphics. F4 saves and restarts.

At this point, you should be able to boot into Arch proper from the
grub selection. Confirm Intel and Nvidia are both recognized and
bumblebee is happy:

```
lspci | grep VGA
journalctl -u bumblebeed
```

Launch your DE via preferred method. Optionally verify bumblebee is running
correctly:

`pacman -S mesa-demos && optirun glxgears -info`

#### Known issues and remarks

* Unfortunately, we're still faced with a blank screen whenever the screen
is shutoff. As a workaround, I've configured this to never happen. Since
I'm normally on AC power, this isn't a huge deal for me, but it's clearly
not ideal. Still troubleshooting this.

* While running bumblebee, you default to Intel graphics until explicitly
executing with discrete graphics using `optirun`. I've setup a number of
programs to execute via discrete graphics by editing their `Exec`
directives inside the respective `/usr/share/applications/*.desktop`
