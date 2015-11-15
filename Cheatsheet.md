## 1. Install Ubuntu Server with RAID and LVM

## 2. Install LXC

sudo su
apt-get update && apt-get upgrade -y
apt-get install lxc
reboot

## 3. Configure LXC

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





## 3. Create and test LXC container basic

    lxc-create -t download -n office1
    lxc-start -n office1
    lxc-stop -n office1

Enter the container and test networking:

    lxc-attach -n office1
    ping 8.8.8.8
    exit
    
## 4.  Useful LXC commands

Gives detailed information about a specific container:

    lxc-info -n office1

Shows all containers and if they are running or not:

    lxc-ls --fancy

### 5. Create PVs, VGs and LVs

    lvcreate -n data -L 1GB /dev/VG1
    
## 6. Thinpools

### 6.1 Basic Thinpools

### 6.2 Create LXC containers in a thinpool

    pvcreate /dev/VG1/cachedlv
  
    vgcreate lxc /dev/VG1/cachedlv

    lvcreate -l 75%FREE --type thin-pool --thinpool lxc lxc

    lxc-create -t download -n lxc_client1 -B lvm --fssize=2G

    vgchange -a y

## 7. Create a cachepool for a logical volume by using another SSD/HDD

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

## 8. LVM Snapshots

### 8.1 Create a snapshot

    lvcreate -L 1GB -s -n [SNAPSHOT_NAME] [LV_PATH]

    lvcreate -L 1GB -s -n rootlv_snap /dev/VG1/rootlv

### 8.2 Restore LV with a snapshot

    lvconvert --merge /dev/VG1/data2_snap

Confirm data LV has been restored, one way is to mount and list

    mount /dev/VG1/data /data
    ls -la /data
    
Another way is checking diskspace

    df -Th
    
### 8.3 Extend a snapshot

    lvextend -L +1G /dev/VG1/rootlv_snap

### 8.4 Reduce a snapshot

### 8.5 Autoextend snapshots

### 8.6 Use a snapshot script to make backups automatically

## 9. LVM and DRBD combined