In this lab we'll simulate the whole forensics process with a linux file system instead of previous lab windows file system.

# disclaimer
Not all tools/files/VMs/networks will be available. These are notes taken for labs duiring a course, so some stuff is copyright protected and can't be shared. Where possible I'll try to link the resources.

# Acquisition
I can't be remineded enough of the acquisition phase. this is very important!!!
## Stop the service & mask it
``sudo systemctl stop udisks2.service && sudo systemctl mask udisks2.service``
## add disk to machine
## rescan scsi disk:
Find host: ``ls /sys/class/scsi_host``.
scan scsi disk: `` sudo su`` & ``echo "- - -" > /sys/class/scsi_host/host[x]/scan`` where [x] is the host.
## verify disk is online:
``lsblk``
## check metadata:
``sudo hdparm -I /dev/sdb``
## Make md5sum:
``sudo md5sum /dev/sdb`` (here: ``237a48efac10c0a00c91f1ddaa3358f0``)
## Make backup image of the full disk using ``dd``:
``sudo dd if=/dev/sdb of=sdb-backup.img status=progress``
## Verify backup is the same:
``md5sum ./sdb-backup.img`` (here: ``237a48efac10c0a00c91f1ddaa3358f0`` - this is correct)
## remove disk from list NOT DISCONNECT.
``sudo su`` & ``echo 1 > /sys/block/sd[x]/device/delete`` where [x] is the device in ``/dev/sd[x]``
## verify it's not in list anymore:
``lsblk``
## Disconnect disk
## restart the udisks2 service again:
``sudo systemctl unmask udisks2.service``
## find the loop device to mount
``sudo losetup -fPr sdb-backup.img --show`` (here: /dev/loop0)
## verify if loop device is correct:
``sudo md5sum /dev/loop0`` (here: ``237a48efac10c0a00c91f1ddaa3358f0`` - this is correct)
## mount the loop device
Find loop devices with partitions: ``ls -l /dev/ | grep loop0``
mount loop device:``sudo mount /dev/loop0 /mnt/loop0`` -> did not work; this is because of the different partitions, each partition will have to be mounted seperatly.

``sudo mount /dev/loop0p1 /mnt/loop0p1`` & ``sudo mount /dev/loop0p2 /mnt/loop0p2`` & ``sudo mount /dev/loop0p3 /mnt/loop0p3`` & keep doing this till all partitions have been mounted.

# ---------------------------------

# Analysing our own disk.
Before we'll try to analyze the suspect's disk; we'll play around with our own disk (you can follow this too. I'm using a Kali linux 20.4 machine; but this should be possible on any other linux distro)

# Discorvery of block devices
As I've done so many times before: ``lsblk`` shows a lot of information about the current block devices on the system and shows us, in my opinion, the most default information that you need in most cases.
Let's take a closer look to the ``MAJ:MIN``. We can see that there are two numbers here: ``8:0`` the MAJ or major number representes the device an dthe MIN or minor number represents the the partition of the device meaning that ``x:1`` is the first partition and we can verify this by looking at the number after the ``sda``. The first partition is ``sda1``.
Going back to the major number, this number tells us what type of device it is, with some [kernel documentation](https://www.kernel.org/doc/Documentation/admin-guide/devices.txt) we can go look for the representation of that number. Apperently the number ``8`` represents that this is a SCSI device.

Doing the same thing with the suspect's drive we can see that ``loop0`` has MAJ: ``7`` wich means it's a loopback device. but the different partitions have a MAJ number of ``259`` adn according to the [kernel documentation](https://www.kernel.org/doc/Documentation/admin-guide/devices.txt) this is a ``Block Extended Major``, this is some sort of 'catch all' partition

## Discovering the file system of the drives
The ``lsblk --fs`` command can show us some more info about what the file system of the device is. In a normal linux environment you can generally find a ext3 and ext4 filesystem.
Here we can see that the file system of our Kali linux is ``ext4``

## Block sizes
1) There are multiple ways of accuiring the blocksize of a disk.
If you only want to know the block size: ``sudo blockdev --getbsz /dev/sda1``

2) If you want to know a lot more info including inodes: ``sudo fsstat /dev/sda1``. I do reccomand running this with the ``less`` command at the end to avoid any chaos: ``sudo fsstat /dev/sda1 | less``
These commands tell us that our linux machine has a blocksize of ``4096`` (every file will take up atleast 4KB of space)
Some more interesting information you can read from this commad is that the root directory (eg: ``/``) is located on ``inode 2`` (which is normal and in most cases inode 2). Where as ``inode 11`` is the ``lost+found`` directory.
   - This is because inodes 1-10 are reserved for special purposes and the first ``user file`` is ``lost+found`` is ``inode 11``

3) Another command that gives A LOT of information and is like the fsstat command is the ``sudo dumpe2fs -o blocksize=4096 /dev/sda1`` command. This command lists the information in a 2 collumn view and is in my opinion a bit less user friendly. Here too I'd suggest ping the command into ``less``
   - ``sudo dumpe2fs -o blocksize=4096 /dev/sda1 | less``

## Find files according to their inode
1) The ``ls`` command also has a feature to list the inode number: ``-i``. If we run ``ls -li /`` we can also list the inode number of the root directory; all be it a bit more confusing then the previous ways.
2) When looking directly for a file you can use the ``ffind`` command: ``sudo ffind [device] [inode-number]``. 
   - ``sudo ffind /dev/sda1 11`` will give back the ``/lost+found`` directory

