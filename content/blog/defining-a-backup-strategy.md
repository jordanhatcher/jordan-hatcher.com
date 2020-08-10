+++
title = "World Backup Day: Developing a Personal Backup Strategy"
date = 2019-03-31T11:55:05-04:00
draft = false
+++

Lately I have been thinking about setting up proper backups for my data. Like most people probably do, I have several
devices containing important data, such as a laptop, desktop computer, and a phone. In pure coincidence, today happens
to be "World Backup Day", so I decided to take some time to sort out a good backup strategy for all of my data.

## Planning the Backup Strategy
I figured that the most likely scenarios where I would need a backup to recover my data are:

* A fire, causing home devices to be destroyed
* A ransomware attack, encrypting all data on the home network
* Theft of a device containing sensitive data

I had heard of the ["3-2-1 rule" of backups](https://www.backblaze.com/blog/the-3-2-1-backup-strategy/), where there
should ideally be:

* 3 copies of important data
* 2 types of media
* 1 off-site backup

The 3-2-1 rule seems to cover all of my threat scenarios nicely. Even if a fire takes out my house, this backup strategy
would keep my data nice and safe to restore from later.

## Implementation

### Syncing Data
Since my data was spread out across my various devices, I first wanted a way to collect my data centrally to avoid doing a
backup on each device separately. I decided to set up [nextcloud](https://nextcloud.com/) on my home server, since it is
Free and Open Source Software, and has clients for both Linux and Android. Syncing my devices gives me two useful features:

1. I can take snapshots of the nextcloud data directory to instantly get a backup of all my important data
2. Local copies of data are distributed across some of my devices, to provide additional redundancy

It should be noted that #2 does NOT count towards the "3 copies" in the 3-2-1 rule, since deleting or corrupting a file on
one device would have that change be propagated to the other devices that are in sync. However if the hard drive in my
laptop were to suddenly die, I would likely be able to just plop in a new drive and sync it, so I feel that there is still
some benefit in having the synchronization layer there.

### Snapshots
I had used [Btrfs](https://btrfs.wiki.kernel.org/index.php/Main_Page) for the filesystem on my home server when I was
first setting it up. I mainly chose it because it seemed to have many of the features that ZFS is known for (copy-on-write,
snapshots, drive pooling, checksums), but was supported out of the box when installing Fedora (my Linux distribution of
choice). As it turns out, Brtfs snapshots are a great way to perform backups! After ensuring that the directory to back up
is a Btrfs subvolume, and that the backup drive (ideally an external drive) is formatted as Btrfs:

```bash
btrfs subvolume snapshot -r /var/nextcloud/data /snapshots/nexcloud_data_2019-03-31 # initial snapshot
btrfs send /backups/nextcloud_data_2019-03-31 | btrfs receive /mnt/drive/backups # send snapshot to the backup drive

# the next day...
btrfs subvolume snapshot -r /var/nextcloud/data /snapshots/nexcloud_data_2019-04-01 # snapshot for the current day
btrfs send -p /snapshots/nexcloud_data_2019-03-31 /snapshots/nexcloud_data_2019-04-01 | # send incremental diff to backup drive
brtfs receive /mnt/drive/backups
```

The snapshots could easily be created in a daily cron job, and old snapshots can simply be deleted using:
```bash
btrfs subvolume delete /snapshots/nextcloud_data_2019-03-31
```

Also, the subvolumes snapshots are all created as read-only, so that may provide some resistance to basic ransomware
attacks. The backup drive I used to receive the snapshots is also encrypted with LUKS, and is now stored elsewhere to
function as my off-site backup.

### Disk Backups
I know disks are becoming a thing of the past, but they can still hold plenty of pictures, videos, and documents. I don't
actually have a ton of data to back up, so burning DVDs is a cheap extra backup, and fulfils the requirement for a second
type of media. The important data that I do have, like family photos, is fairly static, so burning it to a read-only disk
is perfectly fine.

## The End?
Between my data being replicated across my devices, backups of Btrfs snapshots, and disk backups, I think I now have a
solid backup plan. A good backup strategy means nothing if it doesn't work though, so I plan to regularly test restoring
the data on my snapshots, as well as making sure to perform backups frequently.