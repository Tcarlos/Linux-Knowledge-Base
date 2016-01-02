** 2nd UPDATE JAN 1st **

WORKS AGAIN!

note that I also done on livenode6:

drbdadm create-md r0
drbdadm up r0

LIVENODE5 output:

root@livenode5:/home/office# drbdadm create-md r0
^C
root@livenode5:/home/office# nano /etc/drbd.d/r0.res
root@livenode5:/home/office# drbdadm create-md r0
md_offset 10737414144
al_offset 10737381376
bm_offset 10737053696

Found LVM2 physical volume signature
    10481664 kB data area apparently used
    10485404 kB left usable by current configuration

Even though it looks like this would place the new meta data into
unused space, you still need to confirm, as this is only a guess.

Do you want to proceed?
[need to type 'yes' to confirm] yes

initializing activity log
NOT initializing bitmap
Writing meta data...
New drbd meta data block successfully created.
root@livenode5:/home/office# drdbadm r0
No command 'drdbadm' found, did you mean:
 Command 'drbdadm' from package 'drbd-utils' (main)
drdbadm: command not found
root@livenode5:/home/office# drbdadm up r0
root@livenode5:/home/office# cat /proc/drbd
version: 8.4.5 (api:1/proto:86-101)
srcversion: 82483CBF1A7AFF700CBEEC5

 1: cs:Connected ro:Secondary/Secondary ds:Inconsistent/Inconsistent C r-----
    ns:0 nr:0 dw:0 dr:0 al:0 bm:0 lo:0 pe:0 ua:0 ap:0 ep:1 wo:f oos:10485404
root@livenode5:/home/office# drbdadm primary --force r0
root@livenode5:/home/office# cat /proc/drbd
version: 8.4.5 (api:1/proto:86-101)
srcversion: 82483CBF1A7AFF700CBEEC5

 1: cs:SyncSource ro:Primary/Secondary ds:UpToDate/Inconsistent C r-----
    ns:17872 nr:0 dw:0 dr:18152 al:0 bm:0 lo:0 pe:0 ua:0 ap:0 ep:1 wo:f oos:10467532
        [>....................] sync'ed:  0.2% (10220/10236)M
        finish: 0:58:09 speed: 2,976 (2,976) K/sec


** UPDATE JAN 1st **

The three stacked setup only works with a working DRBD device, stacked upon another. Richards setup doesnt have a working DRBD device, the stacked resource is only on one node, and one node isnt a working DRBD device.

So, I need to create a 3rd node back home on Virtual Box, in the meantime I can fairly say the basic 2 node setup will work with the LXC shebang, since Dec 23 update already has the basic setup on top of the wellknown thinpool setup as described in testcase 3.



**NOTES DECEMBER 31st**

Trying to implement the new DRBD knowledge in testcase 3.

- Discovered that to remove the LV's under the DRBD device, first do drbdadm down r0
- Comparing Richards setup with the three node setup, I can conclude that Richards setup is one node, that is connected to a backup node, with protocol A, which is useful for long distance replication according to  https://drbd.linbit.com/users-guide/s-replication-protocols.html. 


Ok, create a thinpool on PV/VG thingy


**Create PV & VG**

        pvcreate /dev/VG1/cachedlv
        vgcreate DRBDVG /dev/VG1/cachedlv

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
        
