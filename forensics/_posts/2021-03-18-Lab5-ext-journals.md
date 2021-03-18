**This lab is not entierly finished. All flags have been found but some awnsers on the awnser sheet were not entierly correct. *This message will be deteted* when the anwser sheet has been filled in.** 

In this lab we'll simulate the whole forensics process with a linux file system and we'll take a look at the ext journals.

# disclaimer
Not all tools/files/VMs/networks will be available. These are notes taken for labs duiring a course, so some stuff is copyright protected and can't be shared. Where possible I'll try to link the resources.

# Tools we can use:
* fls
* istat
* blkcat
* file
* dumpe2fs
* fsstat
* xxd
* dd
* jls
* ext3grep
* ext4magic
* debugfs
* parted
* losetup
* lsblk


# Acquisition
I can't stress enough: The acquisition phase, this is very important!!!
## Stop the service & mask it
``sudo systemctl stop udisks2.service && sudo systemctl mask udisks2.service``
## Add disk to machine
Add disk to machine
## rescan scsi disk:
Find host: ``ls /sys/class/scsi_host``.
scan scsi disk: `` sudo su`` & ``echo "- - -" > /sys/class/scsi_host/host[x]/scan`` where [x] is the host.
## verify disk is online:
``lsblk``
## check metadata:
``sudo hdparm -I /dev/sdb``
## Make md5sum:
``sudo md5sum /dev/sdb`` (here: ``d4f26090e50ffdfcbb671aadb862e021``)
## Make backup image of the full disk using ``dd``:
``sudo dd if=/dev/sdb of=sdb-backup.img status=progress``
## Verify backup is the same:
``md5sum ./sdb-backup.img`` (here: ``d4f26090e50ffdfcbb671aadb862e021`` - this is correct)
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
``sudo md5sum /dev/loop0`` (here: ``d4f26090e50ffdfcbb671aadb862e021`` - this is correct)
## mount the loop device
Find loop devices with partitions: ``ls -l /dev/ | grep loop0``
mount loop device:``sudo mount /dev/loop0 /mnt/loop0`` -> did not work; this is because of the different partitions, each partition will have to be mounted seperatly.

``sudo mount /dev/loop0p1 /mnt/loop0p1`` & ``sudo mount /dev/loop0p2 /mnt/loop0p2`` & ``sudo mount /dev/loop0p3 /mnt/loop0p3`` & keep doing this till all partitions have been mounted.

# ---------------------------------

# Analysis of the suspect's disk.
looking for free space on the disk:
``lsblk``, ``fdisk``, ``parted`` are good tools to use for this. I like to use ``parted`` for this. It gives a bit more information. we can see that the only partition on this disk is ``1073M`` big and is an ext3 filesystem.
* ``sudo parted /dev/loop0`` -> ``print all``

### ------ v REVISIT v ------ (ex. 3)
Before we can really start with finding flags or recovering deleted files let's start with a bit applied theoretics and find an answer on following questions:
* Each block group has a ‘data/block bitmap’ and an ‘inode’ bitmap. How many blocks are each of these in size?
  * ``sudo dumpe2fs /dev/loop0p1 | less``
  * Both bitmaps are only 1 block per block group
* How many blocks are there in each block group (except for the last one)? 
  * -> ``Blocks per group:         32768``
* Inspect with dumpe2fs the size of one inode. How many inodes thus fit within one block of this file system?
  * 16: ``Inodes per group:         8160`` / ``Inode blocks per group:   510``
  * or: 16: ``Block size:               4096`` / ``Inode size:               256``
* You can verify with the ‘imap \<inode>’ command within debugfs /dev/loop0p1 indeed which inodes are located on the same block.
  * ``sudo dbugfs /dev/loop0p1`` -> ``imap <16>`` for inode 16
* Inspect with dumpe2fs or fsstat the number of inodes per block group and the number of blocks that are used to save these inodes per block group. 
  * There are ``_FILL ME IN LATER_ `` inodes per block group:  ``16 inodes in one group`` * ``_FILL ME IN LATER_``
* Thus each block group uses how many blocks to save these inodes?
  * 510: ``Inode blocks per group:   510``


### ------ ^ REVISIT ^ --------
Inspect the file system of the suspect's disk structure using dumpe2fs or fsstat.
using: ``fsstat /dev/loop0p1``
* What is the block size?
  * 4096 bytes: ``Block Size: 4096``
* How many block groups are there?
  * 8: ``Number of Block Groups: 8``