## Making partitions
There is a tool called ``mkfs.ext4`` that will make an ext4 partition, but if we look at the man page of this tool we can see that it is actually the ``mke2fs`` tool that will make the partition. 
If we look for the config file of this tool ``ls /etc | grep mke2fs`` we find the file ``/etc/mke2fs.conf`` in this file a small partition has a blocksize of ``1024 bytes`` and not ``512 bytes``

# ---------------------------------

# Capture the flag - Analysis of the suspect drive
## Disk layout

- Look for unused space:
  - ``lsblk`` or ``fdisk``->``p`` we can count the size of the drive this is almost 1024 bytes, we'll count the missing ``.3`` bytes as overhead. And we can count 3 partitions.
- The different partitions:
  - loop0p1: ``ext2``, UUID: ``f2659ce4-fb21-4cb5-888b-30074f6df512``, Block Size (BS): ``4096``
  - loop0p2: ``ext3``, UUID: ``64d3fe6f-aac6-4e30-9c87-2e57404da015``, Block Size (BS): ``1024``
  - loop0p3: ``ext4``, UUID: ``adaaf9fa-0f0c-440e-8706-eab3e31b4a44``, Block Size (BS): ``1024``

## Data recovery on ext2 FS
- Flag 1: go to the mounted directory and have a look around. There is a hidden file ``.top_secret.png`` wich has the flag: ``FLG-36458742`` this has inode: ``12`` (can be found by running ``ls -lia`` in the mounted directory).
- Flag 2: We've concluded that there are no more files to be found on the system that are not yet deleted. Now we look at the delted files: ``sudo fls -p /dev/loop0p1`` we can see 3 more files that are delted. because this is a ``ext2`` file system the inodes are deleted indicated by the ``* 0: [filename]``. Going through the deleted files one by one with the ``sudo istat /dev/loop0p1 [inode]`` command we can find multiple inodes that were in use once but according to the ``fls`` command are not in use. These indicate a delted file. Starting with Inode 14 (because inode 13 was the biggest inode and at the same time smallest inode that a user could create themselves). once I found that an Inode had data blocks i used the ``sudo blkcat /dev/loop0p1 [datablock]] | xxd | less`` command to read out these datablocks. in datablock ``4096`` (inode 14) there was a plain text file with the flag: ``FLG-20886378``
  - used commands:
  - ``sudo istat /dev/loop0p1 14``
  - ``sudo blkcat /dev/loop0p1 4096 | xxd | less``
- Flag 3: going further throught he inodes, inode 15 has a PDF file and multiple direct blocks. Here ``blkcat`` can only show us that it is a PDF file but can't recover the full file from it. Luckily ``icat`` can. using icat we can extract the file to another directory and open the pdf here we get a fun picture of the queen and the flag: ``FLG-77968854``
  - commands used:
  - ``sudo istat /dev/loop0p1 15``
  - ``sudo blkcat /dev/loop0p1 4096 | xxd | less`` - to view the PDF indicator
  - ``sudo icat /dev/loop0p1 15 > file.pdf``

## Data recovery on an ext3 file system
- Flag 1: another very easy one. After checking the file directory (should always do first). I ran the ``fls`` command on the partition and we can recover the flag in the filename of the deleted file. On ext3 file systems the inode is not deleted and here we have the flag: ``FLG-99658658`` with ``inode 12``. Please note that using the ``sudo istat /dev/loop0p2 12`` command to look for the datablocks, we can't find the datablocks anymore. from ext3 and above these links are deleted when a file is deleted from the system.
  - commands used:
  - ``sudo fls -p /dev/loop0p2``


## Data recovery on an ext4 file system
- Flag 1: all files were deleted. Using ``fls`` we see inode 12 with the filename but it has been deleted. ``istat`` confirms this, there are no blocks to see. Here we'll have to use ``data carving`` to find the data a very simple tool to search for plain text strings is the ``strings`` tool. This will also give us the flag. More in-depth data carving in another lab. Here we can just find the flag: ``FLG-61330010``.
  - commands used:
  - ``sudo strings /dev/loop0p3``

# Bonus round - 1 More flag
- Bonsu Flag: Using ``xxd`` on ``/dev/loop0p2`` we see a referense to a file called ``bonus-flag.txt`` looking a bit beore that, we can see the magic byte ``PK`` this is a ZIP file. Now that we know that we can start extracting the file. Using ``dd`` command we can give the input file as our loopback partition, we don't need an offset because the hidden file is at the start of the partition. Our output file we'll call ``bonus.zip`` and what is very important is giving the right blocksize; we've determined the blocksize already and that's ``1024`` here. By default ``dd`` will read 512 bytes, for us this is no porblem, we checked with ``xxd`` and the file stops before the 512bytes. When the zip file is recovered we can extract it and read the flag from the text file: ``FLG-55798123``
  - commands used:
  - ``sudo xxd -l 512 /dev/loop0p2``
  - ``sudo dd if=/dev/loop0p2 of=bonus.zip bs=1024``

# ---------------------------------
# Resources
* [Block device kernel documentation](https://www.kernel.org/doc/Documentation/admin-guide/devices.txt)