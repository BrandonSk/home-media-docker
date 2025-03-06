**********************************************
**                                          **
**  Proxmox with better than default BTRFS  **
**                                          **
**********************************************

Installing proxmox with btrfs and subvolumes, snapper automated snapshots, and auto-grub...
(Credits: Parts of this guide are based on a post you can find here: https://medium.com/@inatagan/installing-debian-with-btrfs-snapper-backups-and-grub-btrfs-27212644175f)

What do we get at the end?
We will have proxmox installed on a btrfs filesystem. Within this filesystem we will define several subvolumes. For the root subvolume (@) we will configure automatic snapshots via snapper and also automatic updates of grub menu based on these snapshots. So in case your system is severly damaged, you can easily boot into prior snapshot from the grub menu.

Where does this fit within Ultimate Home Media Server series?
This is an alternative step of installing the Proxmox system. You can continue by installing the Debian (or your prefered distro) either into Proxmox VM, or as LXC.
Therfore this guide is fully compatible and does not require any modifications from the rest of guides.

What are hardware requirements?
Nothing special. Just a system on which you plan running proxmox :)
I am using 2 SSDs drives (sda, sdb) and will be setting-up proxmox on SDDs in RAID1 configuration.
If you have only single SSD or HDD, go along this guide, only choose RAID0 during setup. Later I'll show you how to change the btrfs setup from RAID0 to single, which proxmox installer does not offer via menu...

How is this guide structured and why?
This guide has X stages.
Stage 1 - Standard Proxmox installation
Stage 2 - Modification of btrfs created during installation
Stage 3 - Installation of automation tools Snapper and Grub-btrfs

Stage 2 is the meat of this guide. After installation of Proxmox, we will boot the same machine into some Live linux distro (I used Ubuntu Live) to make modifications. This whole stage is necessary for one simple reason - Proxmox does not give you an opportunity to modify btrfs during its installation process. Once you confirm the installation configuration, Proxmox paritions drives and starts copying files. This is different comparing to Debian expert install, where after partitioning of disks and writing changes to them you still have an option to drop to shell and make manual modifications before the system is installed.
Since we do not have this possibility in Proxmox installer, we must do everything ex-post.

Stage 3 is fully optional. It is performed back at the Proxmox system. So the live distro is applicable only in previous stage. If you do not want any kind of automation, you can completely skip this stage and make snapshots manually.

Final notes before we begin:
1) Snapshots are not backups... they give you an option to roll back in time in case something goes wrong.
2) This guide aims to provide snapshots for the Proxmox system(!), i.e. it has nothing to do with docker, etc. In fact, we are excluding certain paths (which become subvolumes) from automated snapshots (but you may automate it of course, as shown later).
3) This guide does not explain everything regarding btrfs. There are many guides and videos on the internet, which you may read-up. So while in someplaces I do explain the logic and what we are doing, I am not going into depth of btrfs and its options.
4) Configuration for automation you find bellow is according to my preferences. Feel free to adjust it.
I do not need many snapshots. I follow roughly this logic:
- I basically do not need hourly snapshots, because there is little chance I'll make use of them on server. Despite that, I make 2 hourly snapshots in case I am working on the server. If I do something wrong, those 2 hourly snapshots should be sufficient for me to notice problem immediately and roll back.
- I have one daily snapshot, in case those hourly snapshots do not suffice.
- I also have 2 weekly snapshots - these I consider the "main" snapshots, as based on frequency of using my server I should discover within this timeframe that there is something wrong.
- And lastly I have 1 monthly snapshot, in case those 2 weekly are not sufficient for rollback.
- On top of that, I also create manual snapshots with clean system after installation.

Let's go.

*************************************************************
Stage 1 - Standard Proxmox installation

There is nothing special to do in this stage. Just follow the installer...
1) Start Proxmox installer and proceed the usual way until partitioning.
2) For partition, choose BTRFS file system, RAID 1 configuration.
3) Deselect all disks, and then for Disk 1 choose sda, and for Disk 2 choose sdb.
4) On advanced options you may choose a compression if you wish. I went with zstd.
5) Continue installation standard way and reboot.

6) After reboot, do not go to browser as instructed. At this point, we need to reboot once more and instead of booting into the proxmox, we will boot into some Live CD distro, e.g. Debian or Ubuntu.
*Note: It is possible to perform what follows in Stage 2 also on proxmox system, e.g. mounting / as read-only, create subvolumes, moving files, etc., but it becomes cumbersome, as fstab would have to be fixed either via grub-rescue or yet again another live cd. So the proposed way is in my opinion the easiest.*

