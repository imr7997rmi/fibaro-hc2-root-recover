# How to Recover Root Password on Fibaro HC2 (Firmware 4.630)

## Acknowledgment
This guide is the result of a long and complex troubleshooting process. The architecture of the Fibaro HC2 firmware is non-standard, distributing system files across multiple partitions and using symbolic links for critical configuration files like `/etc/passwd` and `/etc/shadow`. This guide documents the successful method to regain root access after many other standard approaches failed.

**Note**: The Fibaro HC2 uses an internal USB drive for storage, not an mSATA SSD as some might expect.

## 1. Executive Summary
The core challenge is that the HC2 firmware is designed to be immutable. Standard methods of editing the root filesystem fail because the critical password file (`/etc/shadow`) is actually a symbolic link pointing to a file on a separate hardware data partition (HwData). The solution involves cloning the original drive, mounting the specific hardware data partition, and modifying the real password file.

## 2. Prerequisites

### Hardware:
- A computer running a Linux environment (this guide uses an Ubuntu VM on Parallels).
- The original USB drive from the Fibaro HC2.
- A spare, compatible USB drive to use as a working copy. This is highly recommended to avoid any risk to the original drive.

### Software:
- A tool to perform a raw disk copy, like `dd`.
- A Linux environment with standard command-line tools (`lsblk`, `mount`, `gedit`, `openssl`).
## 3. The Process: Step-by-Step Guide

### Step 1: Full Raw Backup of the Original Drive
This is the most critical step. It creates a perfect, bit-for-bit image of your original, working drive, ensuring you always have a safe fallback.

1. Connect the original HC2 USB drive to your computer.
2. Identify the disk identifier (e.g., `/dev/sdb`, `/dev/sdc`). Use `lsblk` or `diskutil list` (on macOS).
3. Use `dd` to create a raw image file. This example assumes the original disk is `/dev/sdb` and saves the backup to the Downloads folder.

```bash
# Command to execute on a Linux/macOS terminal
sudo dd if=/dev/sdb of=~/Downloads/fibaro_hc2_original.img bs=4M status=progress
```

You now have a safe backup image. Store it securely.

### Step 2: Create and Prepare the Working Copy
To avoid any risk, all modifications will be performed on a separate, spare USB drive.

1. Connect your spare USB drive.
2. Identify its disk identifier (e.g., `/dev/sdc`).
3. Restore the backup image you just created onto this spare drive.

```bash
# Command to execute on a Linux/macOS terminal
sudo dd if=~/Downloads/fibaro_hc2_original.img of=/dev/sdc bs=4M status=progress
```

You now have a working clone. All subsequent steps will be performed on this clone.

### Step 3: Identify the Correct Partition and Modify the Password
This is the core of the recovery. We discovered that the real password file is not in the main system partition but in a separate hardware data partition.

1. Connect the cloned USB drive to your Ubuntu machine.
2. Open a Terminal and list the partitions to identify the device name (e.g., `/dev/sdc`). The system will likely automount them.

```bash
lsblk -f
```

You will see multiple partitions. The key partitions are:
- `sdc2` (mounted at `/media/parallels/SystemFS`)
- `sdc6` (mounted at `/media/parallels/HwData`)

**The Discovery**: The file `/media/parallels/SystemFS/etc/shadow` is a symbolic link pointing to the real file located in the HwData partition. We must edit the real file.

3. Generate a new password hash. It is safer to set a new password than to leave it blank. Open a terminal and generate a hash for your desired password (e.g., `MyNewPass123`).

```bash
openssl passwd -6 MyNewPass123
```

The command will output a long string starting with `$6$`. Copy this entire string.

4. Edit the real shadow file. Open the file with a text editor using sudo to get write permissions.

```bash
sudo gedit /media/parallels/HwData/shadow
```

In the editor, you will see a line starting with `root:...`. It will look something like this:
```
root:OLD_HASH_STRING:17280:0:99999:7:::
```

Carefully delete the old hash string (the text between the first and second colons) and paste the new hash you generated. The line should now look like this:
```
root:NEW_HASH_STRING_YOU_PASTED:17280:0:99999:7:::
```

5. Save the file and close the editor.

### Step 4: Finalize and Test

1. Safely eject the cloned drive from your Ubuntu machine.
2. Install the modified USB drive into your Fibaro HC2.
3. Power on the HC2 and wait for it to boot completely.
4. Find the device's IP address from your router.
5. Attempt to connect via SSH. You may need to allow an older security algorithm.

```bash
# Command to execute on your local machine's terminal
ssh -o HostKeyAlgorithms=+ssh-rsa root@<IP_OF_YOUR_HC2>
```

6. When prompted for a password, enter the new password you chose (e.g., `MyNewPass123`).

You should now have successful root access to your Fibaro HC2. It is highly recommended to change the password again immediately using the `passwd` command once you are logged in.

## ⚠️ Important Warnings

- **Always create a backup before starting**
- **Use a spare USB drive for modifications**
- **This process voids your warranty**
- **Proceed at your own risk**

## 🔧 Troubleshooting

### Common Issues:
- **Partitions not auto-mounting**: Try `sudo mount /dev/sdcX /mnt/temp` where X is the partition number
- **Permission denied when editing**: Make sure to use `sudo` when editing the shadow file
- **SSH connection refused**: Ensure the HC2 is fully booted and check firewall settings
- **Wrong partition**: Double-check that you're editing the file in `/media/parallels/HwData/shadow`, not the symlink

## 💡 Technical Details

The Fibaro HC2 uses a unique partition layout:
- **SystemFS**: Contains the main OS files, but `/etc/shadow` is just a symbolic link
- **HwData**: Contains the actual password files and hardware-specific configuration
- **Other partitions**: Boot, recovery, and data partitions

This architecture makes the system resilient to updates but complicates password recovery since standard methods fail when they try to edit the symbolic link instead of the real file.

## Contributing

If you have improvements or encountered different scenarios, feel free to open an issue or submit a pull request.

## License

This guide is provided as-is for educational purposes. Use at your own risk.

---

**Disclaimer**: This procedure involves low-level disk operations and can permanently damage your device if performed incorrectly. The author assumes no responsibility for any damage or data loss.
