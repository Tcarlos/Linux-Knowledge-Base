# TESTCASE 3: Create an advanced LXC system with 3-node DRBD

# Table of Contents

    1. Description
        1.1 ASCII Diagram
        1.2 Description in detail
    2. Why this setup
    3. Setup
        3.1 Getting the required programs
        3.2 Configure LVM
        3.3 Configure LXC
        3.4 Cache cachedlv on SSD
        3.5 Create on top of cachedlv a thinpool 
        3.6 Setup and configure the DRBD device
          3.6.1 Create internal IPs for the first and second node
          3.6.2 Create IP aliases on the first and second node
          3.6.3 Create DRBDD configuration file on all three nodes
        3.7 Create VPS thinpool on top of DRBD device
        3.8 Create container(s) in VPSthinpool on DRBDVG2
        3.9 Create a startup script
    4. Testing
      4.1 Testing LVM Snapshot functions
        4.1.1 Snapshotting the LV right under the DRBD device
        4.1.2 Snapshotting the individual LXC clients
        4.1.3 Restoring Logical Volumes with Snapshots
        4.1.4 Removing snapshots
        4.1.5 Auto extending snapshots
        4.1.6 Automatic snapshots with DRBD
      4.2 Testing resizing functions
        4.2.1 Resizing LXC clients
            4.2.1.1 Reducing LXC clients
            4.2.1.2 Increasing LXC Clients
        4.2.3 Overprovisioning Thinpools
      4.3 Other Functions
        4.3.1 Configurating LXC clients
        4.3.2 Cloning LXC clients
    5. Usecases
    6. Troubleshooting
    
# 1. Description

This system with LXC clients has the LXC clients inside a thinpool, on top of a DRBD device, with the whole system cached, has many resize options and has 2 ways of LVM snapshots available. 

## 1.1 ASCII Diagram

                                                              +----------------------------------------------------------+
                                                              | [2GB] LXC_CLIENT1LV ||  [2GB] LXC_CLIENT2LV              |
                                                              +----------------------------------------------------------+
                                                              |            [7.5GB] LOGICAL VOLUME THINPOOL "LXC"         |
                                                              +----------------------------------------------------------+
                                                              +----------------------------------------------------------+
                                                              | [0.008GB] thinmeta ||[7.5GB] thindata                    |
                                                              +----------------------------------------------------------+
                                                              +----------------------------------------------------------+
                                                              | [10GB] VOLUME_GROUP2 "LXC"                               |
                                                              +----------------------------------------------------------+
                                                              +----------------------------------------------------------+
                                                              | [10GB] PHYSICAL_VOLUME3                                  |
                                                              +------------------------//--------------------------------+
                                                              +----------------------//----------------------------------+
                                                              |     [10GB] CACHEDLV//                                    |
                                                              +----------------------------------------------------------+
                          after conversion to cachepool       |   CACHEDLV_corig || CACHEMETA_cmeta   || CACHEDATA_cdata |
    +--------------------------------------------------------------------------------------------------------------------+
    | [0.952GB] BOOTLV || [0.952GB] SWAPLV || [2.79GB] ROOTLV || [10GB] CACHEDLV || [0.1GB] CACHEMETA || [4GB] CACHEDATA |
    +---------------------------------------------------------------------------//---------------------------------------+
    +-------------------------------------------------------------------------//-----------------------------------------+
    +-----------------------------------------------------------------------//-------------------------------------------+
    +---------------------------------------------------------------------// --------------------------------------------+
    +-------------------------------------------------------------------//-----------------------------------------------+
    +-----------------------------------------------------------------//-------------------------------------------------+
    +---------------------------------------------------------------//---------------------------------------------------+
    |                                             [24.97GB] VOLUME_GROUP1                                                |
    +--------------------------------------------------------------------------------------------------------------------+
    +--------------------------------------------------------------------------------------------------------------------+
    |          [19.98GB] PHYSICAL_VOLUME1                    ||             [4.99GB] PHYSICAL_VOLUME2                     |
    +--------------------------------------------------------------------------------------------------------------------+
    +--------------------------------------------------------------------------------------------------------------------+
    |            [20GB] RAID1 /dev/md0                      ||               [5GB]RAID1 /dev/md1                         |
    +--------------------------------------------------------------------------------------------------------------------+
    ---------------------------------------------------------------------------------------------------------------------+
    1. [20GB] HDD1: /dev/sda ||  [20GB]HDD2:/dev/sdb         ||      [5GB] SSD1: /dev/sdc  || [5GB] SSD2: /dev/sdd        |
    +--------------------------------------------------------------------------------------------------------------------+
    
