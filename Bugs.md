## 7. Bugs

## Bug #1

lxc lvm container: cant start containers after reboot, 

    lxc-start: bdev.c: mount_unknown_fs: 210 failed to determine fs type for '/dev/lxc/lxc_client2'
    lxc-start: conf.c: mount_unknown_fs: 525 failed to determine fs type for '/dev/dm-10'
    lxc-start: conf.c: setup_rootfs: 1280 failed to mount rootfs
    lxc-start: conf.c: do_rootfs_setup: 3713 failed to setup rootfs for 'lxc_client2'
    lxc-start: start.c: __lxc_start: 1155 Error setting up rootfs mount as root before spawn
    lxc-start: lxc_start.c: main: 344 The container failed to start.

### Bug #2

    lxc-start -n lxc_client1 -F
    lxc-start: conf.c: mount_rootfs: 873 No such file or directory - failed to get real path for '/dev/lxc/lxc_client1'
    lxc-start: conf.c: setup_rootfs: 1280 failed to mount rootfs
    lxc-start: conf.c: do_rootfs_setup: 3713 failed to setup rootfs for 'lxc_client1'
    lxc-start: conf.c: lxc_setup: 3795 Error setting up rootfs mount after spawn
    lxc-start: start.c: do_start: 699 failed to setup the container
    lxc-start: sync.c: __sync_wait: 51 invalid sequence number 1. expected 2
    lxc-start: start.c: __lxc_start: 1164 failed to spawn 'lxc_client1'

**update**

googled and got solutions to check idmap lines, done so already, or permissions, also.
Perhaps it's this https://github.com/lxc/lxc/issues/406, this suggests it could be a template matter.
A workaround could be to make a snapshot and merge right away to restore but to no avail
Do note however, that the new container has no problem. So it could be that the first two just dont work coz the whole LV setup got changed between client1/client2 versus client3.
**Update** make client4 and monitor the workings of client3 and client4. If after a few days there is still no error, we can discard this bug as a child of wrong config, a situation that is obsolete coz with a working system as i have now there are simply no problems.


