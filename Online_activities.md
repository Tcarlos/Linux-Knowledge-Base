Here I'll show you all my interactions with forums. 
Also I'll display lists of websites where you can interact with the community. Help and get helped.
Lastly I will give updates from time to time so you can see hows it going.
Pay it forward!

### TO DO:

- Start using Twitter again, as a micro blogging site of a junior system admin, wipe history because not relevant.

#### Jan 3rd - Mail interaction with Linbit

Already solved the issue before Linbit replied.

From: 	linbit.com <webmaster@linbit.com>
> To: 	sales@linbit.com, sales_us@linbit.com
> 
> 
> *Email* 	bushiri@gmail.com
> *Phone Number* 	
> *Details* 	Hello Linbit,
> 
> I wondered if you could help a junior system admin with a question about
> DRBD. I got a very basic DRBD setup running between 2 nodes, as described in
> your example configuration
> (https://drbd.linbit.com/users-guide/s-configure-resource.html). On top of
> the DRBD device I got a Physical Device, containing a VG, containing several
> Logical Volumes. My question is about testing synchronization with the
> method of making a file on one node and checking if it gets on the other
> node as well, like this link does (https://imanudin.net/2015/03/23/testing-data-replicationsynchronize-on-drbd/).
> 
> But, how can you test synchronization if you have LVM on top of the DRBD
> device? I got the following error:
> 
> root@livenode5:/mnt# mount /dev/drbd1 /mnt
> mount: unknown filesystem type 'LVM2_member'
> 
> I'm realising this might be better asked on a LVM forum or something but I
> hope you guys can give some pointers
well, you "mount" a filesystem; for a PV you'll have to run eg. "vgscan", 
to make LVM detect the PV. This should find the VG; then "lvchange -ay" 
would activate the volumens, and then you should be able to mount the LV.

>If "vgscan" doesn't find the PV, your filter definition in 
>/etc/lvm/lvm.conf might need to be changed.


>Yes, LVM fora would be the right place to ask ;)


