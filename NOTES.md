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
