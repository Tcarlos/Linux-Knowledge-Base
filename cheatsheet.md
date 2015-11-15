# LVM

### Install Ubuntu Server with RAID and LVM

### Install and configure LXC

Install LXC

sudo su
apt-get update && apt-get upgrade -y
apt-get install lxc
reboot

Configure LXC as root user

    sudo su

Manually allocate an UID and GID range to root in /etc/subuid and /etc/subgid by adding this line in both files:

    root:100000:65536

Add these same values in global configuration file /etc/lxc/default.conf, by adding these lines to it:

    lxc.id_map = u 0 100000 65536
    lxc.id_map = g 0 100000 65536

Set permissions andownerships:

    chown root:100000 /var/lib/lxc
    chmod 750 /var/lib/lxc

Enable changing memory and SWAP

Modify a line in /etc/default/grub to:

    GRUB_CMDLINE_LINUX_DEFAULT="swapaccount=1"

update grub and reboot:

    update-grub
    reboot

Add these lines to /var/lib/lxc/office1/config:

    lxc.cgroup.memory.limit_in_bytes = 512M
    lxc.cgroup.memory.memsw.limit_in_bytes = 1G


2.4 Useful LXC commands

gives detailed information about a specific container:

    lxc-info -n office1

shows all containers and if they are running or not:

    lxc-ls --fancy


### Create and test LXC container basic

    lxc-create -t download -n office1
    lxc-start -n office1
    lxc-stop -n office1

Enter the container and test networking:

    lxc-attach -n office1
    ping 8.8.8.8
    exit

### Create PVs, VGs and LVs

lvcreate -n data -L 1GB /dev/VG1

### Create a cachepool for a logical volume by using another SSD/HDD

Add the SSD raid to volume group VG1:

    pvcreate /dev/md1
    vgextend VG1 /dev/md1

    pvs
  
    PV         VG   Fmt  Attr PSize  PFree
    /dev/md0   VG1  lvm2 a--  19.98g 6.02g
    /dev/md1   VG1  lvm2 a--   4.99g 4.99g

now make a cachepool on the SSD raid:

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



### Create a thin pool basic

### Cache an LV on another PV

### Create LXC container in a thinpool

now create a physical volume in logical volume cachedlv and create a thinpool in it for lxc lvm containers.

    pvcreate /dev/VG1/cachedlv
  
    vgcreate lxc /dev/VG1/cachedlv

    lvcreate -l 75%FREE --type thin-pool --thinpool lxc lxc

    lxc-create -t download -n lxc_client1 -B lvm --fssize=2G

    vgchange -a y

### Create a snapshot

### Restore a snapsho

### Extend a snapshot

### Reduce a snapshot

### Autoextend snapshots

### Use a snapshot script to make backups automatically



### LVM and DRBD combined
