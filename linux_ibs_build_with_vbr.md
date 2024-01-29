# Linux Server Setup with Veeam Immutable Backups Integration

**Author:** brossea27 (GitHub) <br>
**Version:** Q1-24-0 (O.R.)

This was made on a Dell server using hardware RAID with multiple NICs. iDrac and NIC1 were in use. Being that both are independent, only NIC1 is available to the OS, and it will use static IP configuration.

This has been written for use with the OS *Ubuntu LTS v20/v22* and the backup software *Veeam Backup & Replication (VBR) v11/v12*.

## 1. BIOS settings:

- Have UEFI enabled.

- Disable Secure Boot

- If using hardware RAID, go ahead and create the array(s).

## 2. OS installation:

- Mount the Ubuntu LTS ISO file and boot to it.

- The installation media will now load. When prompted, select Install Ubuntu LTS.
  - It may take a while to load. If you receive a prompt about "cloud-init failed to complete after…", click Close.

- For language selection, select English.

- If it prompts about an installer update available, click "Update to the new installer". Then wait while it installs.

- Keep the defaults for keyboard configuration.

- For the type of install, select Ubuntu Server.

- Under network connections, the first NIC (eno1) should have already gotten an IP address through DHCP. Don't worry about eno2 showing not connected.

- Change the IP configuration of eno1 (NIC1) to Static and input the needed stack into it.
 - If successful, eno1 should show static with the IP address given.

- You may now disable IPv6 on it if you wish. It is recommended to disable it if you don't intend on using it (will help with DNS).

- Keep defaults for proxy address.

- For configure ubuntu archive mirror, keep the defaults. **If IP configuration was truly successful (where it can hit the internet), it should say “This mirror location passed tests.”**

- If it fails, go back and make sure your IP configuration was right!

- For storage layout, choose Custom.

- If existing partitions show under Available Devices, you'll need to remove them. **For each of them, you'll need to reformat them.** This option can be accessed by using SPACE to access its sub-menu. If it doesn’t show any partitions, you may skip the step of reformatting/deleting them.

- **If done correctly, it should now show all free space.**

- Access the sub-menu for free space > Add GPT Partition

- Make this one 30g (30 GB), select ext4 for its format, and / as the mount.

- Make another GPT partition with the free space. Put nothing in its size, so it will be the max. Select **xfs** for its format and Other as the Mount. Name the mount */mnt/veeamrepo*.

- Verify partitioning information

- Input user information

- Next screen: Skip for now and click Continue.

- The OS will now install.

- Reboot Now (when it is finished)

- When the OS comes up, login.

## 3. OS configuration:

- Update and upgrade packages on the OS:

```bash
     sudo apt update -y && sudo apt upgrade -y 
```

- Install OpenSSH Server (if needed):

```bash
     sudo apt install openssh-server
```

- Enable and start OpenSSH Server:

```bash
     sudo systemctl enable ssh
     sudo systemctl start ssh
```

- Verify OpenSSH server status:

```bash
     sudo systemctl status ssh
```

- Allow SSH access in the firewall:

```bash
     sudo ufw allow ssh
```

- Set the time zone:

```bash
     sudo timedatectl
     set-timezone US/Central
```

- Verify date and time:

```bash
     date
     timedatectl
```

- Make user account locveeam:

```bash
     sudo useradd locveeam --create-home -s /bin/bash
```

- Set locveeam's password:

```bash
     sudo passwd locveeam
```

- Add locveeam to sudo group:

<!-- We will be using this for a short while and then will revert back. -->

```bash
     sudo usermod -a -G sudo locveeam
```

- Set root's password:

```bash
     sudo passwd root
```

- Exit session:

```bash
     exit
```

- Test all 3 accounts.

- Modify SSH configuration file to allow access to your standard and locveeam user accounts:

```bash
     sudo vi /etc/ssh/sshd_config
```

- Add the following line:
  - AllowUsers standarduser1 locveeam

- **Be sure to use the TAB key between all 3 words!**

- Save the file and exit the text editor

- Restart SSH Server:

```bash
     sudo systemctl restart sshd
```

- Exit session:

```bash
     exit
```

- Test SSH login:

This is needed for Veeam. I'll be using Windows CMD to do this.

