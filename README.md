My NixOS configuration.

Prerequisite for creating a NixOS instance in VirtualBox using this config:

* Create a VM with your desired settings.
* Add the virtual disk image file contained in the Apricorn key to the VM as a virtual hard drive. Reference it by label in configuration.nix. It does not work to add the Apricon key as a USB device. NixOS is not able to see it when it is added that way.

Steps for creating a NixOS instance on a VirtualBox VM or bare metal PC:

* Download a NixOS installer ISO image. Boot the machine into it.
* Use `lsblk` to find the medium you will install to.
* Create a single partition in the installation target medium (for legacy BIOS) or two partitions including a 512MB FAT32 boot partition (for UEFI). 
* For UEFI:
```bash
sudo parted $medium -- mklabel gpt
sudo parted $medium -- mkpart primary 512MiB 100%
sudo parted $medium -- mkpart ESP fat32 1MiB 512MiB
sudo parted $medium -- set 2 boot on
sudo mkfs.ext4 $mediumPartition1
sudo mkfs.fat -F 32 -n boot $mediumPartition2
```
   * For BIOS:
```bash
sudo fdisk $medium
p
d # repeat for each partition
n
# accept all defaults to make a new partition taking up all the free space
w
sudo mkfs.ext4 $mediumPartition
```
* Mount the installation target medium on `/mnt`.
* For UEFI, `sudo mkdir /mnt/boot` and mount the boot partition on `/mnt/boot`.
* Generate configuration, install, and reboot: 
```bash
sudo nixos-generate-config --root /mnt
cd /mnt/etc
sudo mv nixos nixos-generated
sudo nix-shell -p git --command "git clone https://github.com/morganthomas/nixos.git"
sudo cp nixos-generated/hardware-configuration.nix nixos
sudo nano nixos/bootloader.nix
```
For a computer in legacy BIOS (non-UEFI) mode put in bootloader.nix:
```nix
{
  boot.loader.grub.enable = true;
  boot.loader.grub.version = 2;
  boot.loader.grub.device = "$medium";
 }
 ```
 For a computer in UEFI mode put in bootloader.nix:
 ```nix
 {
   boot.loader.systemd-boot.enable = true;
   boot.loader.efi.canTouchEfiVariables = true;
 }
 ```
Add a hostname in hostname.nix:
```nix
{
  networking.hostName = "$hostname";
}
```
Set the number of build cores in hardware-configuration.nix to equal the number of virtual cores minus one:
```nix
{
  nix.buildCores = 7;
}
```
 Then:
```bash 
 sudo nixos-install
 # set root password
 sudo nixos-enter
 useradd morgan
 passwd morgan
 # set morgan password
 usermod -G wheel,networkmanager,docker morgan
 mkdir /home/morgan
 chown morgan:users /home/morgan
 dd if=/dev/zero of=/swapfile bs=1G count=8
 chmod 600 /swapfile
 mkswap /swapfile
 (ctrl+d to exit chroot)
 sudo shutdown now
 ```
 
 After booting and logging in as Morgan, take the following steps:
 
 ```bash
 mkdir -p media/SECURE_KEY
 sudo mount media/SECURE_KEY
 mkdir .ssh
 ln -s ~/media/SECURE_KEY/platonic ~/.ssh/platonic
 ln -s ~/media/SECURE_KEY/platonic.pub ~/.ssh/platonic.pub
 ssh-add ~/.ssh/platonic
 git clone git@github.com:morganthomas/dotfiles.git
 ```
 
 Then move everything (including the dotted files and directories) from the dotfiles folder into your home folder and remove the dotfiles folder.

 It's possible to use an Apricorn key as NixOS installation medium using a command like this to write the image:
 
 ```bash
 dd if=image.iso of=/dev/sdb bs=1024
 ```
 
 In order for a computer to boot from the Apricorn key, the key must be placed in lock override mode so that it will not lock up during the boot process.

Note that when updating xmobarrc, you may need to `killall xmobar` before mod+q to see the changes to xmobar.
