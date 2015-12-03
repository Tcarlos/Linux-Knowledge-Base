
### Updates 3 dec 

figuring out how to exactly change the config files, little examples online.
The plan ahead 0 works in the global_common.conf


#### Update 2 december

In fixed-rate synchronization, the amount of data shipped to the synchronizing peer per second (the synchronization rate) has a configurable, static upper limit. Based on this limit, you may estimate the expected sync time based on the following simple formula:

    Tsync = D/R

tsync is the expected sync time. D is the amount of data to be synchronized, which you are unlikely to have any influence over (this is the amount of data that was modified by your application while the replication link was broken). R is the rate of synchronization, which is configurable — bounded by the throughput limitations of the replication network and I/O subsystem.

**IMPORTANT** It does not make sense to set a synchronization rate that is higher than the maximum write throughput on your secondary node. You must not expect your secondary node to miraculously be able to write faster than its I/O subsystem allows, just because it happens to be the target of an ongoing device synchronization.



Choosing the more simple and more absolute fixed rate sycnhrhonizing for now

### Update nov 30th 

discovered all LVs boot fine at reboot, it's just the DRBD that doesnt load at boot, with the consequence of not loading the VPS thinpool and its LXC clients
discovered that only a few drbd commands are needed to get it all running again
lxc containers dont start automatically at startup, wether in a DRBD device or not

conclusion: 

- Create a script that activates the DRBD device, followed by the lxc-start command of VPSthinpool containers!
- We can continue working


** UPDATE **

the new challenge is to snapshot the drbd before it syncs. 

- what for drbd thing do i have (https://drbd.linbit.com/users-guide/s-three-nodes.html)

comparing the drbd setup of this testcase 3 with the above links seems to imply i have a two node setup.

If i want to know how to make manual snapshots (before synching) I need to analyze these scripts and/or use common sense combined with the knowledge when synching occurs.


