# Table of Contents

    1. TESTCASE1: test the process of creating and restoring snapshots
    2. TESTCASE2: Install an LXC thinpool on top of an LV and cache the LV on a different volume
    3. TESTCASE3: Setup a system for LXC containers that works with thinpools on top of a cached DRBD device in order to make various kinds of snapshots and have various resize functionalities



# TESTCASE1: test the process of creating and restoring snapshots

Create an LV and confirm creation

    lvcreate -L 1GB -n data  /dev/VG1
    lvs
    
Make a filesystem on it, mount it and confirm mount

    mkfs.ext4 /dev/VG1/data
    mkdir /data
    mount /dev/VG1/data /data
    df -h

Make some files on it and confirm presence of files

    cd /data && touch moo foo shoo
    ls -la
    

Create a 1GB snapshot of LV data and confirm

    lvcreate -L 1G -s -n data_snap /dev/VG1/data
    lvs

Mount the snapshot to confirm snapshot has the same files as LV data

    mkdir /data_snap
    mount /dev/VG1/data_snap /data_snap
    ls -la /data_snap

Make more files on LV data, and confirm /data and /data_snap have a difference in files

    cd /data && touch MOOOO FOOOO SHOOO
    ls -la /data
    ls -la /data_snap

Unmount the data LV and the snapshot and confirm

    umount /data
    umount /data_snap
    df -h
    
Restore data LV with the data_snap snapshot

    lvconvert --merge /dev/VG1/data2_snap

Confirm data LV has been restored to the situation that has only 3 files

    mount /dev/VG1/data /data
    ls -la /data

# TESTCASE2: Install an LXC thinpool on top of an LV and cache the LV on a different volume

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
    
    lvconvert --type cache --cachepool VG1/cachedata VG1/cachedlv
        Logical volume VG1/cachedlv is now cached.
    
    lsblk
    
    NAME                      MAJ:MIN RM  SIZE RO TYPE  MOUNTPOINT
    sda                         8:0    0   20G  0 disk  
    └─sda1                      8:1    0   20G  0 part  
        └─md0                     9:0    0   20G  0 raid1 
        ├─VG1-bootlv          252:0    0  952M  0 lvm   /boot
        ├─VG1-swaplv          252:1    0  952M  0 lvm   [SWAP]
        ├─VG1-rootlv          252:2    0  2.8G  0 lvm   /
        └─VG1-cachedlv_corig  252:6    0  9.3G  0 lvm   
            └─VG1-cachedlv      252:3    0  9.3G  0 lvm   
    sdb                         8:16   0   20G  0 disk  
    └─sdb1                      8:17   0   20G  0 part  
        └─md0                     9:0    0   20G  0 raid1 
        ├─VG1-bootlv          252:0    0  952M  0 lvm   /boot
        ├─VG1-swaplv          252:1    0  952M  0 lvm   [SWAP]
        ├─VG1-rootlv          252:2    0  2.8G  0 lvm   /
        └─VG1-cachedlv_corig  252:6    0  9.3G  0 lvm   
        └─VG1-cachedlv      252:3    0  9.3G  0 lvm   
    sdc                         8:32   0    5G  0 disk  
    └─sdc1                      8:33   0    5G  0 part  
        └─md1                     9:1    0    5G  0 raid1 
        ├─VG1-cachedata_cdata 252:4    0    4G  0 lvm   
        │ └─VG1-cachedlv      252:3    0  9.3G  0 lvm   
        └─VG1-cachedata_cmeta 252:5    0  100M  0 lvm   
        └─VG1-cachedlv      252:3    0  9.3G  0 lvm   
    sdd                         8:48   0    5G  0 disk  
    └─sdd1                      8:49   0    5G  0 part  
        └─md1                     9:1    0    5G  0 raid1 
        ├─VG1-cachedata_cdata 252:4    0    4G  0 lvm   
        │ └─VG1-cachedlv      252:3    0  9.3G  0 lvm   
        └─VG1-cachedata_cmeta 252:5    0  100M  0 lvm   
        └─VG1-cachedlv      252:3    0  9.3G  0 lvm   
    sr0                        11:0    1 1024M  0 rom 

    pvcreate /dev/VG1/cachedlv
        Physical volume "/dev/VG1/cachedlv" successfully created
    vgcreate VG2 /dev/VG1/cachedlv
        Physical volume "/dev/VG1/cachedlv" successfully created
        Volume group "VG2" successfully created
    
    root@livenode5:/home/office# pvs
   
    PV VG Fmt Attr PSize PFree
    /dev/mapper/VG1-cachedlv_corig VG2 lvm2 a-- 9.31g 9.31g
    /dev/md0 VG1 lvm2 a-- 19.98g 5.92g
    /dev/md1 VG1 lvm2 a-- 4.99g 916.00m
    
    lvs
    
    LV VG Attr LSize Pool Origin Data% Meta% Move Log Cpy%Sync Convert
    bootlv VG1 -wi-ao---- 952.00m
    cachedata VG1 Cwi---C--- 4.00g
    cachedlv VG1 Cwi-a-C--- 9.31g cachedata [cachedlv_corig]
    rootlv VG1 -wi-ao---- 2.79g
    swaplv VG1 -wi-ao---- 952.00m

    lvcreate -l 75%FREE --type thin-pool --thinpool lxc VG2
        Logical volume "lvol0" created
        Logical volume "lxc" created
    
    lvs
    
    LV VG Attr LSize Pool Origin Data% Meta% Move Log Cpy%Sync Convert
    bootlv VG1 -wi-ao---- 952.00m
    cachedata VG1 Cwi---C--- 4.00g
    cachedlv VG1 Cwi-a-C--- 9.31g cachedata [cachedlv_corig]
    rootlv VG1 -wi-ao---- 2.79g
    swaplv VG1 -wi-ao---- 952.00m
    lxc lxc twi-a-tz-- 6.96g 0.00 0.73
    
    lxc-create -t download -n lxc_client1 -B lvm --fssize=2G
    
