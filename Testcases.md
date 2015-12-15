# TESTCASE 3: Setup a system with LXC containers that works with thinpools on top of a cached DRBD device in order to make various kinds of snapshots and have various resize functionalities

# Table of Contents

    1. TESTCASE1: test the process of creating and restoring snapshots
 




# 1. Description

Now that we know how to create LXC containers inside a thinpool, know how to cache a logical volume and know how to make snapshots, we can include a DRBD blockdevice in the picture to get the snapshots and resizing functions we want.

**ASCII Diagram:**

# 2. Why this setup

- We originally wanted to cache a thinpool so that we can cache the LXC containers but unfortunately it's not possible to cache thinpools so instead of that we cache the LV named CACHEDLV that's under it, so effectively we're 'caching' the thinpools by caching it's parent LV.
- We want to snapshot the DRBD blockdevice so that we can create a snapshot of all LXC containers on the DRBDD in one snapshot but unfortunately it's not possible to snapshot a DRBD so instead of that we can make a snapshot of the parent  thinpool that's under it, effectively 'snapshotting' the blockdevice that houses all the LXC containers. 
- Besides backing up all LXC containers at once we also want to snapshot the individual LXC clients so on top of the DRBD so we will create lxc containers inside a thinpool so we can snapshot the individual LXC lvs. Also we'll benefit from the resizing capabilities of thin volumes. Also we can resize this thinpool by resizing it's parent VG named DRBDVG2
- DRBD has advantages, namely ...

# 3. Setup

At the start of this scenario, we have just installed Ubuntu Server 15.04, with 4 Logical Volumes on an HDD RAID, and 1 unused SDD RAID as of yet. See testcase #1 in this article if you need more info on the Ubuntu setup. Anyway, start looks like this:

    lvs
    
    LV       VG   Attr       LSize   Pool Origin Data%  Meta%
    bootlv   VG1  -wi-ao---- 952.00m                                                   
    cachedlv VG1  -wi-a-----  10.00g                                                   
    rootlv   VG1  -wi-ao----   2.79g                                                   
    swaplv   VG1  -wi-ao---- 952.00m    
    
    lsblk
    
    NAME               MAJ:MIN RM  SIZE RO TYPE  MOUNTPOINT
    sda                  8:0    0   20G  0 disk  
    └─sda1               8:1    0   20G  0 part  
      └─md0              9:0    0   20G  0 raid1 
        ├─VG1-bootlv   252:0    0  952M  0 lvm   /boot
        ├─VG1-swaplv   252:1    0  952M  0 lvm   [SWAP]
        ├─VG1-rootlv   252:2    0  2.8G  0 lvm   /
        └─VG1-cachedlv 252:3    0  9.3G  0 lvm   
    sdb                  8:16   0   20G  0 disk  
    └─sdb1               8:17   0   20G  0 part  
      └─md0              9:0    0   20G  0 raid1 
        ├─VG1-bootlv   252:0    0  952M  0 lvm   /boot
        ├─VG1-swaplv   252:1    0  952M  0 lvm   [SWAP]
        ├─VG1-rootlv   252:2    0  2.8G  0 lvm   /
        └─VG1-cachedlv 252:3    0  9.3G  0 lvm   
    sdc                  8:32   0    5G  0 disk  
    └─sdc1               8:33   0    5G  0 part  
      └─md1              9:1    0    5G  0 raid1 
    sdd                  8:48   0    5G  0 disk  
    └─sdd1               8:49   0    5G  0 part  
      └─md1              9:1    0    5G  0 raid1 
    sr0                 11:0    1 1024M  0 rom
    
cachedlv will be our target blockdevice. On top of this we will create our DRBD LXC thinpool structure. As the name probably told you, cachedlv will also be cached, on the SSD raid.

## 3.1 Getting the required programs

  sudo su
    apt-get uddate && apt-get upgrade
    screen
    apt-get install lxc drbd-utils thin-provisioning-tools
    
## 3.2 Configure LVM

