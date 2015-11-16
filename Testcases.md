# TESTCASE1: test the process of creating and restoring snapshot

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

#TESTCASE2: Install an LXC thinpool on top of an LV and cache the LV on a different volume

    root@livenode1:/home/office# lvcreate -L 100M -n cachemeta VG1 /dev/md1
    Logical volume "cachemeta" created
    root@livenode1:/home/office# lvcreate -L 4G -n cachedata VG1 /dev/md1
    Logical volume "cachedata" created
    root@livenode1:/home/office# lvs
    LV        VG   Attr       LSize   Pool Origin Data%  Meta%  Move Log Cpy%Sync Convert
    bootlv    VG1  -wi-ao---- 952.00m                                                    
    cachedata VG1  -wi-a-----   4.00g                                                    
    cachedlv  VG1  -wi-a-----   9.31g                                                    
    cachemeta VG1  -wi-a----- 100.00m                                                    
    rootlv    VG1  -wi-ao----   2.79g                                                    
    swaplv    VG1  -wi-ao---- 952.00m                                                    
    root@livenode1:/home/office# lvconvert --type cache-pool --cachemode writethrough --poolmetadata VG1/cachemeta VG1/cachedata
    WARNING: Converting logical volume VG1/cachedata and VG1/cachemeta to pool's data and metadata volumes.
    THIS WILL DESTROY CONTENT OF LOGICAL VOLUME (filesystem etc.)
    Do you really want to convert VG1/cachedata and VG1/cachemeta? [y/n]: y
        Logical volume "lvol0" created
        Converted VG1/cachedata to cache pool.
    root@livenode1:/home/office# lvs
    LV        VG   Attr       LSize   Pool Origin Data%  Meta%  Move Log Cpy%Sync Convert
    bootlv    VG1  -wi-ao---- 952.00m                                                    
    cachedata VG1  Cwi---C---   4.00g                                                    
    cachedlv  VG1  -wi-a-----   9.31g                                                    
    rootlv    VG1  -wi-ao----   2.79g                                                    
    swaplv    VG1  -wi-ao---- 952.00m                                                    
    root@livenode1:/home/office# lvconvert --type cache --cachepool VG1/cachedata VG1/cachedlv
    Logical volume VG1/cachedlv is now cached.
    root@livenode1:/home/office# lsblk
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
    Found duplicate PV 61P64oL5CdMpmu1EbEni9B8TUYMlNjUJ: using /dev/mapper/VG1-cachedlv_corig not /dev/VG1/cachedlv
    Found duplicate PV 61P64oL5CdMpmu1EbEni9B8TUYMlNjUJ: using /dev/VG1/cachedlv not /dev/mapper/VG1-cachedlv_corig
    Found duplicate PV 61P64oL5CdMpmu1EbEni9B8TUYMlNjUJ: using /dev/mapper/VG1-cachedlv_corig not /dev/VG1/cachedlv
    Found duplicate PV 61P64oL5CdMpmu1EbEni9B8TUYMlNjUJ: using /dev/VG1/cachedlv not /dev/mapper/VG1-cachedlv_corig
    Found duplicate PV 61P64oL5CdMpmu1EbEni9B8TUYMlNjUJ: using /dev/mapper/VG1-cachedlv_corig not /dev/VG1/cachedlv
    Physical volume "/dev/VG1/cachedlv" successfully created
    Volume group "VG2" successfully created
    root@livenode5:/home/office# pvs
    Found duplicate PV 5hJoMPZf0RDqcw5s8GJfk7vC9HEuQuS6: using /dev/mapper/VG1-cachedlv_corig not /dev/VG1/cachedlv
    PV VG Fmt Attr PSize PFree
    /dev/mapper/VG1-cachedlv_corig VG2 lvm2 a-- 9.31g 9.31g
    /dev/md0 VG1 lvm2 a-- 19.98g 5.92g
    /dev/md1 VG1 lvm2 a-- 4.99g 916.00m
    root@livenode5:/home/office# lvs
    Found duplicate PV 5hJoMPZf0RDqcw5s8GJfk7vC9HEuQuS6: using /dev/mapper/VG1-cachedlv_corig not /dev/VG1/cachedlv
    LV VG Attr LSize Pool Origin Data% Meta% Move Log Cpy%Sync Convert
    bootlv VG1 -wi-ao---- 952.00m
    cachedata VG1 Cwi---C--- 4.00g
    cachedlv VG1 Cwi-a-C--- 9.31g cachedata [cachedlv_corig]
    rootlv VG1 -wi-ao---- 2.79g
    swaplv VG1 -wi-ao---- 952.00m
    root@livenode5:/home/office# lvcreate -l 75%FREE --type thin-pool --thinpool lxc lxc
    Found duplicate PV 5hJoMPZf0RDqcw5s8GJfk7vC9HEuQuS6: using /dev/mapper/VG1-cachedlv_corig not /dev/VG1/cachedlv
    Volume group "lxc" not found
    root@livenode5:/home/office# lvcreate -l 75%FREE --type thin-pool --thinpool lxc VG2
    Found duplicate PV 5hJoMPZf0RDqcw5s8GJfk7vC9HEuQuS6: using /dev/mapper/VG1-cachedlv_corig not /dev/VG1/cachedlv
    Logical volume "lvol0" created
    Logical volume "lxc" created
    root@livenode5:/home/office# lvs
    Found duplicate PV 5hJoMPZf0RDqcw5s8GJfk7vC9HEuQuS6: using /dev/mapper/VG1-cachedlv_corig not /dev/VG1/cachedlv
    LV VG Attr LSize Pool Origin Data% Meta% Move Log Cpy%Sync Convert
    bootlv VG1 -wi-ao---- 952.00m
    cachedata VG1 Cwi---C--- 4.00g
    cachedlv VG1 Cwi-a-C--- 9.31g cachedata [cachedlv_corig]
    rootlv VG1 -wi-ao---- 2.79g
    swaplv VG1 -wi-ao---- 952.00m
    lxc lxc twi-a-tz-- 6.96g 0.00 0.73
    root@livenode5:/home/office# lxc-create -t download -n lxc_client1 -B lvm --fssize=2G
    vgchange -a y


# TESTCASE 3: include DRBD in the advanced LXC setup with thinpools and snapshots