*************************************************************
Stage 2 - Modification of btrfs created during installation

Once booted into the live CD, run terminal and check, that we have btrfs filesystem available:
```sh
sudo btrfs filesystem show
```
```sh
ubuntu@ubuntu:~$ sudo btrfs filesystem show
Label: none  uuid: 0f3af461-d815-4129-8cea-43bb23ccd944
	Total devices 2 FS bytes used 2.52GiB
	devid    1 size 237.47GiB used 5.01GiB path /dev/sda3
	devid    2 size 237.47GiB used 5.01GiB path /dev/sdb3
```
*Write down the UUID, we will need it later.*

As you can see, proxmox has created filesystem on our two SSDs. As they are in RAID1, the structure is the same. The first two partitions on each ssd are BIOS boot and EFI system.

I will further structure this stage into steps.
What we will do next:
Step 1 - Create temporary mount of btrfs filesystem
Step 2 - Move the / data
Step 3 - Create additional subvolumes and move respective data there.
Step 4 - Modify fstab.
Step 5 - Update grub.
Step 6 - Reboot into our new system. 

______________________
Step 1 - Create temporary mount of btrfs filesystem

Here we will mount the btrfs filesystem into /mnt directory. If you are new to btrfs, this may seem confusing at first. But for simplicity think of it this way - the filesystem is the topmost space, into which we will put subvolumes, and in the subvolumes, will be our data. The way Proxmox does it is, that it does not create separate subvolume for / directory, but puts it directly into the topmost space. This is not according to what other distros do and we will modify this in this and the next step.

First let's mount the topmost space into mnt.
```sh
sudo mount -t btrfs -o defaults /dev/disk/by-uuid/0f3af461-d815-4129-8cea-43bb23ccd944 /mnt
ls /mnt
```
*Note: Use your UUID of course, which you noted earlier.*

You should see structure of a / in your /mnt directory.

______________________
Step 2 - Move the / data