**Note:** If you are having trouble with the SSH login, be sure you are not trying to remote into the iDrac IP! It needs to be the IP of eno1 (NIC1).

- SSH into the server:

```bat
     ssh csaadmin@<IP>
```

**Note:** If it fails, be sure SSH is enabled and that the SSH rules are in the firewall.

- Press Y to accept the SSH fingerprint.

- Verify that the veeamrepo mount point shows:

```bash
     df -h
```

- Verify that the veeamrepo mount point uses xfs:

```bash
     sudo blkid /dev/sd##
```

- Assign locveeam as the owner of the veeamrepo mount point and then verify changes:

```bash
     sudo chown -R locveeam:locveeam /mnt/veeamrepo
```

- Set permissions on the veeamrepo mount point and then verify changes:

```bash
     sudo chmod -R 755 /mnt/veeamrepo
     ls -alh /mnt/veeamrepo
```

- Create firewall rules for Veeam:

**Note:** Do a replace all with <IP> with the VBR server’s IP and paste into the active session. For instances with multiple Veeam backup servers, you will need to repeat this for each additional backup server. Just replace the IP with the other IP and run it.

```bash
     sudo ufw allow from <IP> to any port 2500 proto tcp comment 'incoming Veeam rule - transmission channels - modified' && sudo ufw allow from <IP> to any port 2501 proto tcp comment 'incoming Veeam rule - transmission channels - modified' && sudo ufw allow from <IP> to any port 2502 proto tcp comment 'incoming Veeam rule - transmission channels - modified' && sudo ufw allow from <IP> to any port 2503 proto tcp comment 'incoming Veeam rule - transmission channels - modified' && sudo ufw allow from <IP> to any port 6160 proto tcp comment 'incoming Veeam rule - installer service - modified' && sudo ufw allow from <IP> to any port 6162 proto tcp comment 'incoming Veeam rule - data mover service - modified' && sudo ufw allow from <IP> to any port 6166 proto tcp comment 'incoming Veeam rule - controlling port for rpc calls - modified' && sudo ufw allow from <IP> to any port 6182 proto tcp comment 'incoming Veeam rule - control channel - modified' && sudo ufw allow out to <IP> port 2500 proto tcp comment 'outgoing Veeam rule - transmission channels - modified' && sudo ufw allow out to <IP> port 2501 proto tcp comment 'outgoing Veeam rule - transmission channels - modified' && sudo ufw allow out to <IP> port 2502 proto tcp comment 'outgoing Veeam rule - transmission channels - modified' && sudo ufw allow out to <IP> port 2503 proto tcp comment 'outgoing Veeam rule - transmission channels - modified' && sudo ufw allow out to <IP> port 6160 proto tcp comment 'outgoing Veeam rule - installer service - modified' && sudo ufw allow out to <IP> port 6162 proto tcp comment 'outgoing Veeam rule - data mover service - modified' && sudo ufw allow out to <IP> port 6166 proto tcp comment 'outgoing Veeam rule - controlling port for rpc calls - modified' && sudo ufw allow out to <IP> port 6182 proto tcp comment 'outgoing Veeam rule - control channel - modified'
```

- Verify that it created them (Rules updated results).

- Enable and start the OS firewall:

```bash
     sudo ufw enable
```

**Note:** Since you added the ssh rules earlier, it should not kick you off. If it did, verify that the rules were added!

- Verify that the firewall is now running:

```bash
     sudo ufw status verbose
```

- Reload the firewall:

```bash
     sudo ufw reload
```

## 4. VBR configuration:

- We are now ready to add the server to Veeam Backup and Replication. You need to leave this session minimized.

- Open VBR and log into it.

- Backup Infrastructure > Managed Servers > Add Server

- Select Linux

- Enter the IP address of the Linux server. You can add "Hardened Repository" to the description if you wish.

- Add… > single-use credentials for hardened repository…

- Input locveeam and its password > OK

- It will now ask you about the SSH fingerprint. Click Yes.

- Verify that the account now shows.

**Note:** If it has trouble detecting its components, be sure that the locveeam user has sudo and SSH access, and that the SSH service was restarted after its configuration file was modified!

- The component Transport (and any others that it needs) will be installed now.

- The server will now be added.

- Verify information

