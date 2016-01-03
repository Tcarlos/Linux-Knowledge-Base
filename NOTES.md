

looking over at the 3 stacked setup I see 4 IPs. After some long guessing, me thinks IP number 3 is the public IP of the primary, the synchronization source, 'the master', the one with the primary force r0 command

### 1

noticed that during a reboot of livenode 5, the cat /proc/drbd of livenode 6 went from 

cat /proc/drbd
version: 8.4.5 (api:1/proto:86-101)
srcversion: 82483CBF1A7AFF700CBEEC5

 1: cs:WFConnection ro:Secondary/Unknown ds:UpToDate/DUnknown C r-----
    ns:0 nr:246700 dw:246700 dr:0 al:0 bm:0 lo:0 pe:0 ua:0 ap:0 ep:1 wo:f oos:0
root@livenode6:/home/office# cat /proc/drbd
version: 8.4.5 (api:1/proto:86-101)
srcversion: 82483CBF1A7AFF700CBEEC5

 1: cs:Connected ro:Secondary/Secondary ds:UpToDate/UpToDate C r-----
    ns:0 nr:250796 dw:250796 dr:0 al:0 bm:0 lo:0 pe:0 ua:0 ap:0 ep:1 wo:f oos:0
   
    

### 2    

 after rebooting livenode 5, only this command is necessairy to enable/activate the blockdevice drbd1, which houses the lxc thinpool
 
 drbdadm primary --force r0
 
 
### 3 
 
 cat /proc/drbd shows green light
 
### 4 more DRBD links:

**Contains data replication testing!!**

https://help.ubuntu.com/lts/serverguide/drbd.html
https://imanudin.net/2015/03/23/testing-data-replicationsynchronize-on-drbd/

**replication data testing requires mounting files, sameway like testing snapshots and the old rsync stuff. but can this also be done with not just a normal mount thing (with a file system?), but also with vps thinpool and/or lxc clients?

https://drbd.linbit.com/users-guide/s-nested-lvm.html
https://blogs.it.ox.ac.uk/jamest/2011/06/03/replicating-block-devices-with-drbd/

HMZ..... first try to test data replication with just a normal mount setup, THEN try to figure out how to test data replication of an LV (thinpool LV with the LXC clients in it)

#### 5 info on lxc mount location

/var/lib/lxc/my-container/config
/etc/lxc/default.conf
lxc.rootfs = /dev/DRBDVG2/my-container2

**mount /dev/mapper/DRBDVG2-my--container2 /mnt**




**2nd UPDATE JAN 2ND**

Didnt got the basic setup working again. Pondered and pondered, till I discovered that the commands drbdadm create-md r0 and drbdadm up r0 most be done on BOTH nodes..
At least I got some xp on reproducing and updated the documentation.
So, during my vacation I disovered richard has a seemingly non working 2 node setup, and I discovered that the above commands must be done on both nodes.

Still lots of work to do, namely:

- update documentation 
- test richards '2 node setup', just in case
- make a 3 node drbd setup at home
- now that we have confirmed synchronization, find out when it syncs and how long, (see testcase 3, chapter 4.1.6.3)
- make a backup script that makes a snapshot before the syncing takes place
- test that data written on livenode 5 is synced and visible on livenode 6!! IF not working with VPS try with easier data instead of VPS thinpool!!!!!
- study drbd common tasks drbd.init.com chapter 6, for instance drbd resize
- -FIND MOUNT LOCATION OF LXC CONTAINER!!! AND DOCUMENT

b

       resource r0 {
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




**UPDATE JAN 1ST**

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
