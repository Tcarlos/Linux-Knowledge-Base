# 1. Introduction

Some disadvantages were discovered using LXC: less stable, less documented than KVM. Although LXC is light and fast, we must explore the KVM alternative, for it is widely used, documented and stable.

## 1.1 ASCII Diagram (needs update!)

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
  
  
# 2. Setup
  
## 2.1 Start
 
We start with having 3 freshly installed Ubuntu Server VMs (livenode1, livenode2, livenode3) with the following aspects:

   - Each VM has 2 HDDs of 20GB as a RAID 1 device
   - Each VM has 2 SSD of 5GB as a RAID 1 device
   - On top of the HDD RAID we have 4 LVs, namely:
    - 1 GB bootlv, mounted on /boot
    - 1 GB swaplv, used as swap
    - 3 GB rootlv, mounted on /
    - 10 GB cachedlv, unmounted and to be used as basis for a whole lot of shebang

## 2.2 Cache cachedlv on the SSD raid

Caching has the advantage of quicker disk access. For optimum results we will create the cachepool on a seperate SSD disk, which is ofcourse faster than if it were on the same HDD of the LV.

**Make a physical device of the SSD raid, so we can use it for the LVM cache feature.**

    pvcreate /dev/md1
        Physical volume "/dev/md1" successfully created
        
    pvs
  
    PV         VG   Fmt  Attr PSize  PFree
    /dev/md0   VG1  lvm2 a--  19.98g 6.02g
    /dev/md1   VG1  lvm2 a--   4.99g 4.99g

**Now we will add this Physical Device to the same Volume Group of cached LV. As you can see that Volume Group is named VG1. We do this by extending the VG.**

    vgextend VG1 /dev/md1
        Volume group "VG1" successfully extended
                                              
**Next we will create the cachelayer. This consists of 2 to be created LVs on the fast SSD. One is the CacheDataLV, which is where the caching takes place. The other is the CacheMetaLV which is used to store an index of the data blocks that are cached on the CacheDataLV. The documentation says that the CacheMetaLV should be 1/1000th of the size of the CacheDataLV, but a minimum of 8MB.**

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

**Next we will convert these 2 LV's into a cachepool**

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
  
**And finally, we attach the cachepool to cachedlv, making the caching function operational.**
  
    lvconvert --type cache --cachepool VG1/cachedata VG1/cachedlv
        Logical volume VG1/cachedlv is now cached.


2.3 Create on top of cachedlv a thinpool

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

**Create Thinpool:**

    lvcreate -n cachedDRBDthinpool -l 1254 DRBDVG
    lvcreate -n cachedDRBD_thin_meta -l 125 DRBDVG

    lvconvert --type thin-pool --poolmetadata DRBDVG/cachedDRBD_thin_meta DRBDVG/cachedDRBDthinpool
    WARNING: Converting logical volume DRBDVG/cachedDRBDthinpool and DRBDVG/cachedDRBD_thin_meta to pool's data and metadata volumes.
THIS WILL DESTROY CONTENT OF LOGICAL VOLUME (filesystem etc.)
Do you really want to convert DRBDVG/cachedDRBDthinpool and DRBDVG/cachedDRBD_thin_meta? [y/n]: y
Logical volume "lvol0" created
Converted DRBDVG/cachedDRBDthinpool to thin pool.

**Create Thin Volume:**

    lvcreate -n DRBDLV1 -V 1g --thinpool DRBDVG/cachedDRBDthinpool

**Set filters in /etc/lvm/lvm.conf**

    filter = [ "r|/dev/mapper/VG1-cachedlv_corig|", "r|/dev/mapper/DRBDVG-DRBDLV1|", 
    /etc/init.d/lvm2 restart
    vgck




## 2.4 Add eth1 devices
  
Next, we will make eth1 devices for **livenode1** and **livenode2**. These will be used for the 3node DRBD device.
  
**Add these lines in /etc/network/interfaces:**

**on livenode1:**

    auto eth1
    iface eth1 inet static
      address 10.1.1.31
      netmask 255.255.255.0

**on livenode2:**

    auto eth1
    iface eth1 inet static
      address 10.1.1.32
      netmask 255.255.255.0

**Now shutdown the VMs, rightclick -> settings -> Network -> adapter2 tab. In Virtualbox, apply these settings:**

    Attached to: Internal Network
    name: intn
    promiscious mode: allow all

**Finally, restart the VMs**


  
  
   
    