- Once added, the server should now show in the Managed Servers module.

- Backup Repositories > Add Repository

- Choose 'Direct attached storage'

- Choose Linux

**Note:** With v12, it adds a new option for the hardened repository. **Be sure to select that instead if it gives it!**

- Name: Hardened Repository > Next

- Populate > Select /mnt/veeamrepo > Next

- Populate (it should then be able to get its sizes)

- Check "Use fast cloning…"

- Check "Make recent backups…" and set for 7 days

- Go with defaults for directories and make sure they actually exist.

- It will now add the backup repository.

- Verify the information

- The repository should now show under Backup Repositories.

- Home > Backup Job > Virtual machine…

- Name: <LINUX_HOSTNAME>_DLY_Hardened

- Add… > localhost > Add

- **Select - Hardened Repository for the backup repository**

- Set Retention Policy to 14 days

- Advanced > check Incremental (recommended) > check the synthetic full backup option (every Saturday) > OK

The reason we select this (instead of active full) for Linux backup servers is that synthetic can take advantage of the XFS file system to be much more efficient.

- Check "Run the job…" > set Daily at 2 AM everyday

- Verify the information > uncheck "Run the job…" 

- After the backup job is created, it should now show under Backup Jobs. **Be sure its target is the Hardened Repository!**

- VBR configuration is now finished. Stop here for the day and let the backup job run overnight. 

Why did we not let it run immediately shortly ago? We need to make sure it doesn't butt heads with their other jobs.

**Note:** If for whatever reason the backup job is taking forever to run, be sure the server isn’t running a 100 M connection (NIC1), but instead a 1 G connection.

- The next day make sure the job was successful. 

- We will now need to test the immutability of the backups.

- Backups > Disk

- Right click the hardened backup > Properties

- Be sure it shows the backup file as immutable.

- Now to actually test this, you will need to try and delete the backup. **Do not do this on existing data though, unless it’s a new backup chain you’ve made by running the immutable backup job initially.**

- Right click the backup > Delete from disk

- If successful, it should not be able to delete the backup, and will instead tell you when it can be deleted.

- The Veeam configuration is now finished. You may close out of it. 

## 5. Post Veeam configuration:

**Note:** Don't do this section over SSH. Do it instead locally or at a console level.

- Verify permissions on the backup directories and files:

```bash
     sudo ls -alh /mnt/veeamrepo/backups
     sudo ls -alh /mnt/veeamrepo/backups/<HOST_NAME>-Hardened-Backup/
```

- Verify immutable flag on backup file: 

```bash
     sudo lsattr -l /mnt/veeamrepo/backups/<HOST_NAME>-Hardened-Backup/
```

- Verify unattended-upgrades package status:

```bash
     sudo systemctl status unattended-upgrades
```

- If for whatever reason it shows it not loaded, run "sudo apt install unattended-upgrades" to install it and then re-run the status check.

- Install package update-notifier-common (if missing):

```bash
     sudo apt install update-notifier-common
```

- Set up automatic updates and allow daily reboots as needed at 11 AM:

```bash
     sudo vi /etc/apt/apt.conf.d/50unattended-upgrades
```

- Make the following adjustments and be sure each are verbatim to the configuration:
  - Uncomment out/adjust:
    - Unattended-Upgrade::Automatic-Reboot "true";
  - Uncomment out/adjust:
    - Unattended-Upgrade::Automatic-Reboot-WithUsers "true";
  - Uncomment out/adjust:
    - Unattended-Upgrade::Automatic-Reboot-Time "11:00";
- Save the changes

- Modify the other config file:

```bash
     sudo vi /etc/apt/apt.conf.d/20auto-upgrades
```

- Verify that both entries show 1

- Stop and disable SSH Server: 

```bash
     sudo systemctl stop ssh.service
     sudo systemctl disable ssh.service
```

- Remove sudo access from locveeam user account: 

```bash
     sudo deluser locveeam sudo
```

- Remove SSH rules (ipv4, ipv6) from firewall, verify changes, and reload it:

```bash
     sudo ufw status numbered
     sudo ufw delete #
     sudo ufw status numbered
     sudo ufw delete #
     sudo ufw status verbose
     sudo ufw reload
```

- Exit session:

```bash
     exit
```
