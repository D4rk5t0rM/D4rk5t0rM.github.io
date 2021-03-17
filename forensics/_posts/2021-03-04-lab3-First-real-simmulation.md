In this lab we'll simulate the whole forensics process and try to recover some flags.

# disclaimer
Not all tools/files/VMs/networks will be available. These are notes taken for labs duiring a course, so some stuff is copyright protected and can't be shared. Where possible I'll try to link the resources.

# Acquisition
## Stop the service & mask it
``sudo systemctl stop udisks2.service && sudo systemctl mask udisks2.service``
## add disk to machine
## rescan scsi disk:
`` sudo su`` & ``echo "- - -" > /sys/class/scsi_host/host[x]/scan`` where [x] is the host. To find the host: ``ls /sys/class/scsi_host``.
## verify disk is online:
``lsblk``
## check metadata:
``sudo hdparm -I /dev/sdb``
## Make md5sum:
``sudo md5sum /dev/sdb`` (here: ``22c2d2ab21a26813ba6479d4874bdcb5``)
## Make backup image of the full disk using ``dd``:
``sudo dd if=/dev/sdb of=sdb-backup.img status=progress``
## Verify backup is the same:
``md5sum ./sdb-backup.img`` (here: ``22c2d2ab21a26813ba6479d4874bdcb5`` - this is correct)
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
``sudo md5sum /dev/loop0`` (here: ``22c2d2ab21a26813ba6479d4874bdcb5`` - this is correct)
## mount the loop device
``sudo mount /dev/loop0 /mnt/loop0`` -> did not work; this might be because the different partitions.

``sudo mount /dev/loop0p1 /mnt/loop0p1`` & ``sudo mount /dev/loop0p2 /mnt/loop0p2`` to mount both partitions

# --------------------------------------
# Full Disk Analysis
## Have a look at your backup image and list all the active partitions therein using fdisk, parted
and/or mmls (see previous lab)
* commands:
``sudo fdisk -l /dev/loop0``
``sudo mmls /dev/loop0``
* How many active partitions can you find?
Loop0 has 2 partitions
* What file systems are they using?
loop0p1: FAT16; loop0p2: HPFS/NTFS/exFAT (probably NTFS)
* Any other useful information from these tools?
  - mmls shows 2 unallocated spaces before and after the 2 partitions

# -----------------------------------------
# FAT Partition analysis
There are 3 flag on the FAT partition.

tools used: `` xxd, sleuthkit (fsstat, fls, icat, mactime)``
## How many files do you expect to see when using fsstat?
* 5 files:
``sudo fsstat /dev/loop0p1``

``FAT CONTENTS (in sectors):
336-351 (16) -> EOF
8704-8719 (16) -> EOF
8720-8735 (16) -> EOF
8768-8815 (48) -> EOF
8816-9343 (528) -> EOF``


## analysis of the files currently present
### Look at the FAT in a hex editor, such as xxd. How many files are present?
* Hint: use the data for the offset and length from fsstat output. Remember xxd works
with bytes not sectors! ``xxd –s offset[Bytes] –l length[Bytes] FILE_OR_PARTITION | less``
``sudo xxd -s [offset*sector] -l [length*sector] /dev/loop0p1 | less``
### Mount the FAT partition and have a look at the files still present. Does the number of files match
* with your earlier expectation, based on fsstat output?
Yes, the files are in $RECYCLE.BIN; file $RKWA1KF.pdf in here has the flag: ``FLG-62153879``

## data recovery of deleted files
### Use fls (look at manpage) to recover other artifacts from the partition
* ``fls -p /dev/loop0p1`` shows a lot more files that are deleted.
* ``sudo fls -m ./ /dev/loop0p1`` (for some reason ``./`` is needed
* Making a timeline: ``sudo fls -m ./ /dev/loop0p1 | mactime -b``

### Restoring files:
``sudo icat -r /dev/loop0p1 11 > nsa.jpg`` -> ``FLG-46874687``
* What files can be restored?
	ExamQuestions.txt,  TopSecret.txt, NsaPoster2019.jpg, Dilbert.pdf
* What files are destroyed?
	ExamQuestions.txt, Dilbert.pdf
* Extra: Can you tell what was the cause for not being able to recover certain files?
	overwritten by other files
Second flag:
``sudo testdisk`` > create > loop0 > continue > EFI GPT > Analysis > Quick search > partition with start 128 > ``p`` > Myfiles > Somefile.txt > ``c`` > ``c`` > flag: ``FLG-35168547``


# NTFS Partition Analysis
## analysis of the files currently present
### Have a look at the filesystem details using fsstat on the partition
``Size of Index Records: 4096 bytes``
### Mount the NTFS partition and have a look at the files still present. Can you find any interesting artifacts? (Note: in case of ntfs-3g-mount error, use the -o loop option.) Try to explain them.
there are a few files; nothing too interesting; one working PDF; 2 folders in the system volume informations there is mention of a ``user-pc``. The PDF does have a author: ``Hendrik`` and soe extra metadata like a title: ``dt171205.gif``; the producer is ``Microsoft: Print To PDF`` and there is a modification date and a creation date.

## data recovery of deleted files
* Recovery of files:
``sudo icat -r /dev/loop0p2 35-128-3 > ./NSA.txt:ZoneIdentifier.txt`` in zoneid there is a flag: ``FLG-96347867`` that would point to a website.
* second flag:
Door gebruik te maken van ``xxd`` & ``grep`` kon ik de flag vinden; dit alleen maar doordat ik voorafgaande informatie hierover had dat het te maken had met "loremIpsum":
``sudo xxd /dev/loop0p2 | grep lorem`` -> offset hex 27920 bytes; -> hex to dec: 162080 -> ``sudo xxd -s 162080 /dev/loop0p2 -l 100000 | grep FLG`` -> offset hex 298ec bytes -> hex to dec: 170220 -> ``sudo xxd -s 170220 /dev/loop0p2 -l 512`` -> flag:``FLG-65441658`` spread over 2 lines
# Bonus:
``sudo testdisk`` > create > loop0 > continue > EFI GPT > Analysis > Quick search > partition with start 1228928 > ``p`` > EasterEgg.jpeg > ``c`` > ``c`` > flag: ``FLG-68691246``
