
## 1. Introduction and Prerequisites

This guide will walk you through setting up a bcachefs filesystem on Debian Sid (aka rolling release). The setup uses a mix of nvme and spinning disk for a flexible storage solution suitable for various use cases, including hosting large and small files, Docker containers, and VMs.

I wrote this guide essential for myself as a reference, you might find it useful if you need to get up to speed quickly.  The device names are the ones that I have used, you will need to map your own devices 'sudo blkid' and 'sudo lsblk'. 

Prerequisites:
- Debian Sid (Debian 6.9.12-1 or newer) installed
- Root or sudo access
- The following devices available:
  - nvme0n1p1 (1.8T) for cache
  - nvme0n1p2 (1.8T) for primary storage
  - sda (2.7T) for bulk storage
  - sdb (2.7T) for bulk storage

## 2. Installing necessary software

First, update your system and install bcachefs-tools:

```bash
sudo apt update
sudo apt upgrade
sudo apt install bcachefs-tools
```

## 3. Preparing the devices

Ensure the devices are not mounted and don't contain any important data:

```bash
sudo umount /dev/nvme0n1p1
sudo umount /dev/nvme0n1p2
sudo umount /dev/sda
sudo umount /dev/sdb
```

## 4. Creating and Configuring the bcachefs Filesystem

We'll create the bcachefs filesystem, set the labels, and configure the caching behaviour all in one command:

```bash
sudo bcachefs format \
--label=nvme.cache /dev/nvme0n1p1 \
--label=nvme.tier0 /dev/nvme0n1p2 \
--label=hdd.bulk1 /dev/sda \
--label=hdd.bulk2 /dev/sdb \
--compression=lz4 \
--foreground_target=nvme.tier0 \
--promote_target=nvme.cache \
--metadata_target=nvme \
--background_target=hdd
```

This single command does the following:

- Creates a bcachefs filesystem spanning all four devices
- Labels each device appropriately
- Sets LZ4 compression for the entire filesystem
- Sets nvme.tier0 (nvme0n1p2) as the foreground target for new writes
- Sets nvme.cache (nvme0n1p1) as the promote target for caching reads
- Sets devices labeled with "hdd" as the background target for data migration

**It's crucial to note:**

1. The labels **must be** specified before the additional elements like foreground_target, promote_target, and compression.
2. This format command should only be **run once**, as each run will remove the previous format data.

> [!NOTE] Check the devices have been properly defined from the output.
> 
	  metadata_target:          nvme
	  foreground_target:       nvme.tier0
	  background_target:     hdd
	  promote_target:           nvme.cache

This setup implements a writeback caching configuration where:

- New writes go to the faster NVMe tier0 device (nvme0n1p2)
- Reads are cached on the NVMe cache device (nvme0n1p1) if not already there
- Data is moved to the slower HDD devices in the background
- The rebalance thread will continually move data to the HDD devices
- When data is moved to the background target, the copy on the original device will be kept but marked as cached
- Cached data is evicted as needed in LRU (Least Recently Used) fashion

*Note: This was a pita to get done in a single clean command.*
## 5. Mounting the filesystem and setting up fstab

Create the mount point:

```bash
sudo mkdir /datadepot
```

Mount the filesystem:

```bash
sudo mount -t bcachefs /dev/nvme0n1p2:/dev/nvme0n1p1:/dev/sda:/dev/sdb /datadepot
```

To make this mount persistent, grab UUID and add the following line to /etc/fstab:

```bash
lsblk -f | grep bcachefs
```

Add the entry to the /etc/fstab, using the filesystem-uuid from the lsblk command:

```bash
UUID=<filesystem-uuid> /datadepot bcachefs defaults,device=/dev/nvme0n1p2,device=/dev/nvme0n1p1,device=/dev/sda,device=/dev/sdb 0 0
```

Have systemd reload the mount points, to pick up the entry:

```bash
systemctl daemon-reload
```


