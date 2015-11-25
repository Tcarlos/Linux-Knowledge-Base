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

    lvconvert --type cache --cachepool VG1/cachedata VG1/cachedLV

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

Why DRBD: We also needed under the DRBD a thinpool because we want to snapshot the DRBD. On ttop of the DRBD we need a thinpool so we can snapshot the LXC lvs. DRBD is like a networked RAID

### 6.1 Setup 
    
    lvcreate -n cachedLV -L 10G /dev/VG1
    
    lvs
    
    LV       VG   Attr       LSize   Pool Origin Data%  Meta%
    bootlv   VG1  -wi-ao---- 952.00m                                                   
    cachedLV VG1  -wi-a-----  10.00g                                                   
    rootlv   VG1  -wi-ao----   2.79g                                                   
    swaplv   VG1  -wi-ao---- 952.00m  
  
Add another device to the Volume Group:
    
    pvcreate /dev/md1
    vgextend VG1 /dev/md1
    
Create a cachepool on it:

    lvcreate -L 100M -n cachemeta VG1 /dev/md1
        Logical volume "cachemeta" created
    lvcreate -L 4G -n cachedata VG1 /dev/md1
        Logical volume "cachedata" created
        lvconvert --type cache-pool --cachemode writethrough --poolmetadata VG1/cachemeta VG1/cachedata
    WARNING: Converting logical volume VG1/cachedata and VG1/cachemeta to pool's data and metadata volumes.
    THIS WILL DESTROY CONTENT OF LOGICAL VOLUME (filesystem etc.)
    Do you really want to convert VG1/cachedata and VG1/cachemeta? [y/n]: y
        Logical volume "lvol0" created
        Converted VG1/cachedata to cache pool.
    lvconvert --type cache --cachepool VG1/cachedata VG1/cachedLV
        Logical volume VG1/cachedLV is now cached.

Set in /etc/lvm/lvm.conf:

    filter = [ "r|/dev/mapper/dev/mapper/VG1-cachedLV_corig|" ]
    
    /etc/init.d/lvm2 restart
    vgck
    
Create PV on cachedLV:

    pvcreate /dev/VG1/cachedLV
        Physical volume "/dev/VG1/cachedLV" successfully created
                                            
    vgcreate DRBDVG /dev/VG1/cachedLV
        Found duplicate PV 7u94b0iYNkobo6RZ4534NN9hb9MOhoyl: using /dev/mapper/VG1-cachedLV_corig not /dev/VG1/cachedLV
        Found duplicate PV 7u94b0iYNkobo6RZ4534NN9hb9MOhoyl: using /dev/VG1/cachedLV not /dev/mapper/VG1-cachedLV_corig
        Found duplicate PV 7u94b0iYNkobo6RZ4534NN9hb9MOhoyl: using /dev/mapper/VG1-cachedLV_corig not /dev/VG1/cachedLV
        Found duplicate PV 7u94b0iYNkobo6RZ4534NN9hb9MOhoyl: using /dev/VG1/cachedLV not /dev/mapper/VG1-cachedLV_corig
        Found duplicate PV 7u94b0iYNkobo6RZ4534NN9hb9MOhoyl: using /dev/mapper/VG1-cachedLV_corig not /dev/VG1/cachedLV
        Physical volume "/dev/VG1/cachedLV" successfully created
        Volume group "DRBDVG" successfully created
        
Create a thinpool on top of that:

    lvcreate -n cachedDRBDthinpool -l 1254 DRBDVG
        Found duplicate PV ykuH1ZISKj3uonD2Pw9R5yFwaH76yYiN: using /dev/mapper/VG1-cachedLV_corig not /dev/VG1/cachedLV
        Logical volume "cachedDRBDthinpool" created
  
    lvcreate -n cachedDRBD_thin_meta -l 125 DRBDVG
        Found duplicate PV ykuH1ZISKj3uonD2Pw9R5yFwaH76yYiN: using /dev/mapper/VG1-cachedLV_corig not /dev/VG1/cachedLV
        Logical volume "cachedDRBD_thin_meta" created
    lvconvert --type thin-pool --poolmetadata DRBDVG/cachedDRBD_thin_meta DRBDVG/cachedDRBDthinpool
        Found duplicate PV ykuH1ZISKj3uonD2Pw9R5yFwaH76yYiN: using /dev/mapper/VG1-cachedLV_corig not /dev/VG1/cachedLV
        WARNING: Converting logical volume DRBDVG/cachedDRBDthinpool and DRBDVG/cachedDRBD_thin_meta to pool's data and metadata volumes.
        THIS WILL DESTROY CONTENT OF LOGICAL VOLUME (filesystem etc.)
        Do you really want to convert DRBDVG/cachedDRBDthinpool and DRBDVG/cachedDRBD_thin_meta? [y/n]: y
        Logical volume "lvol0" created
        Converted DRBDVG/cachedDRBDthinpool to thin pool.
        
lvcreate -n DRBDLV1 -V 1g --thinpool DRBDVG/cachedDRBDthinpool
        
Config DRBD config file:
  
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
    
Create DRBD meta datablock

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
    
Promote (set primary):

    drbdadm primary --force r0
    
create DRBD meta data on the stacked resources:

    drbdadm create-md --stacked r0-U
    
Then, you may enable the stacked resource:

    drbdadm up --stacked r0-U 
    drbdadm primary --force --stacked r0-U
      
Check
    
    cat /proc/drbd
    version: 8.4.5 (api:1/proto:86-101)
    srcversion: 82483CBF1A7AFF700CBEEC5
    0: cs:StandAlone ro:Primary/Unknown ds:UpToDate/DUnknown   r----s
    ns:0 nr:0 dw:40 dr:1592 al:0 bm:0 lo:0 pe:0 ua:0 ap:0 ep:1 wo:f oos:1048508

Set filter in /etc/lvm/lvm.conf

     filter = [ "r|/dev/mapper/VG1-cachedLV_corig|", "r|/dev/mapper/DRBDVG-DRBDLV1|", "r|/dev/drbd0|"]
     
    /etc/init.d/lvm2 restart
    
Create VPS thinpool on top of DRBD device

    pvcreate /dev/drbd10
    vgcreate DRBDVG2 /dev/drbd10
    pvscan

check how much space we have

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

lvresize DRBDVG2/VPSthinpool -L +50M
  Rounding size to boundary between physical extents: 52.00 MiB
  Size of logical volume DRBDVG2/VPSthinpool_tdata changed from 128.00 MiB (32 extents) to 180.00 MiB (45 extents).
  Logical volume VPSthinpool successfully resized

drbdadm -- --assume-peer-has-space resize r0 drbdadm -S -- --assume-peer-has-space resize r0-U

pvresize /dev/drbd10

lvextend -L +50M DRBDVG2/VPSthinpool 
  Rounding size to boundary between physical extents: 52.00 MiB
  Size of logical volume DRBDVG2/VPSthinpool_tdata changed from 180.00 MiB (45 extents) to 232.00 MiB (58 extents).
  Logical volume VPSthinpool successfully resized
root@livenode1:/home/office# drbdadm -- --assume-peer-has-space resize r0
root@livenode1:/home/office# drbdadm -S -- --assume-peer-has-space resize r0-U
root@livenode1:/home/office# pvresize /dev/drbd10
  Physical volume "/dev/drbd10" changed
  1 physical volume(s) resized / 0 physical volume(s) not resized
  
Create container(s) in VPSthinpool on DRBDVG2

    lxc-create -t download -n my-container -B lvm --vgname DRBDVG2 --thinpool VPSthinPool --fssize=500M











