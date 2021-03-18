In this lab we'll take a look at the basics of a forensics investigation.

# disclaimer
Not all tools/files/VMs/networks will be available. These are notes taken for labs duiring a course, so some stuff is copyright protected and can't be shared. Where possible I'll try to link the resources.

# Commands:
`lsblk` show drives (block devices) & mountpath in tree form
`lsscsi` show scsi devices with names -didn't work on kali; worked on host linux
    - what file represents your hard disk where your kali OS is installed on?
        - sda
    - What block device file represents the partition where the root folder ( / ) is
installed on?
        - sda1
`blockdev` get physical information about a (virtual) disk
    - What block device file represents the partition where the root folder ( / ) is
installed on?
        - `blockdev --report`
		- `blockdev --getpbsz` physical block size per sector
		- `blockdev --getss` logical block size per sector
`udevadm info --query=all --name=/dev/sda` information about linux kernel detected about a disk
    - what is the value of ID_VENDOR?
        - No value
`usidkctl` sends notitifications to desktop applications when a USB device is available
    - check the status of udisk2 service: `systemctl status udisks2.service`
    - use `udisksctl` to get D-Bus info of the disk: `udisksctl info -b /dev/sda`

different tools for getting metadata info from Disk firmware. (might not give a lot of info on vmdks because these don't have a firmware)
    - `udevadm info --query=all --name=/dev/sdb`
    - `sudo /lib/udev/ata_id --export /dev/sdb` --did not work for me
    - `hdparm -I /dev/sdb` --best tool?
`dd if=/dev/sdb of=sdb-backup.img status=progress` - create a backup image of a disk
`losetup -fPr sdb-backup.img --show` - make a loop device of the image
`mount /dev/loop0 /mnt/loop` - mount the loop device

# Monitoring:
## Monitor attaching disks:
two commands to monitor what happens when you attach a disk:
`udevadm monitor` & `udisksctl monitor`

## Stop the services:
Stopping the services is important because if they are not stopped and the disk is mounted this will edit the md5 hash of the disk and this is a no-go in forensics.

Stop the service via: `systemctl stop udisks2.service`
prevent service from being starte via any other process: `systemctl mask udisks2.service`

Is it immediately detected by your Kali Linux OS? Check with the `lsblk` tool.
    - Yes it is detected as `sdb`. (`udevadm monitor` also detected the change when the disk was added)

Which of the lines above made it work for you? Or to ask the question in another way: on what
SCSI host (0,1 or 2) was the new disk added?
    - host2 (could alos be found in the `udevadm monitor` tool when the disk was added that this was the disk

# creating image backup and hash
create the hash of a disk:
    - `sudo md5sum /dev/sda` - the bigger the disk the longer this will take
Making a full backup of a disk:
    - `dd if=/dev/sdb of=sdb-backup.img status=progress`
- verify if the hash of the disk and the backup are the same; what is this value?
    - `44a0d9419931c9d1ecee920337646e09`

# Removing disk
remove the disk with the command: `echo 1 > /sys/block/sdb/device/delete` -- this did not work for me
unmask udisks2.service: `systemctl unmask udisks2.service`

# The Disk image
inspect the disk image: `losetup -fPr sdb-backup.img --show` -- might need to run in sudo
This will make a loop device (/dev/loop0). This device will work like a scsi/sata device (eg. /dev/sdb) this can be mounted too:
    - check the MD5 hash (`md5sum /dev/loop0`) of the new loop device, is it the same?
        - yes: `44a0d9419931c9d1ecee920337646e09`

## Mounting
Now you can mount the device: `sudo mount /dev/loop0 /mnt/loop` (a new directory may be needed to be created)
    - what file is present?
        - a file called `hello.txt`

when attaching a device, the hash will change after the unmount. This is why we don't attach the real drive but we make a copy of it and can use that copy. 
    - new hash: `5e583c903c2432ef23d0154452243e7a`
