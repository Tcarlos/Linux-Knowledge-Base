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