Create our first subvolume called '@' and move all / data into it.
```sh
cd /mnt
# Create subvolume
sudo btrfs subvolume create @
# Move all data from /
sudo mv ./* @/
sudo mv ./.* @/
```
*Note: probably the first move command is sufficient as proxmox (as of now) does not have any hidden files or folders directly under /*
*Note 2: You may see some warnings as result of no hidden files or folders, but ignore them.*

______________________
Step 3 - Create additional subvolumes and move respective data there.

We will create a bunch of additional subvolumes. Why so many? The full explanation is beyond the scope of this guide, but in simple terms - we are doing it because of snapshots. When doing snapshot of a subvolume in btrfs (let's say our root subvolume @), the snapshot will skip all data contained in other subvolumes, regardless where they are mounted. So if I create separate subvolume @home for /home, then this does not get snapshot when I do snapshot of @, and vice versa. 
This conserves a lot of space. While of course you do want to do snapshots of @home (separately of course), you do not need to snapshot bunch of files in /var/log, /var/spool, etc... You get the point now.

So let's go (we are still in /mnt):
```sh
# Create subvolumes
sudo btrfs subvolume create @snapshots
sudo btrfs subvolume create @home
sudo btrfs subvolume create @varlog
sudo btrfs subvolume create @vartmp
sudo btrfs subvolume create @varcache
sudo btrfs subvolume create @varspool
sudo btrfs subvolume create @tmp
```
You can check our subvolumes structure by issuing:
```sh
sudo btrfs subvolume list .
```
Do not worry about the subvolume @/var/lib/pve/local-btrfs which was created and is managed by proxmox itself.

The @snapshots subvolume will hold our snapshots. But we do not have mountpoint (/.snapshots) for it yet, so let's create it. 
```sh
sudo mkdir -p ./@/.snapshots
```
Except for the just created .snapshots directory, we now need to move the content of all relevant directories into their file system subvolumes. Let's briefly explain: 
Until now, all proxmox files sit under the @ subvolume (we moved them there in step 2). So for example the log files are in the ./@/var/log directory of the @ subvolume. This is not what we want. We will now move these files from ./@/var/log into the @varlog subvolume (think of it as a "partition", although it is not a partition) and then mount this @varlog subvolume into the @/var/log directory. Wait...WHAT?

Yes, from proxmox system perspective, once we boot into proxmox, the files will still be located at this path /var/log. But from filesystem perspective, while the **directory** "/var/log" is located in @ subvolume (again, think of it as "partition"), the **content** of /var/log is located in @varlog subvolume (another "partition"). And later we will mount this content from @varlog subvolume into the /var/log directory.

So first, let's do the moving (we are still in /mnt !):
```sh
sudo mv ./@/var/log/* @varlog/
sudo mv ./@/var/log/.* @varlog/
```
*Note: we do 2 move commands, to make sure also the hidden files and folders are moved*

Now our @/var/log directory (which will become /var/log after reboot) is empty and all the log files are sitting "somewhere" in the filesystem. That somewhere (@varlog subvolume) will be mounted to the /var/log in fstab (see next step).

Feel free to ls the different paths (./@/var/log/ and ./@varlog/) to confirm and get better understanding.

Now let's move the rest and populate our subvolumes...
```sh
sudo mv ./@/home/* @home/
sudo mv ./@/home/.* @home/
sudo mv ./@/var/tmp/* @vartmp/
sudo mv ./@/var/tmp/.* @vartmp/
sudo mv ./@/var/cache/* @varcache/
sudo mv ./@/var/cache/.* @varcache/
sudo mv ./@/var/spool/* @varspool/
sudo mv ./@/var/spool/.* @varspool/
sudo mv ./@/tmp/* @tmp/
sudo mv ./@/tmp/.* @tmp/
```
*Note: You may encouter errors/warnings when executing above commands about device/resource being busy. Just ignore them. It's triggered by the fact that there is nothing to move in some of the direcotries.*

In the next step, we will modify fstab file to correctly mount these subvolumes to their repsective directories on next proxmox boot.

______________________
Step 4 - Modify fstab

We are still in /mnt (!)
Since we have moved all files from /mnt (the topmost space of our btrfs filesystem) to @ subvolume, out fstab file is now at this location ./@/etc/fstab
```sh
sudo nano ./@/etc/fstab
```

You should see content similar to this:
```sh
# <file system> <mount point> <type> <options> <dump> <pass>
UUID=0f3af461-d815-4129-8cea-43bb23ccd944 / btrfs defaults,compress=zstd 0 1
proc /proc proc defaults 0 0
```

The line that starts with UUID is the one that mounted our / after proxmox installation. We will first modify this line and then duplicate it 7 times to accommodate all of our subvolumes.

Modify the line so it looks like this:
# <file system> <mount point> <type> <options> <dump> <pass>
```sh
UUID=0f3af461-d815-4129-8cea-43bb23ccd944 / btrfs defaults,noatime,ssd,discard=async,compress=zstd,subvol=@ 0 1
```
Notice, that we have added these parameters in options column: noatime,ssd,discard=async,subvol=@
If you have diferent preference, feel free to modify if you know what you're doing.

One thing you cannot modify is the MAIN CHANGE we are making -> subvol=@ parameter. This is the magic part, where we are telling the system that the subvolume @ should be mounted as /
It will be very similar for the remaining subvolumes.

With cursor on the UUID line, press CTRL+K (if you are using nano) and then 8 times CTRL+U. The line will duplicate.
Now move on the second line starting with UUID. Please note that UUID stays the same for all subvolumes. It is the id of the filesystem, in which our subvolumes reside. Modify each line so the result looks something like this:
```sh
UUID=0f3af461-d815-4129-8cea-43bb23ccd944 / btrfs defaults,noatime,ssd,discard=async,compress=zstd,subvol=@ 0 1
UUID=0f3af461-d815-4129-8cea-43bb23ccd944 /.snapshots btrfs defaults,noatime,ssd,discard=async,compress=zstd,subvol=@snapshots 0 1
UUID=0f3af461-d815-4129-8cea-43bb23ccd944 /home btrfs defaults,noatime,ssd,discard=async,compress=zstd,subvol=@home 0 1
UUID=0f3af461-d815-4129-8cea-43bb23ccd944 /var/log btrfs defaults,noatime,ssd,discard=async,compress=zstd,subvol=@varlog 0 1
UUID=0f3af461-d815-4129-8cea-43bb23ccd944 /var/cache btrfs defaults,noatime,ssd,discard=async,compress=zstd,subvol=@varcache 0 1
UUID=0f3af461-d815-4129-8cea-43bb23ccd944 /var/spool btrfs defaults,noatime,ssd,discard=async,compress=zstd,subvol=@varspool 0 1
UUID=0f3af461-d815-4129-8cea-43bb23ccd944 /var/tmp btrfs defaults,noatime,ssd,discard=async,compress=zstd,subvol=@vartmp 0 1
UUID=0f3af461-d815-4129-8cea-43bb23ccd944 /tmp btrfs defaults,noatime,ssd,discard=async,compress=zstd,subvol=@tmp 0 1
```
For each line we are modifying the mountpoint and the subvol parameter.
Save the file and exit nano.
So we are almost ready, but we need to tell grub that things have changed a little bit.

______________________
Step 5 / Modify grub

First some preparation. We need to unmount currently mounted filesystem in /mnt and then mount the @ subvolume there:
```sh
cd /
sudo umount -l /mnt
# Mount the @ subvolume to /mnt/
sudo mount /dev/disk/by-uuid/0f3af461-d815-4129-8cea-43bb23ccd944 /mnt/ -t btrfs -o subvol=@
# For grub, we will need these additional mounts
for i in dev dev/pts sys proc run; do sudo mount --bind /$i mnt/$i; done
# UEFI ONLY
# First check (sudo fdisk -l) where is your EFI partition. For me it is /dev/sda2
sudo mount /dev/sda2 /mnt/boot/efi
```

Now we chroot into the /mnt to convice grub to do its job :)
```sh
sudo chroot /mnt
# Reinstall boot loader
grub-install
# Update grub config
update-grub
# Leave chroot
exit
```

Finally we should unmount our temporary mounts:
```sh
sudo umount /mnt/boot/efi
for i in run proc sys dev/pts dev; do sudo umount mnt/$i; done
sudo umount /mnt
```

And that should be it. Since you are in a live bistro, remove the installation media if needed and reboot!

______________________
Step 6 - Reboot
```sh
sudo reboot
```

*************************************************************
Stage 3 - Installation of automation tools Snapper and Grub-btrfs

Remember - this stage is about automation and is completely optional.

If all went well you should easily boot to proxmox.
Stay in the console or ssh to the host.

Steps in this stage:
Step 1 - Verification
Step 2 - Snapper and its configuration
Step 3 - Grub autoconfiguration via grub-btrfs

______________________
Step 1 - Verification

Lets verify, that everything works:
```sh
# Check subvolumes of root /
root@hppl:~# btrfs subvolume list /
ID 256 gen 23 top level 257 path var/lib/pve/local-btrfs
ID 257 gen 417 top level 5 path @
ID 258 gen 377 top level 5 path @snapshots
ID 259 gen 382 top level 5 path @home
ID 260 gen 417 top level 5 path @varlog
ID 261 gen 399 top level 5 path @vartmp
ID 262 gen 414 top level 5 path @varcache
ID 263 gen 399 top level 5 path @varspool
ID 264 gen 413 top level 5 path @tmp

# Check mountpoints
root@hppl:~# mount | grep @
/dev/sda3 on / type btrfs (rw,noatime,compress=zstd:3,ssd,discard=async,space_cache=v2,subvolid=257,subvol=/@)
/dev/sda3 on /.snapshots type btrfs (rw,noatime,compress=zstd:3,ssd,discard=async,space_cache=v2,subvolid=258,subvol=/@snapshots)
/dev/sda3 on /home type btrfs (rw,noatime,compress=zstd:3,ssd,discard=async,space_cache=v2,subvolid=259,subvol=/@home)
/dev/sda3 on /tmp type btrfs (rw,noatime,compress=zstd:3,ssd,discard=async,space_cache=v2,subvolid=264,subvol=/@tmp)
/dev/sda3 on /var/cache type btrfs (rw,noatime,compress=zstd:3,ssd,discard=async,space_cache=v2,subvolid=262,subvol=/@varcache)
/dev/sda3 on /var/tmp type btrfs (rw,noatime,compress=zstd:3,ssd,discard=async,space_cache=v2,subvolid=261,subvol=/@vartmp)
/dev/sda3 on /var/log type btrfs (rw,noatime,compress=zstd:3,ssd,discard=async,space_cache=v2,subvolid=260,subvol=/@varlog)
/dev/sda3 on /var/spool type btrfs (rw,noatime,compress=zstd:3,ssd,discard=async,space_cache=v2,subvolid=263,subvol=/@varspool)
```
What I like to do as very first thing is to manually snapshot a clean system.
I do it for @ and @home subvolumes, but since we do not have any users yet, @ will suffice.
I put manually created snapshots via btrfs into /.snapshost/manual-snaps/ directory
The /.snapshots direcory will be used for auto snapshots done by snapper which we will configure shortly.

```sh
mkdir /.snapshots/manual-snaps
root@hppl:~# btrfs subvolume snapshot -r / /.snapshots/manual-snaps/manual-root-$(date +%Y-%m-%d)-clean_install
ls -lah /.snapshots/manual-snaps/
```

______________________
Step 2 - Snapper and its configuration

The very next thing I like to do is to install and configure Snapper. Snapper will make automatic snapshots of your subvolumes - not all, only those you define. It also has a neat feature to make snapshots right before and right after you run apt install command. That's why I install and configure it before installing any additional stuff via apt.

So first thing is to install snapper.
```sh
apt install snapper
```
Next we need to do few adjustments...
Courtesy of medium website:
"The default way that snapper works is to automatically create a new subvolume “.snapshots” under the path of the subvolume that we are creating a snapshot. Because we want to keep our snapshots separated from the backed up subvolume itself we must remove the snapper created “.snapshot” subvolume and then re-mount using the one that we created before in a separate subvolume at @snapshots.
We can do this by moving to the root directory, unmount the .snapshots directory and then remove it:"
```sh
cd /
umount .snapshots
rm -r .snapshots
```
Do not worry. Despite the above, our manual snapshot we took earlier is still saved in @snapshots subvolume. Next we create a configuration for snapper for the / directory. That will also create .snapshot directory under / into which we will subsequently mount our @snapshots subvolume.
```sh
snapper -c root create-config /
```
Remove Snapper's automatically created subvolume:
```sh
btrfs subvolume delete /.snapshots
```
Recreate the /.snapshots directory and remount our @snapshots subvolume into it:
```sh
mkdir /.snapshots
mount -av
```
If you now look inside the /.snapshots directory you will see our manual snapshots directory and inside of it the manual snapshot we have taken earlier.
Although this was a little cumbersome, now we have our setup as we want it and snapper is also happy.

Let's do some modificatioins to how snapper works. All of these are optional and you may adjust them to your liking.
1) No snapshots on boot. I do not want snapper to create a snapshot each time I boot, so I disable that timer.
```sh
systemctl disable snapper-boot.timer
```
2) Better apt snapshots.
By default, snapper creates snapshots before and after running apt command. However, in the listings you only see "pre" and "post", but don't actually have a reference to which command was executed. A modification exists which will include the apt command description.
```sh
# change to /root directory
cd
# download the script file and make it executable
wget -L https://gist.github.com/imthenachoman/f722f6d08dfb404fed2a3b2d83263118/raw/dpkg-pre-post-snapper.sh
chmod +x ./dpkg-pre-post-snapper.sh
# modify the configuration file for auto apt snapshots to refer to our downloaded script
nano /etc/apt/apt.conf.d/80snapper
```
In this file, comment out the original lines (or delete them if you wish), and insert two new lines as below.
*Note: if you saved the script elsewhere, not in /root, modify the path accordingly.*
```sh
# https://bugs.debian.org/cgi-bin/bugreport.cgi?bug=770938
#  DPkg::Pre-Invoke  { "if [ -e /etc/default/snapper ]; then . /etc/default/snapper; fi; if [ -x ...
#  DPkg::Post-Invoke { "if [ -e /etc/default/snapper ]; then . /etc/default/snapper; fi; if [ -x ...
DPkg::Pre-Invoke { "/root/dpkg-pre-post-snapper.sh pre"; };
DPkg::Post-Invoke { "/root/dpkg-pre-post-snapper.sh post"; };
```

Adjustments which follow now apply only to configuration for snapshots for / , thus "-c root". If you automate snapshots of another subvolume, e.g. @home, then you must create new snapper configuration for that, named for example "home" and define its parameters in similar manner as below, but using "-c home" in the commands.

3) Allow syncing of ACLs:
```sh
snapper -c root set-config 'SYNC_ACL=yes'
```
4) Define limits for maximum of snapshots (used by cleanup algorithms)
```sh
snapper -c root set-config "NUMBER_LIMIT=10"
snapper -c root set-config "NUMBER_LIMIT_IMPORTANT=10"
```
5) Edit how many different snapshots for each period you want:
```sh
nano /etc/snapper/configs/root
```
Find and adjust this section to your liking (this is my configuration)
```sh
# limits for timeline cleanup
TIMELINE_MIN_AGE="1800"
TIMELINE_LIMIT_HOURLY="2"
TIMELINE_LIMIT_DAILY="1"
TIMELINE_LIMIT_WEEKLY="2"
TIMELINE_LIMIT_MONTHLY="1"
TIMELINE_LIMIT_YEARLY="0"
```

We can also create snapshot with snapper also manually, not just via timers:
```sh
snapper -c root create --description "default fresh install"
```

And when you list your snapper snapshots, you should see something like this:
```sh
snapper -c root list

  # | Type   | Pre # | Date                            | User | Cleanup | Description           | Userdata
----+--------+-------+---------------------------------+------+---------+-----------------------+---------
 0  | single |       |                                 | root |         | current               |
1  | single |       | Thu 06 Mar 2025 08:52:05 AM CET | root |         | default fresh install |
```

Later when you run the same command, you will see the timeline snapshots appearing as well. Cleanup algorithm runs once a day, which for me is ok, but it can be adjusted if you want.

Finally, when you want to automate snapshots for another subvolume, the steps to take are:
1) create config
snapper -c CONF_NAME create-config PATH_WHERE_SUBVOLUME_IS
*e.g. snapper -c home create-config /home/*
2) Adjust config for home
either via snapper -c NAME set-config 'CONFIGURATION_LINE'
or by direct edits: nano /etc/snapper/configs/NAME
3) Manual snapshots via snapper can be made:
snapper -c NAME create --description "Your meaningful description :) "

______________________
Step 3 - Grub autoconfiguration via grub-btrfs

This step is even more optional then those before.
What we will do here is to monitor our /.snapshots directory for new snapshots, and each time there is a new snapshot, a grub menu entry will be created, which will allow you to boot directly into that snapshot. Hopefully you will never need this + you can modify grub also manually to boot into a snapshot if such situation arises. But on the other hand, why not to save ourselves some time and some googling, and have everything done automatically and ready.

For this to work we will:
3.3.1) Need to install inotify-tools
option 3.3.2a) Either build and install grub-btrfs (for that you'll need to also install git and make), or
option 3.3.2b) Download pre-made deb package from openSUSE project, prepared for Debian (I placed it into my github repository, but feel free to find and use openSuse one here: https://software.opensuse.org//download.html?project=home%3Amkrwc%3Adebian&package=grub-btrfs)
*Note: step #2 is required, because for some reason there is no grub-btrfs package in debian repositories. Fortunately, the build itself is very easy, just clone git and run make install. However, I did not want to install anything unnecessary into proxmox installation, so I build myself a deb package on another machine, which you can download and install.*

3.3.1)
```sh
apt install inotify-tools
```

Next do only 1 of the two options!

Option 3.3.2a)
```sh
apt install git, make
cd
git clone https://github.com/Antynea/grub-btrfs.git
cd grub-btrfs
make install
cd
rm -r grub-btrfs
```

Option 3.3.2b)
```sh
cd
# Download and install the ready made package
wget -L https://github.com/BrandonSk/home-media-docker/raw/refs/heads/master/grub-btrfs-deb/grub-btrfs_4.12-gsd1_all.deb
apt install ./grub-btrfs_4.12-gsd1_all.deb
```
Step 3)
Run the daemon to check if there are no errors (if not, nothing is displayed)
```sh
systemctl start grub-btrfsd
```
...and make it start on system start:
```sh
systemctl enable grub-btrfsd
```

And voila... We are all done.
I hope you enjoyed this guide and the system works for you. Now you can proceed with remaining guides on setting up the ultimate home media server.


______________________________________
Extra info - converting btrfs RAID0 to Single

In case you only have 1 SSD or HDD on which you are installing proxmox, you have no other alternative if you choose btrfs than making it RAID0.
But for a single device, the "single" mode should be used.
The command below will use profile 'single' for data, and profile dup for metadata.
This command should be run in Stage 2, after Step 1 and before Step 2:
```sh
btrfs balance start -dconvert=single -mconvert=dup /mnt
```