optional if LV status is somehow now available (visible via lvdisplay):
    
    vgchange -a y


# TESTCASE 3: Setup a system with LXC containers that works with thinpools on top of a cached DRBD device in order to make various kinds of snapshots and have various resize functionalities

### 1. Description

Now that we know how to create LXC containers inside a thinpool, know how to cache a logical volume and know how to make snapshots, we can include a DRBD blockdevice in the picture to get the snapshots and resizing functions we want.

### 2. ASCII diagram

### 3. Why this setup

    
- we originally wanted to cache a thinpool so that we can cache the LXC containers but unfortunately it's not possible to cache thinpools so instead of that we cache the LV named CACHEDLV that's under it, so effectively we're 'caching' the thinpools by caching it's parent LV.
- we want to snapshot the DRBD blockdevice so that we can create a snapshot of all LXC containers on the DRBDD in one snapshot but unfortunately it's not possible to snapshot a DRBD so instead of that we can make a snapshot of the parent  thinpool that's under it, effectively 'snapshotting' the blockdevice that houses all the LXC containers. 
- Besides backing up all LXC containers at once we also want to snapshot the individual LXC clients so on top of the DRBD so we will create lxc containers inside a thinpool so we can snapshot the individual LXC lvs. Also we'll benefit from the resizing capabilities of thin volumes. Also we can resize this thinpool by resizing it's parent VG named DRBDVG2
- DRBD has advantages, namely ...

#### 4. Setup

#### 4.1 start

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

#### 4.2 Cache cachedlv on SSD

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

    lsblk
    NAME                      MAJ:MIN RM  SIZE RO TYPE  MOUNTPOINT
    sda                         8:0    0   20G  0 disk  
    └─sda1                      8:1    0   20G  0 part  
      └─md0                     9:0    0   20G  0 raid1 
        ├─VG1-bootlv          252:0    0  952M  0 lvm   /boot
        ├─VG1-swaplv          252:1    0  952M  0 lvm   [SWAP]
        ├─VG1-rootlv          252:2    0  2.8G  0 lvm   /
        └─VG1-cachedlv_corig  252:6    0  9.3G  0 lvm   
          └─VG1-cachedlv      252:3    0  9.3G  0 lvm   
    sdb                         8:16   0   20G  0 disk  
    └─sdb1                      8:17   0   20G  0 part  
      └─md0                     9:0    0   20G  0 raid1 
        ├─VG1-bootlv          252:0    0  952M  0 lvm   /boot
        ├─VG1-swaplv          252:1    0  952M  0 lvm   [SWAP]
        ├─VG1-rootlv          252:2    0  2.8G  0 lvm   /
        └─VG1-cachedlv_corig  252:6    0  9.3G  0 lvm   
          └─VG1-cachedlv      252:3    0  9.3G  0 lvm   
    sdc                         8:32   0    5G  0 disk  
    └─sdc1                      8:33   0    5G  0 part  
      └─md1                     9:1    0    5G  0 raid1 
        ├─VG1-cachedata_cdata 252:4    0    4G  0 lvm   
        │ └─VG1-cachedlv      252:3    0  9.3G  0 lvm   
        └─VG1-cachedata_cmeta 252:5    0  100M  0 lvm   
          └─VG1-cachedlv      252:3    0  9.3G  0 lvm   
    sdd                         8:48   0    5G  0 disk  
    └─sdd1                      8:49   0    5G  0 part  
    └─md1                       9:1    0    5G  0 raid1 
        ├─VG1-cachedata_cdata 252:4    0    4G  0 lvm   
        │ └─VG1-cachedlv      252:3    0  9.3G  0 lvm   
        └─VG1-cachedata_cmeta 252:5    0  100M  0 lvm   
          └─VG1-cachedlv      252:3    0  9.3G  0 lvm   
    sr0                        11:0    1 1024M  0 rom 