## 1.2 Description in detail

1. The first layer of this setup consists of 2 HDDs of 20GB each, and 2 SSDs of 5GB each
2. The second layer consists of two RAID 1 devices, 2 x 20GB HDD combined and 2x 5GB combined.

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
    
## 3.3 Configure LXC

 sudo su

**manually allocate an UID and GID range to root in /etc/subuid and /etc/subgid by adding this line in both files:**

    root:100000:65536

**add these same values in global configuration file /etc/lxc/default.conf, by adding these lines to it:**

    lxc.id_map = u 0 100000 65536
    lxc.id_map = g 0 100000 65536
    
**set permissions andownerships:**
    
    chown root:100000 /var/lib/lxc
    chmod 750 /var/lib/lxc
    
**creating and testing container:**

    lxc-create -t download -n office1
    lxc-start -n office1
    lxc-stop -n office1

**get inside the container and test networking:**

    lxc-attach -n office1
    ping 8.8.8.8
    exit
    
**you can see the container status by running following commands:**
    
    lxc-info -n office1
    lxc-ls --fancy
    
**Enable changing memory and SWAP by modifying a line in /etc/default/grub to:**

    GRUB_CMDLINE_LINUX_DEFAULT="swapaccount=1"
    
**Modify these lines to optionally limit memory or memory+SWAP in /var/lib/lxc/office1/config:**

    lxc.cgroup.memory.limit_in_bytes = 512M
    lxc.cgroup.memory.memsw.limit_in_bytes = 1G

**update grub and reboot:**
    
    update-grub
    reboot

## 3.4 Cache cachedlv on SSD

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
        
## 3.5 Create on top of cachedlv a thinpool 

In this thinpool we use a thin volume as backing device and basis for DRBD

**Set in /etc/lvm/lvm.conf:**

    filter = [ "r|/dev/mapper/dev/mapper/VG1-cachedlv_corig|" ]

    /etc/init.d/lvm2 restart
    vgck

**Create PV on cachedLV:**

    pvcreate /dev/VG1/cachedlv
        Physical volume "/dev/VG1/cachedlv" successfully created
        
**Create VG on top of that:**

    vgcreate DRBDVG /dev/VG1/cachedlv
        Found duplicate PV 7u94b0iYNkobo6RZ4534NN9hb9MOhoyl: using /dev/mapper/VG1-cachedLV_corig not /dev/VG1/cachedLV
        Found duplicate PV 7u94b0iYNkobo6RZ4534NN9hb9MOhoyl: using /dev/VG1/cachedLV not /dev/mapper/VG1-cachedLV_corig
        Found duplicate PV 7u94b0iYNkobo6RZ4534NN9hb9MOhoyl: using /dev/mapper/VG1-cachedLV_corig not /dev/VG1/cachedLV
        Found duplicate PV 7u94b0iYNkobo6RZ4534NN9hb9MOhoyl: using /dev/VG1/cachedLV not /dev/mapper/VG1-cachedLV_corig
        Found duplicate PV 7u94b0iYNkobo6RZ4534NN9hb9MOhoyl: using /dev/mapper/VG1-cachedLV_corig not /dev/VG1/cachedLV
        Physical volume "/dev/VG1/cachedlv" successfully created
            Volume group "DRBDVG" successfully created
            
