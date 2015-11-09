# LVM

LVM stands for Logical Volume Manager and has great features such as:

  - Thin Provisioning
  - Snapshots

## Example Situation used in this article

Diagram of Example 1, a situation of my work in November 2015:

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
    | [20GB] HDD1: /dev/sda ||  [20GB]HDD2:/dev/sdb         ||      [5GB] SSD1: /dev/sdc  || [5GB] SSD2: /dev/sdd        |
    +--------------------------------------------------------------------------------------------------------------------+

Here are some outputs:

**root@livenode5:/home/office# lsblk**
  
    NAME                          MAJ:MIN RM  SIZE RO TYPE  MOUNTPOINT
    sda                             8:0    0   20G  0 disk
    └─sda1                          8:1    0   20G  0 part
      └─md0                         9:0    0   20G  0 raid1
        ├─VG1-bootlv              252:0    0  952M  0 lvm   /boot
        ├─VG1-swaplv              252:1    0  952M  0 lvm   [SWAP]
        ├─VG1-rootlv              252:2    0  2.8G  0 lvm   /
        └─VG1-cachedlv_corig      252:5    0   10G  0 lvm
          └─VG1-cachedlv          252:6    0   10G  0 lvm
            ├─lxc-lxc_tmeta       252:7    0    8M  0 lvm
            │ └─lxc-lxc-tpool     252:9    0  7.5G  0 lvm
            │   ├─lxc-lxc         252:10   0  7.5G  0 lvm
            │   └─lxc-lxc_client2 252:11   0    2G  0 lvm
            └─lxc-lxc_tdata       252:8    0  7.5G  0 lvm
              └─lxc-lxc-tpool     252:9    0  7.5G  0 lvm
                ├─lxc-lxc         252:10   0  7.5G  0 lvm
                └─lxc-lxc_client2 252:11   0    2G  0 lvm
    sdb                             8:16   0   20G  0 disk
    └─sdb1                          8:17   0   20G  0 part
      └─md0                         9:0    0   20G  0 raid1
        ├─VG1-bootlv              252:0    0  952M  0 lvm   /boot
        ├─VG1-swaplv              252:1    0  952M  0 lvm   [SWAP]
        ├─VG1-rootlv              252:2    0  2.8G  0 lvm   /
        └─VG1-cachedlv_corig      252:5    0   10G  0 lvm
          └─VG1-cachedlv          252:6    0   10G  0 lvm
            ├─lxc-lxc_tmeta       252:7    0    8M  0 lvm
            │ └─lxc-lxc-tpool     252:9    0  7.5G  0 lvm
            │   ├─lxc-lxc         252:10   0  7.5G  0 lvm
            │   └─lxc-lxc_client2 252:11   0    2G  0 lvm
            └─lxc-lxc_tdata       252:8    0  7.5G  0 lvm
              └─lxc-lxc-tpool     252:9    0  7.5G  0 lvm
                ├─lxc-lxc         252:10   0  7.5G  0 lvm
                └─lxc-lxc_client2 252:11   0    2G  0 lvm
    sdc                             8:32   0    5G  0 disk
    └─sdc1                          8:33   0    5G  0 part
      └─md1                         9:1    0    5G  0 raid1
        ├─VG1-cachedata_cdata     252:3    0    4G  0 lvm
        │ └─VG1-cachedlv          252:6    0   10G  0 lvm
        │   ├─lxc-lxc_tmeta       252:7    0    8M  0 lvm
        │   │ └─lxc-lxc-tpool     252:9    0  7.5G  0 lvm
        │   │   ├─lxc-lxc         252:10   0  7.5G  0 lvm
        │   │   └─lxc-lxc_client2 252:11   0    2G  0 lvm
        │   └─lxc-lxc_tdata       252:8    0  7.5G  0 lvm
        │     └─lxc-lxc-tpool     252:9    0  7.5G  0 lvm
        │       ├─lxc-lxc         252:10   0  7.5G  0 lvm
        │       └─lxc-lxc_client2 252:11   0    2G  0 lvm
        └─VG1-cachedata_cmeta     252:4    0  100M  0 lvm
          └─VG1-cachedlv          252:6    0   10G  0 lvm
            ├─lxc-lxc_tmeta       252:7    0    8M  0 lvm
            │ └─lxc-lxc-tpool     252:9    0  7.5G  0 lvm
            │   ├─lxc-lxc         252:10   0  7.5G  0 lvm
            │   └─lxc-lxc_client2 252:11   0    2G  0 lvm
            └─lxc-lxc_tdata       252:8    0  7.5G  0 lvm
              └─lxc-lxc-tpool     252:9    0  7.5G  0 lvm
                ├─lxc-lxc         252:10   0  7.5G  0 lvm
                └─lxc-lxc_client2 252:11   0    2G  0 lvm
    sdd                             8:48   0    5G  0 disk
    └─sdd1                          8:49   0    5G  0 part
      └─md1                         9:1    0    5G  0 raid1
        ├─VG1-cachedata_cdata     252:3    0    4G  0 lvm
        │ └─VG1-cachedlv          252:6    0   10G  0 lvm
        │   ├─lxc-lxc_tmeta       252:7    0    8M  0 lvm
        │   │ └─lxc-lxc-tpool     252:9    0  7.5G  0 lvm
        │   │   ├─lxc-lxc         252:10   0  7.5G  0 lvm
        │   │   └─lxc-lxc_client2 252:11   0    2G  0 lvm
        │   └─lxc-lxc_tdata       252:8    0  7.5G  0 lvm
        │     └─lxc-lxc-tpool     252:9    0  7.5G  0 lvm
        │       ├─lxc-lxc         252:10   0  7.5G  0 lvm
        │       └─lxc-lxc_client2 252:11   0    2G  0 lvm
        └─VG1-cachedata_cmeta     252:4    0  100M  0 lvm
          └─VG1-cachedlv          252:6    0   10G  0 lvm
            ├─lxc-lxc_tmeta       252:7    0    8M  0 lvm
            │ └─lxc-lxc-tpool     252:9    0  7.5G  0 lvm
            │   ├─lxc-lxc         252:10   0  7.5G  0 lvm
            │   └─lxc-lxc_client2 252:11   0    2G  0 lvm
            └─lxc-lxc_tdata       252:8    0  7.5G  0 lvm
              └─lxc-lxc-tpool     252:9    0  7.5G  0 lvm
                ├─lxc-lxc         252:10   0  7.5G  0 lvm
                └─lxc-lxc_client2 252:11   0    2G  0 lvm
    sr0                            11:0    1 1024M  0 rom

