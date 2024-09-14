# Arch Linux Setup Guide
## 0.1 Proxmox Initial Setup

### Download ISO
* Find the iso at the [Arch Linux Mirrors Page](https://archlinux.org/download/).
* Find a good mirror for the iso you want, then copy the link to the download and paste into proxmox's iso downloader.
* Copy the SHA256 hash from the mirror page and paste it into the hash field on the ISO downloader.

### Pre Install VM Setup
Enabling EFI for Arch as guest is optional. If you want to install Arch Linux in EFI mode inside Proxmox, you must change the firmware mode for the virtual machine. This must be done before installing Arch as guest, changing the option afterwards will result in unbootable machine unless the setting is reverted.

* Select OVMF (UEFI) over SeaBIOS in Hardware > BIOS (cf. BIOS and UEFI).
* Start the virtual machine.
* Immediately press ESC to enter the firmware setup.
* Disable the secure boot else the ArchLinux ISO won't be detected.
* Stop the virtual machine.
* Edit the boot order option to put the virtual CDROM player before your disk (not by default).
* Start the virtual machine.
* (Optional) [Verify the boot mode](https://wiki.archlinux.org/title/Installation_guide#Verify_the_boot_mode).

## 1.7 Verify Internet Connection
Internet should be enabled by default on proxmox. Check for an ip with `ip addr` and `ping archlinux.org` to check connection status.

## 1.9 Disk Partition Setup 
To set up your partitions on a 450 GB drive with fdisk, follow these detailed steps. This will create:

- A **1 GB EFI system partition**
- An **8 GB swap partition**
- The **remaining space as the root partition**

**Step 1: Start `fdisk` on your drive**

Assuming your drive is `/dev/sda`, open fdisk:

```bash
fdisk /dev/sda
```

**Step 2: Create a new GPT partition table**

Inside fdisk, type:

```
g
```

This initializes a new empty GPT partition table.

**Step 3: Create the EFI system partition**

1. **Create a new partition:**

   ```
   n
   ```

2. **Partition number:** Press `Enter` to accept the default (1).

3. **First sector:** Press `Enter` to accept the default (start at the beginning).

4. **Last sector:** Type `+1G` to specify the size.

   ```
   +1G
   ```

5. **Set the partition type to EFI System:**

   ```
   t
   ```

   - **Partition number:** Press `Enter` (defaults to the last partition created, which is 1).
   - **Hex code or type:** Type `1` for EFI System.

     ```
     1
     ```

**Step 4: Create the swap partition**

1. **Create a new partition:**

   ```
   n
   ```

2. **Partition number:** Press `Enter` (defaults to 2).

3. **First sector:** Press `Enter` to accept the default (starts after the EFI partition).

4. **Last sector:** Type `+8G` to specify the size.

   ```
   +8G
   ```

5. **Set the partition type to Linux swap:**

   ```
   t
   ```

   - **Partition number:** Press `Enter` (defaults to 2).
   - **Hex code or type:** Type `19` for Linux swap.

     ```
     19
     ```

**Step 5: Create the root partition**

1. **Create a new partition:**

   ```
   n
   ```

2. **Partition number:** Press `Enter` (defaults to 3).

3. **First sector:** Press `Enter` to accept the default.

4. **Last sector:** Press `Enter` to use the remaining space on the drive.

5. **Partition type:** The default (`Linux filesystem`) is correct, so you can skip changing it.

**Step 6: Write the changes and exit**

Type:

```
w
```

This writes the partition table to the disk and exits fdisk.

**Next Steps:**

After partitioning, you'll need to format the partitions and continue with the Arch Linux installation:

- Format the EFI partition to FAT32:

  ```bash
  mkfs.fat -F32 /dev/sda1
  ```

- Initialize the swap partition:

  ```bash
  mkswap /dev/sda2
  swapon /dev/sda2
  ```

- Format the root partition (e.g., to ext4):

  ```bash
  mkfs.ext4 /dev/sda3
  ```

- Mount the root partition:

  ```bash
  mount /dev/sda3 /mnt
  ```

- Create and mount the EFI directory:

  ```bash
  mkdir /mnt/boot
  mount /dev/sda1 /mnt/boot
  ```

## 2 Installing and updating
We're going to use reflector to update our mirrors. We'll have to first update our packages then install reflector.
```sh
# Sync pacman mirrors
pacman -Sy
# Install reflector
pacman -S reflector
```
Backup your current mirrorlist
```sh
cp /etc/pacman.d/mirrorlist /etc/pacman.d/mirrorlist.bak
```
Generate a new mirrorlist using Reflector focused on NA and European servers.
```sh
reflector --country 'United States,Canada,Germany,France,United Kingdom,Netherlands,Sweden,Finland,Denmark,Norway,Austria,Belgium,Switzerland,Spain,Italy' \
          --age 12 \
          --protocol https \
          --sort country \
          --save /etc/pacman.d/mirrorlist
```
### 2.1 check mirrors
```sh
cat /etc/pacman.d/mirrorlist
```
Modify mirrors as desired. [Mirror List](https://wiki.archlinux.org/title/Mirrors)

### 2.2 Install essential packages

Install the base packages with pacstrap.
```sh
pacstrap -K /mnt base linux linux-firmware
```

## Configuring the system
### 3.1 Fstab

Generate an fstab file (use -U or -L to define by UUID or labels, respectively):
```sh
genfstab -U /mnt >> /mnt/etc/fstab
```
Check the resulting /mnt/etc/fstab file, and edit it in case of errors.

### 3.2 Mount into the newly setup system
Mounting into `chroot` will allow you to install programs on the OS that will be booted after finishing the install.
```sh
arch-chroot /mnt
```

### 3.3 Timezone
Set the time zone:
```sh
ln -sf /usr/share/zoneinfo/America/New_York /etc/localtime
```
Run hwclock(8) to generate /etc/adjtime:
```sh
hwclock --systohc
```
This command assumes the hardware clock is set to UTC

### 3.4 Localization

Edit /etc/locale.gen and uncomment en_US.UTF-8 UTF-8 and other needed UTF-8 locales. Generate the locales by running:
```sh
locale-gen
```
Create the locale.conf file, and set the LANG variable accordingly:
```sh
touch /etc/locale.conf
vim /etc/locale.conf
```
Uncomment `LANG=en_US.UTF-8`

### 3.5 Network configuration
Create the hostname file:
```sh
touch  /etc/hostname
vim /etc/hostname

# set your hostname
yourhostname
```
Complete the network configuration for the newly installed environment. 

Install a network manager, like NetworkManager CLI
```sh
pacman -S networkmanager

# Enable and start NetworkManager.service
systemctl enable NetworkManager.service
systemctl start NetworkManager.service
```
### 3.7 Root password
Set the root password:
```sh
passwd
```