**Create Thinpool**

        lvcreate -n cachedDRBDthinpool -l 1254 DRBDVG
        lvcreate -n cachedDRBD_thin_meta -l 125 DRBDVG

        lvconvert --type thin-pool --poolmetadata DRBDVG/cachedDRBD_thin_meta DRBDVG/cachedDRBDthinpool
        WARNING: Converting logical volume DRBDVG/cachedDRBDthinpool and DRBDVG/cachedDRBD_thin_meta to pool's data and metadata volumes.
    THIS WILL DESTROY CONTENT OF LOGICAL VOLUME (filesystem etc.)
    Do you really want to convert DRBDVG/cachedDRBDthinpool and DRBDVG/cachedDRBD_thin_meta? [y/n]: y
    Logical volume "lvol0" created
    Converted DRBDVG/cachedDRBDthinpool to thin pool.
    
**Create Thin Volume**

        lvcreate -n DRBDLV1 -V 1g --thinpool DRBDVG/cachedDRBDthinpool
        
**Set filters in /etc/lvm/lvm.conf**

    filter = [ "r|/dev/mapper/VG1-cachedlv_corig|", "r|/dev/mapper/DRBDVG-DRBDLV1|", 
    /etc/init.d/lvm2 restart
    vgck

## 3.6 Setup and configure the DRBD device

Now we will create a 3 node DRBD device.
This is quite an advanced setup. Please follow the exact sequences of commands and watch out you are on the right node.

### 3.6.1 Create internal IPs for the first and second node

Add these lines in /etc/network/interfaces

NODE 1, livenode5

    auto eth1
    iface eth1 inet static
      address 10.1.1.31
      netmask 255.255.255.0

NODE 2, livenode6

    auto eth1
    iface eth1 inet static
      address 10.1.1.32
      netmask 255.255.255.0

Now shutdown the VMs, rightclick -> settings -> Network -> adapter2 tab. Apply these settings:

    Attached to: Internal Network
    name: intn
    promiscious mode: allow all

Restart the VMs

### 3.6.2 Create IP aliases on the first and second node

Find your gateway adress:

    route |grep UG

Next, add these lines in /etc/network/interfaces

NODE 1, livenode5

    auto eth0:1
    allow-hotplug eth0:1
    iface eth0:1 inet static
    address 192.168.0.120  
    netmask 255.255.255.0
    gateway 192.168.0.1

NODE 2, livenode6

    auto eth0:1
    allow-hotplug eth0:1
    iface eth0:1 inet static
    address 192.168.0.120 #yes this IP is the same on both nodes
    netmask 255.255.255.0
    gateway 192.168.0.1
    
Restart the VMs, or do

    service networking restart

### 3.6.3 Create DRBDD configuration file on all three nodes

**Make file**
    
    touch /etc/drbd.d/r0.res

**Now add these lines in the just created file:**

    resource r0 {
        net {
        protocol C;
                }

        on livenode5 {
            device    /dev/drbd1;
            disk      /dev/mapper/DRBDVG-DRBDLV1;
            address   10.1.1.31:7789;
            meta-disk internal;
        }

        on livenode6 {
            device    /dev/drbd1;
            disk      /dev/mapper/DRBDVG-DRBDLV1;
            address   10.1.1.32:7789;
            meta-disk internal;
        }
    }

    resource r0-U {
        net {
        protocol A;
    }

        stacked-on-top-of r0 {
            device     /dev/drbd10;
            address    192.168.0.120:7788;
        }

        on livenode7 {
        device     /dev/drbd10;
        disk       /dev/mapper/DRBDVG-DRBDLV1;
        address    192.168.0.113:7788; # Public IP of the backup node
        meta-disk  internal;
        }
    }

**Note: make sure the node names corresponds with the hostname, which u can find with uname -n**

### 3.6.4 Preparing The DRBD Devices

**Now that the configuration is in place, create the metadata on alpha and bravo**

    livenode5# drbdadm create-md r0

    Writing meta data...
    initializing activity log
    NOT initialized bitmap
    New drbd meta data block successfully created.

livenode6# drbdadm create-md data-lower

    Writing meta data...
    initialising activity log
    NOT initialized bitmap
    New drbd meta data block successfully created.

**Now start DRBD on alpha and bravo**

    livenode5# service drbd start

    livenode6# service drbd start

**Verify that the lower level DRBD devices are connected**

    cat /proc/drbd

    version: 8.3.0 (api:88/proto:86-89)
    GIT-hash: 9ba8b93e24d842f0dd3fb1f9b90e8348ddb95829 build by root@alpha, 2009-02-05 10:36:11
    0: cs:Connected ro:Secondary/Secondary ds:Inconsistent/Inconsistent C r---
    ns:0 nr:0 dw:0 dr:0 al:0 bm:0 lo:0 pe:0 ua:0 ap:0 ep:1 wo:b oos:19530844

**Tell livenode5 to become the primary node**

NOTE: As the command states, this is going to overwrite any data on bravo: Now is a good time to go and grab your favorite drink.

    livenode5# drbdadm -- --overwrite-data-of-peer primary data-lower
    livenode5# cat /proc/drbd

    version: 8.3.0 (api:88/proto:86-89)
    GIT-hash: 9ba8b93e24d842f0dd3fb1f9b90e8348ddb95829 build by root@alpha, 2009-02-05 10:36:11
    0: cs:SyncSource ro:Primary/Secondary ds:UpToDate/Inconsistent C r---
    ns:3088464 nr:0 dw:0 dr:3089408 al:0 bm:188 lo:23 pe:6 ua:53 ap:0 ep:1 wo:b oos:16442556
    [==>.................] sync'ed: 15.9% (16057/19073)M
    finish: 0:16:30 speed: 16,512 (8,276) K/sec

**After the sync has finished, create the meta-data on r0-U on livenode5, followed by livenode7

NOTE: the resource is r0-U and the --stacked option is on livenode5 only.

    livenode5# drbdadm --stacked create-md r0-U

    Writing meta data...
    initialising activity log
    NOT initialized bitmap
    New drbd meta data block successfully created.
    success

    livenode7# drbdadm create-md r0-U

    Writing meta data...
    initialising activity log
    NOT initialized bitmap
    New drbd meta data block sucessfully created.

**Bring up the stacked resource, then make alpha the primary of data-upper**

    livenode5# drbdadm --stacked adjust r0-U

    livenode7# drbdadm adjust data-upper
    livenode7# cat /proc/drbd

    version: 8.3.0 (api:88/proto:86-89)
    GIT-hash: 9ba8b93e24d842f0dd3fb1f9b90e8348ddb95829 build by root@foxtrot, 2009-02-02 10:28:37
    1: cs:Connected ro:Secondary/Secondary ds:Inconsistent/Inconsistent A r---
    ns:0 nr:0 dw:0 dr:0 al:0 bm:0 lo:0 pe:0 ua:0 ap:0 ep:1 wo:b oos:19530208

    livenode5# drbdadm --stacked -- --overwrite-data-of-peer primary r0-U
    
**Observe working sync on al nodes**

    livenode5# cat /proc/drbd
    livenode6# cat /proc/drbd
    livenode7# cat /proc/drbd

    version: 8.3.0 (api:88/proto:86-89)
    GIT-hash: 9ba8b93e24d842f0dd3fb1f9b90e8348ddb95829 build by root@alpha, 2009-02-05 10:36:11
    0: cs:Connected ro:Primary/Secondary ds:UpToDate/UpToDate C r---
    ns:19532532 nr:0 dw:1688 dr:34046020 al:1 bm:1196 lo:156 pe:0 ua:0 ap:156 ep:1 wo:b oos:0
    1: cs:SyncSource ro:Primary/Secondary ds:UpToDate/Inconsistent A r---
    ns:14512132 nr:0 dw:0 dr:14512676 al:0 bm:885 lo:156 pe:32 ua:292 ap:0 ep:1 wo:b oos:5018200
    [=============>......] sync'ed: 74.4% (4900/19072)M
    finish: 0:07:06 speed: 11,776 (10,992) K/sec

That's it! Now we can use device **/dev/drbd10** as the basis for a VPS thinpool that will host LXC clients.

## 3.7 Create VPS thinpool on top of DRBD device

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

## 3.8 Create container(s) in VPSthinpool on DRBDVG2

    lxc-create -t download -n my-container -B lvm --vgname DRBDVG2 --thinpool VPSthinPool --fssize=500M
    
## 3.9 Create a startup script
    
Right after reboot, the DRBVD blockdevice is not yet up and running. With the following three commands we get it back:

    drbdadm primary --force r0

    drbdadm up --stacked r0-U 

    drbdadm primary --force --stacked r0-U
    
Followed by starting the LXC client(s):

    lxc-start -n my-container
    
    
    
# 4. Testing 

## 4.1 Testing LVM Snapshot functions

This setup provides 2 snapshot features

 - Snapshotting the LV / Thinvolume right under the DRBD device
 
 - Snapshotting the individual LXC clients

To see howmuch space is available use these commands for more info:

    pvs
    vgs
    vgdisplay
    lvs
    lvdisplay 
    lvdisplay VG1

### 4.1.1 Snapshotting the LV right under the DRBD device

    lvcreate -L 1G -s -n DRBDLV1_snap1 /dev/DRBDVG/DRBDLV1

### 4.1.2 Snapshotting the individual LXC clients

    lvcreate -L 500M -s -n my-container_snap /dev/DRBDVG2/my-container
    
### 4.1.3 Restoring Logical Volumes with Snapshots

Just enter the the name of the snapshot with its path, and the linked LV that is to be restored will get restored.

    lvconvert --merge /dev/DRBDVG2/lxc1_snap
    lvconvert --merge /dev/DRBDVG/DRBDLV1_snap1

### 4.1.4 Removing snapshots

First, stop any running LXC clients and/or unmount possible mounted volumes

    lxc-ls --fancy
    lxc-stop -n lxc1
    umount /dev/mapper/DRBDVG2-lxc1
    
Then remove the snapshots with lvremove:

    lvremove /dev/DRBDVG/DRBDLV1_snap1 /dev/DRBDVG/DRBDLV1
    
### 4.1.5 Auto extending snapshots

In /etc/lvm/lvm.conf

    thin_pool_autoextend_threshold = 50
    thin_pool_autoextend_percent = 10
    
### 4.1.6 Automatic snapshots with DRBD

Open the global common file and enter the parameters below:

    nano /etc/drbd.d/global_common.conf

    resource r0 {
      handlers {
        before-resync-target "/usr/lib/drbd/snapshot-resync-target-lvm.sh";
        after-resync-target "/usr/lib/drbd/unsnapshot-resync-target-lvm.sh";
        }
    }
    
**Set a fixed resync rate and calculate howlong syncing takes**

Also in the global common file, make the following entries in the disk section:

    disk {

                c-plan-ahead 0;
                resync-rate 33M;
                # size on-io-error fencing disk-barrier disk-flushes
                # disk-drain md-flushes resync-rate resync-after al-extents
                # c-plan-ahead c-delay-target c-fill-target c-max-rate
                # c-min-rate disk-timeout
            }



The snapshot-resync-target-lvm.sh script creates an LVM snapshot for any volume the resource contains **immediately before** synchronization kicks off. In case the script fails, the synchronization does not commence

^^ So this script will do his work every time the DRBD syncrhonizes, **which is set by resync-rate 33M**

Once synchronization completes, the unsnapshot-resync-target-lvm.sh script removes the snapshot, which is then no longer needed. In case unsnapshotting fails, the snapshot continues to linger around.

So a snapshot will remain if unsnapshot-resync-target-lvm.sh fails to remove it. 
So for this test we will hack above script in such a way it fails to remove the snapshot.

#### Measuring how long syncing takes

In fixed-rate synchronization, the amount of data shipped to the synchronizing peer per second (the synchronization rate) has a configurable, static upper limit. Based on this limit, you may estimate the expected sync time based on the following simple formula:

    Tsync = D/R

tsync is the expected sync time. D is the amount of data to be synchronized, which you are unlikely to have any influence over (this is the amount of data that was modified by your application while the replication link was broken). R is the rate of synchronization, which is configurable — bounded by the throughput limitations of the replication network and I/O subsystem.

**Check howmuch data is on the block device:**

    pvs
    
    PV                VG      Fmt  Attr PSize    PFree  
    /dev/VG1/cachedlv DRBDVG  lvm2 a--     9.31g   3.43g
    /dev/drbd10       DRBDVG2 lvm2 a--  1020.00m 360.00m
    /dev/md0          VG1     lvm2 a--    19.98g   5.92g
    /dev/md1          VG1     lvm2 a--     4.99g 916.00m
    
the PV on drbd10 which hosts DRBDVG2, which hosts VPSthinpool and the LXC thin volume clients, has 1020MB size and 360M free, so the ammount of data is 660M

**Check howmuch is the rate of synchronization as was set in the global common file**

    cat /etc/drbd.d/global_common.conf |grep resync
    
    ...
    resync-rate 33M;
    ...
    
This shows the resync rate is 33M/sec

So, the expected sync time is 660M/33M = 20 secs.

## 4.2 Testing resizing functions

One of the best features of this system is that you can resize various volumes to our needs.

### 4.2.1 Resizing the VPS thinpool (followed by updating DRBD size)

In this situation resizing of the VPS thinpool is easily done with lvresize or lvextend command. Right after this we update the PV of the DRBD too to optimize volume usage. 

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

### 4.2.2 Resizing LXC clients

#### 4.2.2.1 Reducing LXC clients

1) Stop the container:

    lxc-stop -n debian7

