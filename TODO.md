## 8. TO DO

- fix the bugs of lxc_client1 and lxc_client2 (HANDLED, PROBABLY OSBOLETE ERROR)
- complete snapshot cheatsheet section
- include a couple of DBRD sections [DONE]
- make restore snapshot of lxc thinpool containers work OR: RUN TESTCASE 3: include DRBD in the advanced LXC setup with thinpools and snapshots!
- https://gitlab.rimotecloud.com/tmarinus/mainline-rve/blob/GL1/docs/install.md implement Richards LXC config section1
- TEST if metadata and datalv really can grow if needed!!

blockdevice explanation
Here we try to explain the increddible complex partitioning/RAID/DRBD/LVM cahe thinpool stuff. Dont do this, it will be explained step by step. but its a good reference for explanation.

1. First we need (at least) two HDDs and two SSDs fysically available. 
Lets call the HDDs /dev/sda and /dev/sdb and the SSDs /dev/sdc/ and /dev/sdd

2. now we want to combine these fysical disk in to RAID 1 (mirror) arrays so we can handle a device failure without major problems. We do this by combineing /dev/sda and /dev/sdb in to /dev/md0 and /dev/sdc and /dev/sdd to /dev/md1. So we are left with two redundant blockdevices: a giant slow /dev/md0 and a smaller fast /dev/md1

3. Now we need to make /dev/md0 a PV so we can use it as a source for LVM. After this we will create a VG on top of that. Within that VG we will create the crucial LVM's:
4. bootlv (/boot)
5. swaplv (SWAP)
6. rootlv (/)
7. cachedLV (this LV will be cached by the SSD)

8. Then we create from the /dev/md1 (fast SSDs) a PV and then ADD IT to the existing VG (that was only on /dev/md0 till now). We now can create specifically on the /dev/md1 two LV's that together form a cachepool. This cacheppol again will be combined with the chachedLV (on the slow /dev/dm0) so we have a truly cached LV.
9. This cached LV will become a PV of its own with a own VG. Within that VG we create a thinpool. Most of the space within that VG will be uses to create one LV.
10. the just created (cached) LV now will be the basis of DRBD.
11.the DRBD blockdevice now will become a PV and we put a VG on that.
12. On this VG we will create two LV's: a metadata and data partition. Together they form a thinpool that will be used for the creation of LXC LV's (for the containers)

**Notes:** we cant cache a thinpool so we first had to create a cached LV and then base a thinpool on top of that. We also needed under the DRBD a thinpool because we want to snapshot the DRBD. On ttop of the DRBD we need a thinpool so we can snapshot the LXC lvs. We dont want thinpool for our most important partitions because of possible filling of thinpool and the becomming unusable of the lvs in thath case.



