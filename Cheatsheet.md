# Table of Contents

1. Ubuntu Server installation with LVM (and RAID)
2. LXC
3. Physical Volumes, Volume Groups and Logical Volumes
4. Cachepools
5. LVM Snapshots
6. DBRD

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
    
## 4. Thinpools

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

### 5.5 Autoextend snapshots

### 5.6 Use a snapshot script to make backups automatically

## 6. DBRD

## 7. Bugs

lxc lvm container: cant start containers after reboot, 

lxc-start: bdev.c: mount_unknown_fs: 210 failed to determine fs type for '/dev/lxc/lxc_client2'
lxc-start: conf.c: mount_unknown_fs: 525 failed to determine fs type for '/dev/dm-10'
lxc-start: conf.c: setup_rootfs: 1280 failed to mount rootfs
lxc-start: conf.c: do_rootfs_setup: 3713 failed to setup rootfs for 'lxc_client2'
lxc-start: start.c: __lxc_start: 1155 Error setting up rootfs mount as root before spawn
lxc-start: lxc_start.c: main: 344 The container failed to start.

## 8. TO DO

make restore snapshot of lxc thinpool containers work. 

OR: RUN TESTCASE 3: include DRBD in the advanced LXC setup with thinpools and snapshots!

blockdevice explanation
Here we try to explain the increddible complex partitioning/RAID/DRBD/LVM cahe thinpool stuff. Dont do this, it will be explained step by step. but its a good reference for explanation.

First we need (at least) two HDDs and two SSDs fysically available. Lets call the HDDs /dev/sda and /dev/sdb and the SSDs /dev/sdc/ and /dev/sdd
now we want to combine these fysical disk in to RAID 1 (mirror) arrays so we can handle a device failure without major problems. We do this by combineing /dev/sda and /dev/sdb in to /dev/md0 and /dev/sdc and /dev/sdd to /dev/md1. So we are left with two redundant blockdevices: a giant slow /dev/md0 and a smaller fast /dev/md1
Now we need to make /dev/md0 a PV so we can use it as a source for LVM. After this we will create a VG on top of that. Within that VG we will create the crucial LVM's:
bootlv (/boot)
swaplv (SWAP)
rootlv (/)
cachedLV (this LV will be cached by the SSD)
Then we create from the /dev/md1 (fast SSDs) a PV and then ADD IT to the existing VG (that was only on /dev/md0 till now). We now can create specifically on the /dev/md1 two LV's that together form a cachepool. This cacheppol again will be combined with the chachedLV (on the slow /dev/dm0) so we have a truly cached LV.
This cached LV will become a PV of its own with a own VG. Within that VG we create a thinpool. Most of the space within that VG will be uses to create one LV.
the just created (cached) LV now will be the basis of DRBD.
the DRBD blockdevice now will become a PV and we put a VG on that.
On this VG we will create two LV's: a metadata and data partition. Together they form a thinpool that will be used for the creation of LXC LV's (for the containers)
Notes: we cant cache a thinpool so we first had to create a cached LV and then base a thinpool on top of that. We also needed under the DRBD a thinpool because we want to snapshot the DRBD. On ttop of the DRBD we need a thinpool so we can snapshot the LXC lvs. We dont want thinpool for our most important partitions because of possible filling of thinpool and the becomming unusable of the lvs in thath case.