**root@livenode5:/home/office# pvs**

    Found duplicate PV WtHbpf8uHdCyhlTY8E9s9LMdqZmHWKU0: using /dev/VG1/cachedlv not /dev/mapper/VG1-cachedlv_corig
    PV                VG   Fmt  Attr PSize  PFree
    /dev/VG1/cachedlv lxc  lvm2 a--  10.00g   2.50g
    /dev/md0          VG1  lvm2 a--  19.98g   5.23g
    /dev/md1          VG1  lvm2 a--   4.99g 916.00m

**root@livenode5:/home/office# vgs**

    Found duplicate PV WtHbpf8uHdCyhlTY8E9s9LMdqZmHWKU0: using /dev/VG1/cachedlv not /dev/mapper/VG1-cachedlv_corig
    VG   #PV #LV #SN Attr   VSize  VFree
    VG1    2   5   0 wz--n- 24.97g 6.12g
    lxc    1   2   0 wz--n- 10.00g 2.50g

**root@livenode5:/home/office# lvs**

    Found duplicate PV WtHbpf8uHdCyhlTY8E9s9LMdqZmHWKU0: using /dev/VG1/cachedlv not /dev/mapper/VG1-cachedlv_corig
    LV          VG   Attr       LSize   Pool      Origin           Data%  Meta%  Move Log Cpy%Sync Convert
    bootlv      VG1  -wi-ao---- 952.00m
    cachedata   VG1  Cwi---C---   4.00g
    cachedlv    VG1  Cwi-aoC---  10.00g cachedata [cachedlv_corig]
    rootlv      VG1  -wi-ao----   2.79g
    swaplv      VG1  -wi-ao---- 952.00m
    lxc         lxc  twi-a-tz--   7.48g                            6.63   3.47
    lxc_client2 lxc  Vwi-aotz--   2.00g lxc                        24.80


## Creating the partitioning scheme

### Install Ubuntu Server with LVM

### Install LXC

### Set up physical volumes, volume groups, logical volumes

### Set up a thinpool

  - Regular thinppool
  - LXC thinpool

## Snapshots

Check for available space on the volume group(s).

In example 1A we see that volume group named VG1 has a total size of 24.97GB, of which 6.12GB is free.
When we list the logical volumes we see that there are 5 logical volumes in volume group named VG1.
When we add up the space each logical volume in VG1 has, we indeed get to the 24.79GB

Now we will create a snapshot of 1GB

root@livenode5:/home/office# lvcreate -L 1GB -s -n rootlv_snap /dev/VG1/rootlv
  Found duplicate PV WtHbpf8uHdCyhlTY8E9s9LMdqZmHWKU0: using /dev/VG1/cachedlv not /dev/mapper/VG1-cachedlv_corig
  Logical volume "rootlv_snap" created

**CHALLENGE:** create a snapshot of LXC containers, one way or another.

## Thin Provisoning

## Thin Pools

## Thin Volumes

## Overprovisioning

## LXC and LVM







### Sources:
https://www.flockport.com/lxc-and-lvm/
http://www.tecmint.com/setup-thin-provisioning-volumes-in-lvm/