> [!NOTE] Verify with $ sudo mount | grep bcachefs
> /dev/nvme0n1p1:/dev/nvme0n1p2:/dev/sda:/dev/sdb on /datadepot type bcachefs (rw,relatime,compression=lz4,metadata_target=nvme,foreground_target=nvme.tier0,background_target=hdd,promote_target=nvme.cache)

## 6.  Довіряй, але перевіряй (Trust but Verify)

Here are the reliable ways to check your bcachefs filesystem configuration:

1. To view the superblock information for the filesystem:
```bash
sudo bcachefs show-super /dev/nvme0n1p1
```

This will show information about the filesystem, including the UUID, labels, and some configuration details.

2. To see usage statistics and some configuration details:
```bash
sudo bcachefs fs usage /datadepot
```

3. To check the current options set for the filesystem via the sysfs interface:
```bash
ls /sys/fs/bcachefs/*/options/
```

This will list all available options. You can then check specific options like this:
```bash
cat /sys/fs/bcachefs/*/options/compression 
cat /sys/fs/bcachefs/*/options/foreground_target 
cat /sys/fs/bcachefs/*/options/promote_target 
cat /sys/fs/bcachefs/*/options/background_target
```

More than one bcachefs on the host use the filesystem's UUID instead of (*) 

4. To see a list of devices in your bcachefs filesystem:
```bash
lsblk -f | grep bcachefs
```
## 7. Tuning bcachefs

Modify bcachefs options with `set-options`, for example set the journal locations:
```bash
sudo bcachefs set-option /datadepot journal_seq_blacklist_device=1,3,4
```

Adjust the IO scheduler for NVMe drives:

None persistent
```bash
echo none | sudo tee /sys/block/nvme0n1/queue/scheduler
```

Persistence (survives system restart)

Create a new udev rule file:
 ```bash
sudo vi /etc/udev/rules.d/60-scheduler.rules
```

And add this entry (verify the nvme device name):
```bash
ACTION=="add|change", KERNEL=="nvme0n1", ATTR{queue/scheduler}="none"
```

Reboot or just reload udev rules:
```bash
sudo udevadm control --reload-rules 
sudo udevadm trigger
```

## 8. Basic usage and management

Check filesystem usage:

```bash
sudo bcachefs fs usage /datadepot`
```

Set attributes for specific directories (e.g., for example Docker data):

```
sudo mkdir /datadepot/docker sudo bcachefs setattr compression=zstd /datadepot/docker
```

## 8. Maintenance tasks

Schedule a monthly scrub:

```
(crontab -l 2>/dev/null; echo "0 2 1 * * /usr/sbin/bcachefs scrub /datadepot") | crontab -
```

To manually start a scrub:

```
sudo bcachefs scrub /datadepot
```

For rebalancing (if needed):

```
sudo bcachefs rebalance /datadepot
```

## 9. Monitoring the bcachefs system

Check the system journal for bcachefs-related messages:

```
sudo journalctl -t bcachefs
```

Monitor stats: basic bash script, will share once useful.

```
cat /sys/fs/bcachefs/*/internal/
```

## 10. Troubleshooting tips and common issues

Several useful 'known to work' If you encounter issues, check the system logs:

```
sudo journalctl -f
```

Run a filesystem check:

```
sudo bcachefs fsck /datadepot
```

If you need to force-mount the filesystem (use with caution):

```
sudo mount -t bcachefs -o degraded,force /dev/nvme0n1p2:/dev/nvme0n1p1:/dev/sda:/dev/sdb /datadepot
```

## 14. Next Steps and To Dos. 

Have a functional bcachefs filesystem set up on Debian Sid. 
Don't have in no particular order.

- Setting up quotas for users and apps.
- Creating and managing subvolumes (separating NFS. seperate NFS docker volumes)
- Tune / Tweak more caching policies
- Build out serviceability and reporting, for report alerting, etc.

FIN

