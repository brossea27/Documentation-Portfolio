# Restic Full System Restore

**Author:** brossea27 (GitHub) <br>
**Version:** Q1-24-0 (O.R.)

For future reference, an exclusion file was used for backups and contains the following directories:
  
- /dev
- /mnt
- /proc
- /run
- /swapfile
- /sys
- /tmp 

You will need the following:

- Flash drive
- Keyboard
- Monitor
- Available Restic repository that resides on an external drive
- Replacement MicroSD card (if replacing it)

In this example, the following systems were used:

- Raspberry Pi 4b 4 GB model
- MicroSD card
- Ubuntu LTS v22
- Restic

In this example I will be using a Restic repository by the name of *BRX*.

## 1. The Restore Process:

### 1a. Initial tasks:

- Insert a blank flash drive into your computer.

- Run Raspberry PI Imager on a PC and write the Ubuntu Server LTS v22 64-bit image unto it.

- Don’t need to download the OS, as the software will do that for you. It runs a lot like Media Creation Tool for Windows.

- Insert a blank SD card into your computer and repeat the steps above to write a copy of the OS unto it. This will be the permanent OS drive. 

- Power off the Pi by removing its power cable.

- Insert the flash drive into one of the Pi’s USB 3 ports.

- Remove the corrupted SD card.

- Unplug all other USB devices other than the flash drive and a keyboard. **If you only have 1-2 USB 3.0 ports, it is recommended to plug the keyboard into a USB 2.0 port.**

- Keep the ethernet cable plugged in.

- Plug the Pi’s power cable back in.

- The Pi should automatically boot to the flash drive.

### 1b. Live OS tasks:

- Log in using ubuntu as the username and put nothing for the password. it will then ask you to create a password. *Remember, this is only a live drive, and not the permanent OS on the Pi.*

```bash
     sudo apt install restic
     sudo restic self-update
```

- Plug in the drive from where the Restic repository resides. **USB 3 is recommended.**

- Plug in the new SD card.

- Make temporary mount point for the external drive:

```bash
     mkdir /mnt/drv
```

- Make temporary mount point for the SD card:

```bash
     mkdir /mnt/drv2
```

- Open fdisk:

```bash
     sudo fdisk -l
```

- Get the device IDs of the external storage device and the SD card.
  - Example: External storage device is /dev/sda; SD card is /dev/mmcblk0

### 1c. Expand SD card partition:

**Note:** Only do this section if you replaced the SD card with a larger one and wish to expand your OS’s partition. If not, skip ahead to section 1d.

#### I. Expand the partition:

```bash
sudo growpart /dev/mmcblk0 2
```

This targets the 2nd partition on the SD card from where your OS partition is most likely located (the 1st partition being the boot partition).

#### II. Extend file system to fill unallocated space:

```bash
sudo resize2fs /dev/mmcblk0p2
```

This will increase the partition size to the size of the new replacement SD card. 

Example: If swapping from a 16 GB to 32 GB card, this will expand the partition from 16 GB to 32 GB.

### 1d. Back to general steps:

- Mount the external drive:

```bash
     sudo mount /dev/sd# /mnt/drv
```

- Mount the SD card:

```bash
     sudo mount /dev/mmcblk0p2 /mnt/drv2
```

- Verify that you can see the mounts and **that the OS partition shows the correct max size now**:

```bash
     df -h
```

- Make a temporary cache for Restic:

```bash
     sudo mkdir /mnt/drv/_RS
```

- On the SD card, verify that you can see the typical directories of the main mount point (/) of Linux:

```bash
     ls /mnt/drv2
```

- Show all snapshots of the Restic repository:

```bash
     sudo restic -r /mnt/drv/Backups/BRX snapshots
```

- It will ask for the password.

- Gather the Snapshot ID of the latest backup you’re wanting to restore for the next step.

- Perform the restore:

```bash
     sudo restic -r /mnt/drv/Backups/BRX restore <Snapshot_ID> --target /mnt/drv2 –cache-dir /mnt/drv/_RS
```

- It will ask for the password again.

- It will now restore all your data.

- If successful, proceed on to the next steps.

- Delete temporary Restic cache:

```bash
     sudo rm -R /mnt/drv/_RS
```

- Unmount both mount points:

```bash
     sudo umount /mnt/drv
     sudo umount /mnt/drv2
```

- Perform shutdown:

```bash
     sudo shutdown
```

- When it powers down, remove the power cable and flash drive. Plug the power cable back in. It should now boot to the OS normally and without issues.

- If no issues came up, plug all your typical USB cables back in.

## 1e. Final steps:

**Note:** This section is entirely optional to do. This is intended to do what was excluded in the backup (see exception list) for security purposes.

- Copy Rclone config back:

```bash
     sudo cp -v "/mnt/ld/RCCF/rclone.conf" "/root/.config/rclone"
```
  
- Make directories for Restic key file:

```bash
     sudo mkdir /root/ks
     sudo mkdir /root/ks/rst
```

- Input password and save file:

```bash
     sudo vi /root/ks/rst/brx.kz
```

- Lock down file better:

```bash
     sudo chown root /root/ks/rst/brx.kz
     sudo chmod 700 /root/ks/rst/brx.kz
```

- Test Restic credentials by showing snapshots:

```bash
     sudo restic -r /mnt/ld/Shares/General/Backups/BRX --password-file=/root/ks/rst/brx.kz snapshots
```

- Make temporary mount point (for future drives):

```bash
     sudo mkdir /mnt/td
```