Check the /etc/lvm/lvm.conf for:

    thin_pool_chunk_size_policy = "performance"
    thin_check_executable = "/usr/sbin/thin_check"
    thin_check_options = [ "-q", "--clear-needs-check-flag" ]
    thin_repair_executable = "/usr/sbin/thin_repair"
    thin_dump_executable = "/usr/sbin/thin_dump"
    cache_check_executable = "/usr/sbin/cache_check"
    cache_check_options = [ "-q" ]
    cache_repair_executable = "/usr/sbin/cache_repair"
    cache_dump_executable = "/usr/sbin/cache_dump"
    thin_pool_autoextend_threshold = 50
    thin_pool_autoextend_percent = 10
    
## 3.3 Cache cachedlv on SSD

Caching has the advantage of quicker disk access. For optimum results we will create the cachepool on a seperate SSD disk, which is ofcourse faster than if it were on the same HDD of the LV.

Make a physical device of the SSD raid, so we can use it for the LVM cache feature.

    pvcreate /dev/md1
        Physical volume "/dev/md1" successfully created
        
    pvs
  
    PV         VG   Fmt  Attr PSize  PFree
    /dev/md0   VG1  lvm2 a--  19.98g 6.02g
    /dev/md1   VG1  lvm2 a--   4.99g 4.99g

Now we will add this Physical Device to the same Volume Group of cached LV. As you can see that Volume Group is named VG1. We do this by extending the VG.

    vgextend VG1 /dev/md1
        Volume group "VG1" successfully extended
                                              
Next we will create the cachelayer. This consists of 2 to be created LVs on the fast SSD. One is the CacheDataLV, which is where the caching takes place. The other is the CacheMetaLV which is used to store an index of the data blocks that are cached on the CacheDataLV. The documentation says that the CacheMetaLV should be 1/1000th of the size of the CacheDataLV, but a minimum of 8MB.

    lvcreate -L 100M -n cachemeta VG1 /dev/md1
        Logical volume "cachemeta" created
    lvcreate -L 4G -n cachedata VG1 /dev/md1
        Logical volume "cachedata" created

    lvs
    LV        VG   Attr       LSize   Pool Origin Data%  Meta%  Move Log Cpy%Sync Convert
    bootlv    VG1  -wi-ao---- 952.00m                                                    
    cachedata VG1  -wi-a-----   4.00g                                                    
    cachedlv  VG1  -wi-a-----   9.31g                                                    
    cachemeta VG1  -wi-a----- 100.00m                                                    
    rootlv    VG1  -wi-ao----   2.79g                                                    
    swaplv    VG1  -wi-ao---- 952.00m                                                    

Next we will convert these 2 LV's into a cachepool

    lvconvert --type cache-pool --cachemode writethrough --poolmetadata VG1/cachemeta VG1/cachedata
    WARNING: Converting logical volume VG1/cachedata and VG1/cachemeta to pool's data and metadata volumes.
    THIS WILL DESTROY CONTENT OF LOGICAL VOLUME (filesystem etc.)
    Do you really want to convert VG1/cachedata and VG1/cachemeta? [y/n]: y
        Logical volume "lvol0" created
        Converted VG1/cachedata to cache pool.

    lvs

    LV        VG   Attr       LSize   Pool Origin Data%  Meta%  Move Log Cpy%Sync Convert
    bootlv    VG1  -wi-ao---- 952.00m                                                    
    cachedata VG1  Cwi---C---   4.00g                                                    
    cachedlv  VG1  -wi-a-----   9.31g                                                    
    rootlv    VG1  -wi-ao----   2.79g                                                    
    swaplv    VG1  -wi-ao---- 952.00m
  
And finally, we attach the cachepool to cachedlv, making the caching function operational.
  
    lvconvert --type cache --cachepool VG1/cachedata VG1/cachedlv
        Logical volume VG1/cachedlv is now cached.

    lvs

## 3.4 Setup the DRBD device

**Set in /etc/lvm/lvm.conf:**

    filter = [ "r|/dev/mapper/dev/mapper/VG1-cachedlv_corig|" ]

    /etc/init.d/lvm2 restart
    vgck

**Create PV on cachedLV:**

    pvcreate /dev/VG1/cachedlv
        Physical volume "/dev/VG1/cachedlv" successfully created
        
