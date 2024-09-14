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

**Summary of Commands:**

```bash
fdisk /dev/sda
g
n
<Enter>
<Enter>
+1G
t
<Enter>
1
n
<Enter>
<Enter>
+8G
t
<Enter>
19
n
<Enter>
<Enter>
<Enter>
w
```

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

Now you're ready to proceed with the rest of the Arch Linux installation.