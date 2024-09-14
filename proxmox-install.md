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
## 4. Bootloader
A bootloader is critical to booting into the OS after powering on the machine. This tells the BIOS what to load to actually start the operating system.

---

### **Prerequisites**

- **Installed Arch Linux System:** Your Arch Linux system is installed on your disk(s).
- **Root Access:** You have administrative privileges (root access) to perform system changes.
- **Mounted Partitions:** Your root (`/`) and EFI System Partition (ESP) are properly mounted.
- **Network Connection:** For installing packages if not already installed.

---

### **Step 1: Boot into the Arch Linux Installation Medium (If Necessary)**

If you cannot boot into your installed Arch Linux system, you can use the Arch Linux installation medium to access your system.

1. **Boot from the Installation Medium:**

   - Insert your Arch Linux USB or DVD and boot your computer.
   - Ensure you boot in **UEFI mode**. You can check this by running:

     ```bash
     ls /sys/firmware/efi/efivars
     ```

     If the directory exists and is populated, you're in UEFI mode.

2. **Connect to the Internet (If Necessary):**

   - For wired connections, you should have internet access automatically.
   - For wireless connections, use `iwctl` or `wifi-menu` to connect.

---

### **Step 2: Mount Your Partitions**

You need to mount your root partition and the EFI System Partition to access and modify your installed system.

**Identify Your Partitions:**

Assuming:

- **Root Partition:** `/dev/sda3`
- **EFI System Partition (ESP):** `/dev/sda1`

Adjust the device names if your setup is different.

**Mount the Root Partition:**

```bash
mount /dev/sda3 /mnt
```

**Mount the EFI System Partition:**

Create the mount point and mount the ESP:

```bash
mkdir -p /mnt/boot/efi
mount /dev/sda1 /mnt/boot/efi
```

**Note:** If your ESP is mounted at `/boot` instead of `/boot/efi`, adjust the mount point accordingly.

---

### **Step 3: Mount Essential Virtual Filesystems**

To ensure proper operation within the chroot environment, mount the necessary filesystems:

```bash
for dir in /dev /dev/pts /proc /sys /run; do
    mount --bind "$dir" "/mnt$dir"
done
```

Mount `efivarfs` for UEFI variables:

```bash
mount -t efivarfs efivarfs /mnt/sys/firmware/efi/efivars
```

---

### **Step 4: Chroot into Your Installed System**

Change root into your installed system to perform operations as if you had booted into it:

```bash
arch-chroot /mnt
```

You are now operating within your installed Arch Linux system.

---

### **Step 5: Install GRUB and efibootmgr**

Ensure that `grub` and `efibootmgr` packages are installed:

```bash
pacman -Syu grub efibootmgr
```

- **`grub`:** The GRUB bootloader package.
- **`efibootmgr`:** A tool to manage UEFI boot entries.

---

### **Step 6: Install GRUB to the EFI System Partition**

Install GRUB for UEFI systems using the following command:

```bash
grub-install --target=x86_64-efi --efi-directory=/boot/efi --bootloader-id=GRUB
```

**Explanation:**

- `--target=x86_64-efi`: Specifies the target platform for 64-bit UEFI systems.
- `--efi-directory=/boot/efi`: Specifies the mount point of your EFI System Partition.
- `--bootloader-id=GRUB`: Sets the bootloader identifier, which determines the directory name in the ESP (`EFI/GRUB`) and the boot entry name in your UEFI firmware.

**Notes:**

- Ensure that the `--efi-directory` option matches where your ESP is mounted (`/boot/efi` or `/boot`).
- Do **not** include a device path (e.g., `/dev/sda`), as UEFI systems do not use the MBR for bootloading.
- If you receive any errors during this step, make sure that the ESP is mounted correctly and that you're in a UEFI boot mode.

---

### **Step 7: Generate the GRUB Configuration File**

Create the `grub.cfg` configuration file, which includes information about your installed kernels and boot options:

```bash
grub-mkconfig -o /boot/grub/grub.cfg
```

**Explanation:**

- The command scans for available kernels and creates `/boot/grub/grub.cfg` accordingly.
- Watch for any errors or warnings during this process.

---

### **Step 8: Verify the Installation**

#### **8.1 Check the GRUB EFI Binary**

Ensure that the GRUB EFI binary is correctly installed:

```bash
ls /boot/efi/EFI/GRUB/grubx64.efi
```

You should see `grubx64.efi` listed.

#### **8.2 Check the GRUB Configuration File**

Verify that the GRUB configuration file exists and has content:

```bash
ls -l /boot/grub/grub.cfg
```

You should see that the file exists and has a reasonable file size.

#### **8.3 Check the UEFI Boot Entries**

List the UEFI boot entries using `efibootmgr`:

```bash
efibootmgr -v
```

Look for an entry labeled `GRUB` pointing to `\EFI\GRUB\grubx64.efi`.

**If the entry is missing or incorrect, you can create it manually:**

```bash
efibootmgr --create --disk /dev/sda --part 1 --label "GRUB" --loader '\EFI\GRUB\grubx64.efi'
```

Replace `/dev/sda` and `--part 1` with your disk and ESP partition numbers if different.

---

### **Step 9: Ensure the EFI Partition is Properly Mounted at Boot**

To ensure that your system mounts the ESP at `/boot/efi` automatically at boot, verify your `/etc/fstab` file.

#### **9.1 Get the UUID of Your ESP**

Find the UUID of your ESP:

