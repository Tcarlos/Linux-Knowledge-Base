# Table of Contents

    1. Ubuntu Server installation with LVM (and RAID)
    2. LXC
    2.1 Install LXC
    2.2 Configure LXC
    2.3 Create LXC container
    2.4 Useful LXC commands
    2.5 Clone Containers
    2.6 Delete Containers
    3. Physical Volumes, Volume Groups and Logical Volumes
    3.1 Create Physical Volumes
    3.2 Create Volume Groups
    3.3 Extend Volume Groups
    3.4 Create Logical Volumes 
    4. Cachepools
    5. LVM Snapshots
    5.1 Create a snapshot
    5.2 Restore LV with a snapshot
    5.3 Extend a snapshot
    5.4 Reduce a snapshot
    5.5 Autoextend snapshots
    5.6 Use a snapshot script to make backups automatically

## 1. Ubuntu Server installation with LVM (and RAID)

<Enter small description here>

### 1. Installation

- Boot the Ubuntu server image from USB stick or from CD

## 2. LXC

<Enter small description here>

### 2.1 Install LXC

    sudo su
    apt-get update && apt-get upgrade -y
    apt-get install lxc
    reboot

### 2.2 Configure LXC

    sudo su

Manually allocate an UID and GID range to root in /etc/subuid and /etc/subgid by adding this line in both files:

    root:100000:65536

Add these same values in global configuration file /etc/lxc/default.conf, by adding these lines to it:

    lxc.id_map = u 0 100000 65536
    lxc.id_map = g 0 100000 65536

Set permissions andownerships:

    chown root:100000 /var/lib/lxc
    chmod 750 /var/lib/lxc

Enable changing memory and SWAP, by modifying a line in /etc/default/grub to:

    GRUB_CMDLINE_LINUX_DEFAULT="swapaccount=1"

Update grub and reboot:

    update-grub
    reboot

Add these lines to /var/lib/lxc/office1/config:

    lxc.cgroup.memory.limit_in_bytes = 512M
    lxc.cgroup.memory.memsw.limit_in_bytes = 1G

### 2.3 Create LXC container

    sudo su
    lxc-create -t download -n office1
    lxc-start -n office1
    lxc-stop -n office1

Enter the container and test networking:

    lxc-attach -n office1
    ping 8.8.8.8
    exit
    
### 2.4 Useful LXC commands

Gives detailed information about a specific container:

    lxc-info -n office1

Shows all containers and if they are running or not:

    lxc-ls --fancy
    
### 2.5 Clone Containers

### 2.6 Delete Containers

    lxc-destroy -n office4

## 3. Physical Volumes, Volume Groups and Logical Volumes

### 3.1 Create Physical Volumes
    
Create a pv on another disk:
    
    pvcreate /dev/md1
    
or inside an LV
    
    pvcreate /dev/VG1/cachedlv
    
or inside an LXC thinpool client
    
    pvcreate /dev/lxc/lxc_client2
    
### 3.2 Create Volume Groups 

    vgcreate VG1 /dev/md1/
    
    vgcreate VG2 /dev/VG1/cachedlv
    
### 3.3 Extend Volume Groups

Add another PV to an existing Volume Group

    vgextend VG1 /dev/md1

### 3.4 Create Logical Volumes

    lvcreate -n data -L 1GB /dev/VG1
    
### 3.5 Remove PVs, VGs, LVs

    lvremove
    vgremove
    pvremove
    pvremove /dev/md1 --force --force
    
If you ever encounter an unknown device with pvs, use this command


vgreduce --removemissing VG1

    
## 4. Thinpools

    pvcreate /dev/VG1/cachedlv  (autoset 9GB)
    Physical volume "/dev/VG1/cachedlv" successfully created 
    vgcreate VG2 /dev/VG1/cachedlv (using the 9GB of the PV)
    lvcreate -L 9G --thinpool thinpool1 VG2
    

    

### 4.1 Create a basic thinpools

### 4.2 Create LXC containers in a thinpool

    pvcreate /dev/VG1/cachedlv
  
    vgcreate lxc /dev/VG1/cachedlv

    lvcreate -l 75%FREE --type thin-pool --thinpool lxc lxc

    lxc-create -t download -n lxc_client1 -B lvm --fssize=2G

    vgchange -a y

## 4. Cachepools

Add the SSD raid to volume group VG1:

    pvcreate /dev/md1
    vgextend VG1 /dev/md1

    pvs
  
    PV         VG   Fmt  Attr PSize  PFree
    /dev/md0   VG1  lvm2 a--  19.98g 6.02g
    /dev/md1   VG1  lvm2 a--   4.99g 4.99g

Now make a cachepool on the SSD raid:

    lvcreate -L 100M -n cachemeta VG1 /dev/md1
 
    lvcreate -L 4G -n cachedata VG1 /dev/md1
 
    lvs
  
    LV        VG   Attr       LSize   Pool Origin Data%  Meta%  Move Log Cpy%Sync Convert
    bootlv    VG1  -wi-ao---- 952.00m                                                    
    cachedata VG1  -wi-a-----   4.00g                                                    
    cachedlv  VG1  -wi-a-----   9.31g                                                    
    cachemeta VG1  -wi-a----- 100.00m                                                    
    rootlv    VG1  -wi-ao----   2.79g                                                    
    swaplv    VG1  -wi-ao---- 952.00m                                                    

    lvconvert --type cache-pool --cachemode writethrough --poolmetadata VG1/cachemeta VG1/cachedata

    lvconvert --type cache --cachepool VG1/cachedata VG1/cachedlv

## 5. LVM Snapshots

