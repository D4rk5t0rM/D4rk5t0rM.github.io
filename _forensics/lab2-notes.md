# Forensics lab 2 - partitions:
fdisk - mbr
gdisk - GPT

Note: there also other partition styles, such as APM (Apple Partition Map) and BSD (Berkeley Software Distribution)

# Analysis
## What partition style do i havve?
``gdisk -l /dev/sda``
  ``Partition table scan:
  MBR: MBR only
  BSD: not present
  APM: not present
  GPT: not present``
  -> MBR
  
``fdisk –l /dev/sda`` # displays the partition table of the hard disk (sda)
``fdisk /dev/sda`` # opens a CLI wizard to manipulate the partition of sda


``fdisk -l /dev/sda``:
``Device     Boot    Start      End Sectors  Size Id Type
/dev/sda1  *        2048  6293503 6291456    3G 83 Linux
/dev/sda2       10487808 11536383 1048576  512M 82 Linux swap / Solaris
/dev/sda3        8390656 10487807 2097152    1G 83 Linux``

Are the partitions closely following each other? Hint: look at the last line of fdisk output. & What is the actual order of the partitions on the disk: which one comes first on the disk, middle
and last? Hint: Look at the start and end sectors.
* no they are not always following each other, between sda1 & sda3 it looks like there is one missing
* first /dev/sda1 then /dev/sda3 and last /dev/sda3.

## (g)parted
print free space: ``parted /dev/sda print free``

	``Model: VMware, VMware Virtual S (scsi)
	Disk /dev/sda: 16.1GB
	Sector size (logical/physical): 512B/512B
	Partition Table: msdos
	Disk Flags:

	Number  Start   End     Size    Type     File system     Flags
			32.3kB  1049kB  1016kB           Free Space
	 1      1049kB  3222MB  3221MB  primary  ext4            boot
			3222MB  4296MB  1074MB           Free Space
	 3      4296MB  5370MB  1074MB  primary  ext4
	 2      5370MB  5907MB  537MB   primary  linux-swap(v1)
			5907MB  16.1GB  10.2GB           Free Space``

How much free space is there at the end?
	10.2 GB

## nmls - media manager listing (sleuthkit suite -> TSK)
``apt-get update`` (may be needed)
``sudo apt install sleuthkit``

lists the partitions and the partition table entries
``mmls /dev/sda``

hex view of first 512 bytes (= the MBR):
``xxd -l 512 /dev/sda``

* The first bytes of the MBR are boot loader code. The last bytes are the partition table and the boot signature. What is the size of the partition table inside the MBR? ``64 bytes`` Have a look at the theory slides. What is the value of the first byte in this partition table? ``0x80``
* What are the last 2 bytes of the MBR? This is the boot signature. ``55aa``
* Have a look at the ASCII representation of the boot loader code inside the MBR. What boot loader do you recognize here? ``GRUB``

# Creation
## Where can you create an extra primary partiton of 6GB?
``root@debian:~# parted -l /dev/sda
Model: VMware, VMware Virtual S (scsi)
Disk /dev/sda: 16.1GB
Sector size (logical/physical): 512B/512B
Partition Table: msdos
Disk Flags:

Number  Start   End     Size    Type     File system     Flags
 1      1049kB  3222MB  3221MB  primary  ext4            boot
 3      4296MB  5370MB  1074MB  primary  ext4
 2      5370MB  5907MB  537MB   primary  linux-swap(v1)``
 -> immediatly after sda2; before 3 is only +/- 1GB free too little for 6GB. before is only 1MB free. after 2 there is more space free we have a disk that is 16GB big; the end of the last partition is 5907MB which leaves +/- 10GB free to do with as pleased after sda2
 
## what sector will the new partition start (when putting immediatly after the previous awnser):
``root@debian:~# fdisk -l /dev/sda
Disk /dev/sda: 15 GiB, 16106127360 bytes, 31457280 sectors
Units: sectors of 1 * 512 = 512 bytes
Sector size (logical/physical): 512 bytes / 512 bytes
I/O size (minimum/optimal): 512 bytes / 512 bytes
Disklabel type: dos
Disk identifier: 0x721a9151

Device     Boot    Start      End Sectors  Size Id Type
/dev/sda1  *        2048  6293503 6291456    3G 83 Linux
/dev/sda2       10487808 11536383 1048576  512M 82 Linux swap / Solaris
/dev/sda3        8390656 10487807 2097152    1G 83 Linux

Partition table entries are not in disk order.``	
	awnser: Sector: 11536384 (11536383 + 1 because it starts after the previous partition)

## creation of the partition:
``fdisk /dev/sda`` - now we are in interactive mode (m gives us the help menu).
	p: prints the partition table
	n: add a new partition
``n`` for new partition
``p`` for primary partition
``11536384`` 11536383 was already allocated so we have to add 1 to it.
``+6G`` size of the partition (last sector)
``p`` print partition table to see if it worked
``w`` to save & exit
``partprobe -s /dev/sda`` to solve the Re-reading error
``fdisk -l /dev/sda`` to verify again 

## creating even more partitions:
``fdisk /dev/sda``
``n`` -> error: ``To create more partitions, first replace a primary with an extended partition.``
	We are not able to create new partitions
	However we can delete a 1GB partition that's not in use; here we use ``cfdisk`` just to use another tool
Delete partition: ``cfdisk /dev/sda`` -> arrow keys to /dev/sda3 -> left arrow to [delete] -> <enter> -> see the free space went from 1G to 2G
Make new extended partition: go to free space after the newly 6GB created partition & click new -> 3.5GB -> extended
Make logical partition within the extended partition: go to freespace underneed the /dev/sda3: [NEW] -> 2G. do this also for a 800M partition
When all done do !!!![WRITE] -> yes!!!! -> quit -> !!!!``partprobe -s /dev/sda``!!!! -> inspect with a tool

How many partition *tables* are currently present on your system? Use the mmls tool to gain insight and you’ll know the answer...
	Awnser: 3

This EBR is the first sector of partition ``/dev/sda3``. The last two bytes are ``0x55aa``. This EBR ``has no`` boot code present in the first bytes.


To see the second EBR which includes the third partition table, we can't inspect the first sector of a sdaX partition, that trick only worked for the 'extended partition'. To see this EBR, we need to specify explicitly which sector we want to examine. You can use the -s argument of xxd to ‘skip’ a number of bytes from the start of /dev/sda . How many sectors should you skip (cfr mmls output)? Thus how many bytes?
we need to skip ``28315648`` sectors, thus ``14497611776`` bytes (#sectors *512)