```bash
blkid /dev/sda1
```

You'll see output like:

```
/dev/sda1: UUID="XXXX-XXXX" TYPE="vfat" PARTLABEL="EFI System Partition" PARTUUID="YYYYYYYY-YYYY-YYYY-YYYY-YYYYYYYYYYYY"
```

#### **9.2 Edit `/etc/fstab`**

Open `/etc/fstab` with your preferred text editor (e.g., `nano`):

```bash
nano /etc/fstab
```

Ensure there's an entry for your ESP similar to:

```
UUID=XXXX-XXXX  /boot/efi  vfat  defaults  0  1
```

Replace `XXXX-XXXX` with the UUID of your ESP.

**Save and exit** the editor.

---

### **Step 10: Set GRUB as the Default Bootloader**

In most cases, installing GRUB as above and generating the configuration file sets it as the default bootloader. However, to ensure that GRUB is the first boot option:

#### **10.1 Use `efibootmgr` to Adjust Boot Order**

List the current boot order:

```bash
efibootmgr
```

Set GRUB as the first boot option (assuming its boot number is `Boot0000`):

```bash
efibootmgr -o 0000,0001,0002
```

Replace `0000`, `0001`, `0002` with the actual boot numbers from your system, ensuring that the GRUB entry is first.

---

### **Step 11: Exit Chroot and Reboot**

#### **11.1 Exit the Chroot Environment**

```bash
exit
```

#### **11.2 Unmount All Partitions**

```bash
umount -R /mnt
```

If you receive a "busy" error, ensure that all shells and programs have exited from `/mnt`.

#### **11.3 Reboot Your System**

```bash
reboot
```

Your system should now boot into GRUB, displaying the boot menu, and allowing you to select and boot your installed Arch Linux system.

---

### **Additional Considerations**

#### **A. Secure Boot**

If your system uses Secure Boot, additional steps are required to sign the GRUB EFI binary and your kernel. For simplicity, you can disable Secure Boot in your UEFI firmware settings.

#### **B. Using `--removable` Option (Optional)**

As a fallback, you can install GRUB to the default/fallback boot path, which some UEFI firmware uses if no boot entries are found:

```bash
grub-install --target=x86_64-efi --efi-directory=/boot/efi --removable
```

This installs GRUB to `/boot/efi/EFI/BOOT/BOOTX64.EFI`.

#### **C. Verify Kernel and Initramfs Images**

Ensure that your kernel and initramfs images are present in `/boot`:

```bash
ls /boot
```

You should see:

- `vmlinuz-linux` (kernel image)
- `initramfs-linux.img` (initramfs image)
- `initramfs-linux-fallback.img`

If these are missing, install the kernel package:

```bash
pacman -S linux linux-firmware
```

---

### **Troubleshooting**

#### **Issue: GRUB Menu Does Not Appear**

- **Cause:** GRUB may not be properly installed or configured.
- **Solution:** Reinstall GRUB and regenerate `grub.cfg` as in Steps 6 and 7.

#### **Issue: System Boots to GRUB Prompt (`grub>`), Not Menu**

- **Cause:** GRUB cannot find `grub.cfg`.
- **Solution:**

  - Ensure `grub.cfg` exists in `/boot/grub/`.
  - Copy `grub.cfg` to the EFI directory:

    ```bash
    cp /boot/grub/grub.cfg /boot/efi/EFI/GRUB/
    ```

#### **Issue: Errors During `grub-install`**

- **Cause:** ESP not properly mounted, not in UEFI mode, or missing dependencies.
- **Solution:**

  - Confirm the ESP is mounted at `/boot/efi`.
  - Ensure you're booted in UEFI mode.
  - Install missing packages: `grub`, `efibootmgr`.

#### **Issue: Cannot Boot After GRUB Installation**

- **Cause:** Incorrect boot order, Secure Boot interference, or UEFI firmware issues.
- **Solution:**

  - Adjust boot order with `efibootmgr`.
  - Disable Secure Boot.
  - Update UEFI firmware if updates are available.

---

### **Summary of Commands**

```bash
# Boot from installation medium and mount partitions
mount /dev/sda3 /mnt
mkdir -p /mnt/boot/efi
mount /dev/sda1 /mnt/boot/efi

# Mount virtual filesystems
for dir in /dev /dev/pts /proc /sys /run; do
    mount --bind "$dir" "/mnt$dir"
done
mount -t efivarfs efivarfs /mnt/sys/firmware/efi/efivars

# Chroot into the system
arch-chroot /mnt

# Install GRUB and efibootmgr
pacman -Syu grub efibootmgr

# Install GRUB to the EFI System Partition
grub-install --target=x86_64-efi --efi-directory=/boot/efi --bootloader-id=GRUB

# Generate GRUB configuration file
grub-mkconfig -o /boot/grub/grub.cfg

# Verify UEFI boot entries
efibootmgr -v

# (Optional) Adjust boot order
efibootmgr -o 0000,0001,0002  # Replace with actual boot numbers

# Exit chroot and unmount partitions
exit
umount -R /mnt

# Reboot the system
reboot
```

---

### **Final Notes**

- **Consistency in Mount Points:** Ensure that the ESP is consistently mounted at the same location (`/boot/efi` or `/boot`) during installation and normal operation.
- **UEFI Mode:** Verify that your system is booted in UEFI mode when performing these steps.
- **Backup Important Data:** Before making changes to the bootloader, it's wise to backup any important data.

---