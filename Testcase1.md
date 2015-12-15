# TESTCASE1: test the process of creating and restoring snapshots

Create an LV and confirm creation

    lvcreate -L 1GB -n data  /dev/VG1
    lvs
    
Make a filesystem on it, mount it and confirm mount

    mkfs.ext4 /dev/VG1/data
    mkdir /data
    mount /dev/VG1/data /data
    df -h

Make some files on it and confirm presence of files

    cd /data && touch moo foo shoo
    ls -la
    

Create a 1GB snapshot of LV data and confirm

    lvcreate -L 1G -s -n data_snap /dev/VG1/data
    lvs

Mount the snapshot to confirm snapshot has the same files as LV data

    mkdir /data_snap
    mount /dev/VG1/data_snap /data_snap
    ls -la /data_snap

Make more files on LV data, and confirm /data and /data_snap have a difference in files

    cd /data && touch MOOOO FOOOO SHOOO
    ls -la /data
    ls -la /data_snap

Unmount the data LV and the snapshot and confirm

    umount /data
    umount /data_snap
    df -h
    
Restore data LV with the data_snap snapshot

    lvconvert --merge /dev/VG1/data2_snap

Confirm data LV has been restored to the situation that has only 3 files

    mount /dev/VG1/data /data
    ls -la /data