Now, we will create richards DRBD device:

        resource r0 {
         on livenode5 {
         device /dev/drbd0;
         disk /dev/mapper/DRBDVG-DRBDLV1;
         address 10.1.1.31:7789;
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

        on livenode6 {
        device /dev/drbd10;
        disk /dev/mapper/DRBDVG-DRBDLV1;
        address 10.1.1.32:7794;
        meta-disk internal;
        }
        }

** UPDATE **

The three stacked setup only works with a working DRBD device, stacked upon another. Richards setup doesnt have a working DRBD device, the stacked resource is only on one node, and one node isnt a working DRBD device.








**NOTES DECEMBER 23rd**

Have a succesful working basic DRBD 2node syncing setup

Links used

        https://drbd.linbit.com/users-guide/ch-configure.html
        https://drbd.linbit.com/users-guide/s-prepare-network.html
        https://drbd.linbit.com/users-guide/s-configure-resource.html
        https://drbd.linbit.com/users-guide/s-first-time-up.html
        https://drbd.linbit.com/users-guide/s-initial-full-sync.html

### 1. Prepare low level storage

Fresh after Ubunutu Server install we have 4 Logical Volumes: rootlv, bootlv, swaplv, cachedlv. On cachedlv we create a PV, with a VG on top of it, and on that we will create a thinpool as backingdevice for the DRBD device.

**Create PV & VG**

        pvcreate /dev/VG1/cachedlv
        vgcreate DRBDVG /dev/VG1/cachedlv

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
        
**NOW CREATE A 2nd NODE WITH THE EXACT SAME SETUP!**

### 2. Preparing network configuration

Create on our two DRBD hosts each a currently unused network interface, eth1, with IP addresses 10.1.1.31 and 10.1.1.32 assigned to it, respectively, by editing a file in /etc/network/interfaces and the settings of the VM in VirtualBox.

**Add these lines in /etc/network/interfaces**

**NODE 1, livenode5**

        auto eth1
        iface eth1 inet static
          address 10.1.1.31
          netmask 255.255.255.0

**NODE 2, livenode6**

        auto eth1
        iface eth1 inet static
          address 10.1.1.32
          netmask 255.255.255.0
          
**VM configuration** 

Now shutdown the VMs, rightclick -> settings -> Network -> adapter2 tab.

        Attached to: Internal Network
        name: intn
        promiscious mode: allow all

**Restart the VMs**


### 3. Configure DRBD (on both livenode5 and livenode6)

        touch /etc/drbd.d/r0.res


        resource r0 {
           on livenode6 {
              device    /dev/drbd1;
              disk      /dev/mapper/DRBDVG-DRBDLV1;
              address   10.1.1.32:7789;
              meta-disk internal;
           }
           on livenode5 {
                device    /dev/drbd1;
                disk      /dev/mapper/DRBDVG-DRBDLV1;
                address   10.1.1.31:7789;
                meta-disk internal;
           }
        }

### 4. Enabling resource for first time

**Create device meta data**

        drbdadm create-md r0
        initializing activity log
        NOT initializing bitmap
        Writing meta data...
        New drbd meta data block successfully created.
        
**Enable the resource**

        drbdadm up r0
        
#### 5. Initial device synchronization

        drbdadm primary --force r0

After issuing this command, the initial full synchronization will commence. You will be able to monitor its progress via /proc/drbd. It may take some time depending on the size of the device.

By now, your DRBD device is fully operational, even before the initial synchronization has completed (albeit with slightly reduced performance). You may now create a filesystem on the device, use it as a raw block device, mount it, and perform any other operation you would with an accessible block device.
        
        cat /proc/drbd
        
This output now shows a sync in progress, and when its expected to be done.


Now that this working well, I'm confident I can implement this in the more advanced testcase3. In there I'll add some more sync testing stuff, and with the sync now clearly visible in /proc/drbd it wont be too far from writing scripts that backup before synching takes place  \o/






**NOTES DE 15th**


[16:33]

Couldnt create snapshots of a thin volume designated LV, but could do it with a normal LV. The LXC thin volume is snapshottable however.
Review with Richard and/or see how to create a thin pool under the DRBD that is availbe for snapshots, if needed at all.


        LXC1        <--- VPS thinpool that is available for snapshots

        VPSthinpool <--- Thinpool for LXC clients (designated LV)

        DRBDVG2 <--- VG on top of PV

        PV <---- pvcreate /dev/drbd10

        DRBD DEVICE

        DRBDLV1 <--- LV, DRBD backing device, available for snapshots

        DRBDVG  <---- VG on top of PV

        PV      <---- pvcreate /dev/VG1/cachedlv

        cachedlv <--- containing all of the above, on a HDD RAID; is cached on a SSD RAID



Why use only half of VG DRBDVG2?




- make testcase 3 about DRBD and/or create testcase 4 describing the whole situation
- add more descriptions and more functions e.g. now that we have this working we can begin to work with 

        making snapshots of various stages
        recovering from snapshots
        resizing of various stages