2) Just to make sure it's not mounted:

    umount /dev/lxc/debian7

3) Check partition

    e2fsck -f /dev/lxc/debian7

4) Reduce filesystem to a little less size that you need your partition to be.

    resize2fs /dev/lxc/debian7 140G

5) Reduce LVM partition to the size you really want (150 Gb)

    lvreduce -L 150G /dev/lxc/debian7

6) Resize the filesystem.

    resize2fs /dev/lxc/debian7

7) Check partition

    e2fsck -f /dev/lxc/debian7

8) Start container

    lxc-start -d -n debian7

#### 4.2.2.2 Increasing LXC Clients

1) Stop the container:

    lxc-stop -n debian7

2) Just to make sure it's not mounted:

    umount /dev/lxc/debian7

3) Check partition

    e2fsck -f /dev/lxc/debian7

4) Extend LVM partition 

    lvextend -L 300G /dev/lxc/debian7

5) Extend filesystem.

    resize2fs /dev/lxc/debian7 300G

6) Check partition

    e2fsck -f /dev/lxc/debian7

8) Start container

    lxc-start -d -n debian7

### 4.2.3 Overprovisioning Thinpools

    http://www.tecmint.com/setup-thin-provisioning-volumes-in-lvm/

## 4.3 Other Functions

### 4.3.1 Configurating LXC clients

To access the filesystem of an lxc client:

cat /var/lib/lxc/lxc1/config reveals the path of the rootfs of the lxc client is mapped at /dev/DRBDVG2/lxc1
to enter this filesystem:

    mkdir /mount
    mount /dev/DRBDVG2/lxc1 /mount
    ls -la /mount

### 4.3.2 Cloning LXC clients

### 4.3.3 Testing DRBD Synchronisation works correctly 

Monitoring

cat /proc/drbd

    http://drbd.linbit.com/users-guide/ch-admin.html#s-check-status



# 5. Usecases






# 6. Troubleshooting
    
        

    