### 5.1 Create a snapshot

    lvcreate -L 1GB -s -n [SNAPSHOT_NAME] [LV_PATH]

    lvcreate -L 1GB -s -n rootlv_snap /dev/VG1/rootlv

### 5.2 Restore LV with a snapshot

    lvconvert --merge /dev/VG1/data2_snap

Confirm data LV has been restored, one way is to mount and list

    mount /dev/VG1/data /data
    ls -la /data
    
Another way is checking diskspace

    df -Th
    
### 5.3 Extend a snapshot

    lvextend -L +1G /dev/VG1/rootlv_snap

### 5.4 Reduce a snapshot

    lvreduce -L +1G /dev/VG1/rootlv_snap

### 5.5 Autoextend snapshots

    nano /etc/lvm/lvm.conf

    snapshot_autoextend_threshold = 75
    snapshot_autoextend_percent = 20
    
If the snapshot volume reach 75% it will automatically expand the size of snap volume by 20% more.

### 5.6 Use a snapshot script to make backups automatically

        http://askubuntu.com/questions/424225/setting-up-lvm-snapshot-as-a-backup-restore-point-in-ubuntu

## 6. DRBD

- installation: apt-get install drbd-utils
- confirm: appearantly one needs to check pvdisplay and look up PE size and amount of available PE (free PVe)
- helpful links: https://www.howtoforge.com/setting-up-network-raid1-with-drbd-on-debian-squeeze and https://drbd.linbit.com/users-guide/
- Why DRBD: We also needed under the DRBD a thinpool because we want to snapshot the DRBD. On ttop of the DRBD we need a thinpool so we can snapshot the LXC lvs.
- short description: DRBD is like a networked RAID

clue: 

    lvcreate -n DRBDLV -V 1g --thinpool CachedRawDRBDVG/CachedRawDRBDThinpool
    
Where we create a thinpool with the name DRBDLV in VG CachedRAWDRBDVG on top of LV CachedRAWDRBDthinpool.

deleting previous made thinpool with

    root@livenode1:/home/office# lvremove thinpool1 VG2
    
appearantly there needs to be created an LV named CachedRAWDRBDthinpool on VG CachedRawDRBDVRG

    lvcreate -n CachedRawDRBDThinpool -l 428 CachedRawDRBDVG
    
    vgcreate CachedRawDRBDVG /dev/testvg/CachedRawDRBDPV

### 6.1 DRBD test case setup

So reversed engineered, this needs to be done:

- pvcreate CachedRawDRBDPV
- vgcreate CachedRawDRBDVG
- lvcreate -n CachedRawDRBDThinpool -l 428 CachedRawDRBDVG
- lvcreate -n DRBDLV -V 1g --thinpool CachedRawDRBDVG/CachedRawDRBDThinpool

Verify a completely free disk

    lsblk

create a pv on it

    pvcreate /dev/md1
    
create a vg on top of this

    vgcreate CachedRawDRBDVG /dev/md1
    
check free PE

    vgdisplay
    
This shows 1278 free extents. Now, create an LV end use all of the extents for it

    lvcreate -l 1278 -n cachedrawDRBDLV1 CachedRawDRBDVG
    
Confirm there is no more space on pv

    pvs
    








## 7. Bugs

## Bug #1

lxc lvm container: cant start containers after reboot, 

    lxc-start: bdev.c: mount_unknown_fs: 210 failed to determine fs type for '/dev/lxc/lxc_client2'
    lxc-start: conf.c: mount_unknown_fs: 525 failed to determine fs type for '/dev/dm-10'
    lxc-start: conf.c: setup_rootfs: 1280 failed to mount rootfs
    lxc-start: conf.c: do_rootfs_setup: 3713 failed to setup rootfs for 'lxc_client2'
    lxc-start: start.c: __lxc_start: 1155 Error setting up rootfs mount as root before spawn
    lxc-start: lxc_start.c: main: 344 The container failed to start.

### Bug #2

    lxc-start -n lxc_client1 -F
    lxc-start: conf.c: mount_rootfs: 873 No such file or directory - failed to get real path for '/dev/lxc/lxc_client1'
    lxc-start: conf.c: setup_rootfs: 1280 failed to mount rootfs
    lxc-start: conf.c: do_rootfs_setup: 3713 failed to setup rootfs for 'lxc_client1'
    lxc-start: conf.c: lxc_setup: 3795 Error setting up rootfs mount after spawn
    lxc-start: start.c: do_start: 699 failed to setup the container
    lxc-start: sync.c: __sync_wait: 51 invalid sequence number 1. expected 2
    lxc-start: start.c: __lxc_start: 1164 failed to spawn 'lxc_client1'

**update**

googled and got solutions to check idmap lines, done so already, or permissions, also.
Perhaps it's this https://github.com/lxc/lxc/issues/406, this suggests it could be a template matter.
A workaround could be to make a snapshot and merge right away to restore but to no avail
Do note however, that the new container has no problem. So it could be that the first two just dont work coz the whole LV setup got changed between client1/client2 versus client3.
**Update** make client4 and monitor the workings of client3 and client4. If after a few days there is still no error, we can discard this bug as a child of wrong config, a situation that is obsolete coz with a working system as i have now there are simply no problems.

## 8. TO DO

- fix the bugs of lxc_client1 and lxc_client2 (HANDLED, PROBABLY OSBOLETE ERROR)
- complete snapshot cheatsheet section
- include a couple of DBRD sections
- make restore snapshot of lxc thinpool containers work OR: RUN TESTCASE 3: include DRBD in the advanced LXC setup with thinpools and snapshots!
- https://gitlab.rimotecloud.com/tmarinus/mainline-rve/blob/GL1/docs/install.md implement Richards LXC config section1

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