**Create VG on top of that

    vgcreate DRBDVG /dev/VG1/cachedlv
        Found duplicate PV 7u94b0iYNkobo6RZ4534NN9hb9MOhoyl: using /dev/mapper/VG1-cachedLV_corig not /dev/VG1/cachedLV
        Found duplicate PV 7u94b0iYNkobo6RZ4534NN9hb9MOhoyl: using /dev/VG1/cachedLV not /dev/mapper/VG1-cachedLV_corig
        Found duplicate PV 7u94b0iYNkobo6RZ4534NN9hb9MOhoyl: using /dev/mapper/VG1-cachedLV_corig not /dev/VG1/cachedLV
        Found duplicate PV 7u94b0iYNkobo6RZ4534NN9hb9MOhoyl: using /dev/VG1/cachedLV not /dev/mapper/VG1-cachedLV_corig
        Found duplicate PV 7u94b0iYNkobo6RZ4534NN9hb9MOhoyl: using /dev/mapper/VG1-cachedLV_corig not /dev/VG1/cachedLV
        Physical volume "/dev/VG1/cachedlv" successfully created
            Volume group "DRBDVG" successfully created
            

**Create an LV as a backing device for DRBD**

    lvcreate -L 10GB -n DRBDLV1 DRBDVG

**Config DRBD config file**

    touch /etc/drbd.d/r0.res

    resource r0 {
     on livenode5 {
       device /dev/drbd0;
       disk /dev/mapper/DRBDVG-DRBDLV1;
       address 127.0.0.1:7789;
       meta-disk internal;
        }
    }

    resource r0-U {
      net {
        protocol A;
      }

      stacked-on-top-of r0 {
        device /dev/drbd10;
        address 127.0.0.1:7791;
       }

    on bullshit {
        device /dev/drbd10;
        disk /dev/bulldisk;
        address 127.0.0.1:7794;
        meta-disk internal;
        }
    }

Doublecheck that the 2nd line corresponds with uname -n!

**Create DRBD meta datablock**

    drbdadm create-md r0

    initializing activity log
    NOT initializing bitmap
    Writing meta data...
    New drbd meta data block successfully created.

To enable a stacked resource, you first enable its lower-level resource and promote it:

    drbdadm up r0

**IGNORE FOLLOWING ERROR**

    /etc/drbd.d/r0.res:1: in resource r0:
        Missing section 'on <PEER> { ... }'.
    Device '0' is configured!
    Command 'drbdmeta 0 v08 /dev/mapper/DRBDVG-DRBDLV1 internal apply-al' terminated with exit code 20

**Promote (set primary):**

    drbdadm primary --force r0

**Create DRBD meta data on the stacked resources**

    drbdadm create-md --stacked r0-U

**Then, you may enable the stacked resource:**

    drbdadm up --stacked r0-U 
    drbdadm primary --force --stacked r0-U

**Check**

    cat /proc/drbd
    version: 8.4.5 (api:1/proto:86-101)
    srcversion: 82483CBF1A7AFF700CBEEC5
    0: cs:StandAlone ro:Primary/Unknown ds:UpToDate/DUnknown   r----s
    ns:0 nr:0 dw:40 dr:1592 al:0 bm:0 lo:0 pe:0 ua:0 ap:0 ep:1 wo:f oos:1048508

**Set filter in /etc/lvm/lvm.conf**

    filter = [ "r|/dev/mapper/VG1-cachedlv_corig|", "r|/dev/mapper/DRBDVG-DRBDLV1|", "r|/dev/drbd0|"]

    /etc/init.d/lvm2 restart

## 3.5 Create VPS thinpool on top of DRBD device**

    pvcreate /dev/drbd10
    vgcreate DRBDVG2 /dev/drbd10
    pvscan

**Check how much space we have**

    vgdisplay

    VG Name               DRBDVG2
    System ID             
    Format                lvm2
    Metadata Areas        1
    Metadata Sequence No  1
    VG Access             read/write
    VG Status             resizable
    MAX LV                0
    Cur LV                0
    Open LV               0
    Max PV                0
    Cur PV                1
    Act PV                1
    VG Size               1020.00 MiB
    PE Size               4.00 MiB
    Total PE              255
    Alloc PE / Size       0 / 0   
    Free  PE / Size       255 / 1020.00 MiB
    VG UUID               1jh8ux-srXk-Vp0e-ENvW-0C11-Ow0E-zC5ecH