* Which block groups keep a (copy of the) superblock?
  * 0,1,3,5,7 -> found these by useing the search function in ``less``

### ------ v REVISIT v ------ (ex. 5)
* What is the block number of the block bitmap in group 6?
  * 196608: ``Data bitmap: 196608 - 196608``
* Use ``blkcat`` and ``xxd`` to inspect the content of this block. How many blocks are indicated to be ‘allocated’, or simply put: ‘in use’ (i.e. how many bits are set to 1)? 
  * ``sudo blkcat /dev/loop0p1 196608 | xxd | head`` -> 64*F 
  * Miss: 16 blocks (each pair of FFFF = 1 block?)

### ------ ^ REVISIT ^ --------
Again, use dumpe2fs and have a look at the information about the journal.

**Theory: 3 ways to use the journal:**

    1. ordered mode (default): Metadata (no content) is recorded after a write is successful.
    2. writeback mode ('unordered'): metadata (no content) is recorded, but the journal does not wait for the write to be completed.
    3. Journal mode ('data mode'): complete journaling of metadata AND actual content.


* What is saved into the journal? 
  * default config (so only meta data): ``Filesystem features:      has_journal (...)``
* What inode number is used for the journal? 
  * 8: journal is always saved on inode 8
* In what block group is it thus located?
  * the first block group: ``istat /dev/loop0p1 8 | less``-> ``Group: 0``
* What is the first data block which is used by the journal? (istat)
  * 66048: ``istat /dev/loop0p1 8 | less`` -> look at the first number
* How many blocks are being used for the journal? 
  * journal size: 16MB -> 16777216bytes/4096bytes(size of a block)= 4096 blocks for the journal: in ``istat``: ``size: 16777216``

----------
# On to flags
* Flag 1: Looking through the journal we can find the name of a file which is the flag: ``FLG-EXT3-2012``
  * command used:
  * ``icat /dev/loop0p1 8 | xxd | grep -A4 FLG``
  * or we can do it easier: ``fls /dev/loop0p1``
* Flag 2: 
  * ``fls /dev/loop0p1`` -> found file with inode 12
  * ``debugfs /dev/loop0p1 -R "imap <12>"`` -> block 67
  * ``ext3grep /dev/loop0p1 --journal --block 67``
  * taking a look at the first block what's written on the Inode 12 position: ``ext3grep /dev/loop0p1 --block 66052 | less`` we can read ``Direct Blocks: 583``
  * Using the ``blkcat /dev/loop0p1 583 | xxd | head`` command we can read the content of the file, this is a clear text file and can read the flag: ``FLG-99557891``
* Flag 3: 
  * ``fls /dev/loop0p1`` -> found file with inode 14
  * ``debugfs /dev/loop0p1 -R "imap <14>"`` -> block 67
  * ``ext3grep /dev/loop0p1 --journal --block 67``
  * going from last to first looking for what the journal has to say before inode 14 was deleted we get: ``Direct Blocks: 33345 33346 33347 33348 33349``
    * command ``ext3grep /dev/loop0p1 --block 66305 | less`` -> this is the 30th journal transaction 
  * * Using the ``blkcat /dev/loop0p1 33345 | xxd | head`` command we can read the content of the file; This is not a clear text file so we'll have to print it to a directory as a file and run the ``file`` command. Or you can identify this file by the magic bytes you can see and this is a JFIF file - which is a picture format.
  * using ``blkcat /dev/loop0p1 33345 >> flag.jpg`` we can recover blocks of the file. I ran that command for each block and it made the file piece by piece.
    * Flag: ``FLG-26008392`` 
-------
# Bonus
Going back to to previous lab search through the journal to associate **what block number the data of the deleted files were on**, on both the ext3 & ext4 partition:
* Ext3 (/dev/loop0p2): 
  * ``fls /dev/loop0p2`` -> inode 12
  * ``debugfs /dev/loop0p2 -R "imap <12>"`` -> block 263
  * ``ext3grep /dev/loop0p2 --journal --block 263`` -> ``131334`` & ``131344``
  * ``ext3grep /dev/loop0p2 --block 131334 | less`` -> inode 12 -> direct block: ``1537``
* Ext4 (/dev/loop0p3): 
  * ``fls /dev/loop0p3`` -> inode 12
  * ``debugfs /dev/loop0p3 -R "imap <12>"`` -> block 292
  * ext4 is **NOT** ext3 so we use another program:
    * ``ext4magic /dev/loop0p3 -T`` -> 292 -> journal 5
  * ``_FILL ME IN LATER_``