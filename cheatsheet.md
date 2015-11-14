# LVM

### Install Ubuntu Server with RAID and LVM

### Install and configure LXC

sudo su
apt-get update && apt-get upgrade -y
apt-get install lxc
reboot
2.3 LXC configuration (as root user)
sudo su
manually allocate an UID and GID range to root in /etc/subuid and /etc/subgid by adding this line in both files:

root:100000:65536
add these same values in global configuration file /etc/lxc/default.conf, by adding these lines to it:

lxc.id_map = u 0 100000 65536
lxc.id_map = g 0 100000 65536
set permissions andownerships:

chown root:100000 /var/lib/lxc
chmod 750 /var/lib/lxc
creating and testing container:

lxc-create -t download -n office1
lxc-start -n office1
lxc-stop -n office1
get inside the container and test networking:

lxc-attach -n office1
ping 8.8.8.8 (WAIT A FEW SECONDS!)
exit
2.4 Useful LXC commands
gives detailed information about a specific container:

lxc-info -n office1
shows all containers and if they are running or not:

lxc-ls --fancy
2.5 Changing memory and SWAP
modify a line in /etc/default/grub to:

GRUB_CMDLINE_LINUX_DEFAULT="swapaccount=1"
update grub and reboot:

update-grub
reboot
Add these lines to /var/lib/lxc/office1/config:

lxc.cgroup.memory.limit_in_bytes = 512M
lxc.cgroup.memory.memsw.limit_in_bytes = 1G

### Create LXC container basic

### Create PVs, VGs and LVs

lvcreate -n data -L 1GB /dev/VG1

### Create a thin pool basic

### Cache an LV on another PV

### Create LXC container in a thinpool

### Create a snapshot

### Restore a snapsho

### Extend a snapshot

### Reduce a snapshot

### Autoextend snapshots

### Use a snapshot script to make backups automatically



### LVM and DRBD combined