Substract one PE for the pools creation. What have left will be devided. And then create a thinpool with that size. Use 50% of the space for it:

    lvcreate -n VPSthinpool -L 125 DRBDVG2
        Rounding up size to full physical extent 128.00 MiB
        Logical volume "VPSthinpool" created
    lvcreate -n VPSthinpool_meta -L 13 DRBDVG2
        Rounding up size to full physical extent 16.00 MiB
        Logical volume "VPSthinpool_meta" created
    lvconvert --type thin-pool --poolmetadata DRBDVG2/VPSthinpool_meta DRBDVG2/VPSthinpool
    WARNING: Converting logical volume DRBDVG2/VPSthinpool and DRBDVG2/VPSthinpool_meta to pool's data and metadata volumes.
    THIS WILL DESTROY CONTENT OF LOGICAL VOLUME (filesystem etc.)
    Do you really want to convert DRBDVG2/VPSthinpool and DRBDVG2/VPSthinpool_meta? [y/n]: y
        Logical volume "lvol0" created
        Converted DRBDVG2/VPSthinpool to thin pool
        



## 3.6 Create container(s) in VPSthinpool on DRBDVG2

    lxc-create -t download -n my-container -B lvm --vgname DRBDVG2 --thinpool VPSthinPool --fssize=500M
    
    
## 3.7 Create a startup script
    
Right after reboot, the DRBVD blockdevice is not yet up and running. With the following three commands we get it back:

    drbdadm primary --force r0

    drbdadm up --stacked r0-U 

    drbdadm primary --force --stacked r0-U
    
Followed by starting the LXC client(s):

    lxc-start -n my-container
    
    
    
# 4. Testing 

## 4.1 Testing LVM Snapshot functions

This setup provides 2 snapshot features

 - Snapshotting the LV right under the DRBD device
 
 - Snapshotting the individual LXC clients

To see howmuch space is available use these commands for more info:

    pvs
    vgs
    vgdisplay
    lvs
    lvdisplay 
    lvdisplay VG1
    
Overprovisioning


### 4.1.1 Snapshotting the LV right under the DRBD device**

    lvcreate -L 1G -s -n DRBDLV1_snap1 /dev/DRBDVG/DRBDLV1

### 4.1.2 Snapshotting the individual LXC clients

    lvcreate -L 500M -s -n my-container_snap /dev/DRBDVG2/my-container
    
### 4.1.3 Restoring Logical Volumes with Snapshots

### 4.1.4 Removing snapshots**

    lvremove /dev/DRBDVG/DRBDLV1_snap1 /dev/DRBDVG/DRBDLV1
    
### 4.1.5 Auto extending snapshots
    
### 4.1.6 Automatic snapshots with DRBD

## 4.2 Testing resizing of various volumes

**Resizing the VPS thinpool with lvresize and lvextend commands**

    lvresize DRBDVG2/VPSthinpool -L +50M 
    Rounding size to boundary between physical extents: 52.00 MiB Size of logical volume DRBDVG2/VPSthinpool_tdata changed from 128.00 MiB (32 extents) to 180.00 MiB (45 extents). 
    Logical volume VPSthinpool successfully resized

    drbdadm -- --assume-peer-has-space resize r0 
    
    drbdadm -S -- --assume-peer-has-space resize r0-U

    pvresize /dev/drbd10

    lvextend -L +50M DRBDVG2/VPSthinpool Rounding size to boundary between physical extents: 52.00 MiB Size of logical volume DRBDVG2/VPSthinpool_tdata changed from 180.00 MiB (45 extents) to 232.00 MiB (58 extents). 
    Logical volume VPSthinpool successfully resized 
    
        drbdadm -- --assume-peer-has-space resize r0 
    
        drbdadm -S -- --assume-peer-has-space resize r0-U 
        
        pvresize /dev/drbd10 

**resizing lxc clients**
    
        

    
