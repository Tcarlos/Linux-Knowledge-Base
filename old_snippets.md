**Create a thinpool on top of that**

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
    
    
The mentioned commands should be put in a script so we have automation.


**set automatic LVM snapshots on:**

Open the global common file and enter the parameters below:

    nano /etc/drbd.d/global_common.conf

    resource r0 {
      handlers {
        before-resync-target "/usr/lib/drbd/snapshot-resync-target-lvm.sh";
        after-resync-target "/usr/lib/drbd/unsnapshot-resync-target-lvm.sh";
        }
    }
    
Manual snapshotting will follow.
    
**set fixed synchronization rate:**

Also in the global common file, make the following entries in the disk section:

    disk {

                c-plan-ahead 0;
                resync-rate 33M;
                # size on-io-error fencing disk-barrier disk-flushes
                # disk-drain md-flushes resync-rate resync-after al-extents
                # c-plan-ahead c-delay-target c-fill-target c-max-rate
                # c-min-rate disk-timeout
            }


#### Testing automatic snapshots

The snapshot-resync-target-lvm.sh script creates an LVM snapshot for any volume the resource contains **immediately before** synchronization kicks off. In case the script fails, the synchronization does not commence

^^ So this script will do his work every time the DRBD syncrhonizes, **which is set by resync-rate 33M**

Once synchronization completes, the unsnapshot-resync-target-lvm.sh script removes the snapshot, which is then no longer needed. In case unsnapshotting fails, the snapshot continues to linger around.

So a snapshot will remain if unsnapshot-resync-target-lvm.sh fails to remove it. 
So for this test we will hack above script in such a way it fails to remove the snapshot.

#### Measuring when synching occurs

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

Now

### Measuring I/O
