# Building a CyberLab (Oops all BASH)

What is a CyberLab? Dunno. Let's build one anyway.

We'll start by making a directory where we can 'neatly' keep all our lab stuff (virtual machines, CTF write-ups, scripts, whatever- we'll worry about organization once we've got stuff to organize). I'll call it `CyberLab` because it sounds cool, and I'll slap it right in my home directory for convenience.

```bash
stone@stone:~$ mkdir CyberLab
```

No output is a good sign for most of these commands, but I'm not the trusting type so I'm just gonna make sure.

```bash
stone@stone:~$ ls
CyberLab  Desktop  Documents  Downloads  Music  Pictures  Public  Templates  Videos
```

Lovely. We have a directory named `CyberLab` and we can put all our cyber stuff in there.

If we wanted, we could pretty much stop here. We've created a directory to house our labbing stuff, separate from our other files, and we did so using just the command-line interface (CLI). Technically, the objective has been met, but it doesn't feel very wizard. So let's see how else we can spice things up.

I'm pretty sure I've got an unused drive on this machine. Let's take a look.

```bash
stone@stone:~$ lsblk -o NAME,SIZE,FSTYPE,LABEL,MOUNTPOINTS
NAME          SIZE FSTYPE LABEL MOUNTPOINTS
nvme1n1     953.9G              
├─nvme1n1p1   976M vfat         /boot/efi
├─nvme1n1p2 921.1G ext4         /
└─nvme1n1p3  31.8G swap         [SWAP]
nvme0n1     465.8G   
```

There it is. The drive `nvme0n1` has no partitions or recognized filesystems. Just 465.8G of fresh, fertile soil to build my lab on. But despite our good fortune, we can't forget our due diligence. Let's make extra sure nothing already lives here before we start laying bricks. The `lsblk` command gives us a good overview of our block devices, but it doesn't necessarily tell us everything about the signatures that might be lingering on a disk. For that, we can use `wipefs`.

In case you didn't catch it, 'wipefs' stands for "wipe filesystem"- and while it won't *actually* wipe out your filesystem, it WILL happily obliterate the signatures that identify it if you're not careful. With that in mind, we'll run it first with the `-n` (no-act) flag so that it simply scans the drive for recognizable filesystem, RAID, and partition-table signatures without actually modifying anything.

```bash
stone@stone:~$ sudo wipefs -n /dev/nvme0n1
```

Ideally, this will output nothing, indicating that the disk is at least uninhabited enough for our purposes. If you do end up getting any kind of output, but are POSITIVE that nothing there is anything you want to keep, then run the same 'wipefs' command, but replace the -n flag with -a to put it in 'attack' mode (not what -a actually stands for) and let it loose. Afterwards, run it with just the -n flag again to confirm no output so we can get to the good stuff.

Next, we'll be using the `parted` command to create a GUID Partition Table (GPT) and a single partition named `CyberLab` that spans *nearly* the entire disk. Note, however, that I'll actually have the partition start at 1 MiB rather than 0. This provides proper partition alignment, which is particularly important for modern SSDs and NVMe drives. Also remember that these are extremely destructive commands. We're casting big boy spells here, not cantrips- you swim at your own risk.

```bash
stone@stone:~$ sudo parted /dev/nvme0n1 mklabel gpt
stone@stone:~$ sudo parted /dev/nvme0n1 mkpart CyberLab 1MiB 100%         
```

These commands -despite their awesome and formidable destructive power- will almost certainly whine about updating `/etc/fstab`. That's okay, we'll take care of it later. For now, let's run `lsblk` again to verify our work.

```bash
stone@stone:~$ lsblk -o NAME,SIZE,FSTYPE,LABEL,MOUNTPOINTS
NAME          SIZE FSTYPE LABEL MOUNTPOINTS
nvme1n1     953.9G              
├─nvme1n1p1   976M vfat         /boot/efi
├─nvme1n1p2 921.1G ext4         /
└─nvme1n1p3  31.8G swap         [SWAP]
nvme0n1     465.8G              
└─nvme0n1p1 465.8G   
```

Glorious. Our enchantments have taken hold of the disk and now we've got ourselves a partition. So let's give it a filesystem. I've decided to use ext4 because it's my project and I can do what I want. I'm also going to give the filesystem a `CyberLab` label just like I did with the partition. It's not super-duper necessary, but I like the consistency.

```bash
stone@stone:~$ sudo mkfs.ext4 -L CyberLab /dev/nvme0n1p1
mke2fs 1.47.4 (6-Mar-2025)
Discarding device blocks: done                            
Creating filesystem with 122096384 4k blocks and 30531584 inodes
Filesystem UUID: 97c8199f-2a92-4a6f-a06e-a5b476b318ab
Superblock backups stored on blocks: 
        32768, 98304, 163840, 229376, 294912, 819200, 884736, 1605632, 2654208, 
        4096000, 7962624, 11239424, 20480000, 23887872, 71663616, 78675968, 
        102400000

Allocating group tables: done                            
Writing inode tables: done                            
Creating journal (262144 blocks): done
Writing superblocks and filesystem accounting information: done  
```

That was easy. The `lsblk` command we used earlier will confirm that our new partition now has an ext4 filesystem and a fun label to make it easy to identify as awesome. Now we're ready to mount the CyberLab partition to the CyberLab directory we created earlier.

Before we do, though, there's something worth understanding about mountpoints: When we mount a filesystem to an existing directory, its contents effectively take the place of whatever was previously visible inside that directory. The original contents aren't deleted, but they'll be hidden until the filesystem is unmounted. Basically, we're replacing a door with a portal. The portal transports us to a spacious pocket dimension whenever we walk through it, but in doing so, makes the room behind the doorway inaccessible. Since the 'room' in this analogy (the CyberLab directory) is brand new and completely empty, this doesn't cause any problems for us.

And since we're touching on the theory side of things a bit already, this seems like as good a place as any to revisit my earlier comment about `/etc/fstab` (the file that our earlier 'parted' commands nagged us to update). This file tells Linux which filesystems should be mounted, where they should be mounted, and which options should be used. By adding our new partition to it, we can ensure that CyberLab is automatically mounted whenever the machine boots.

Now let's get back to the action. We'll need the new filesystem's UUID for this next part, so let's grab that. I'm going to pipe the output of the `blkid` command into `tr` to replace the spaces with newlines, just to make the output nicer to look at. You don't have to.

```bash
stone@stone:~$ sudo blkid /dev/nvme0n1p1 | tr " " "\n"
/dev/nvme0n1p1:
LABEL="CyberLab"
UUID="97c8199f-2a92-4a6f-a06e-a5b476b318ab"
BLOCK_SIZE="4096"
TYPE="ext4"
PARTLABEL="CyberLab"
PARTUUID="8c6982aa-8988-43ae-87f1-5a392565b925"
```

Fantastic. Now that we've got the UUID, we can add a new entry to the `/etc/fstab` file using any text editor. I'm going to use nano for this because I don't hate myself enough to use vim and I don't trust myself enough to append it with echo.

```bash
stone@stone:~$ sudo nano /etc/fstab
```

Now I'll add this line to the end of the file:

```fstab
UUID=97c8199f-2a92-4a6f-a06e-a5b476b318ab /home/stone/CyberLab ext4 defaults,nofail 0 2
```

Here's a quick breakdown of what we're looking at:
`UUID=...`		-->	Identifies the filesystem we want to mount.
`/home/stone/CyberLab`	-->	The directory where we want the filesystem mounted.
`ext4`			-->	The filesystem type.
`defaults,nofail`	-->	Uses standard mounting behavior. Allows the system to continue booting if the drive isn't available.
`0`			-->	Disables the legacy dump utility. (I literally don't know what this even is)
`2`			-->	Tells the filesystem checker to check this partition *after* the root filesystem.


After I save and exit, I'm going to run the command mentioned in the file comments to ask systemd to reload its configuration. Since systemd uses `/etc/fstab` to generate mount units, this ensures it recognizes our changes.

```bash
stone@stone:~$ sudo systemctl daemon-reload
```

Before we attempt to mount anything (I promise we'll get there), let's make sure our new `/etc/fstab` entry is valid.

```bash
stone@stone:~$ sudo findmnt --verify --verbose
...
/home/stone/CyberLab
   [ ] target exists
   [ ] userspace options: nofail
   [ ] UUID=97c8199f-2a92-4a6f-a06e-a5b476b318ab translated to /dev/nvme0n1p1
   [ ] source /dev/nvme0n1p1 exists
   [ ] FS type is ext4
...
```
Everything here looks good. We can now use the `mount` command with the `-a` flag to mount all the filesystems specified in `/etc/fstab`, including the new one we just added. This is essentially what will happen automatically whenever the machine boots in the future, so we should only need to do this once.

```bash
stone@stone:~$ sudo mount -a
```

Let's run our trusty `lsblk` command again to make sure everything still looks like we expect.

```bash
stone@stone:~$ lsblk -o NAME,SIZE,FSTYPE,LABEL,MOUNTPOINTS
NAME          SIZE FSTYPE LABEL    MOUNTPOINTS
nvme1n1     953.9G                 
├─nvme1n1p1   976M vfat            /boot/efi
├─nvme1n1p2 921.1G ext4            /
└─nvme1n1p3  31.8G swap            [SWAP]
nvme0n1     465.8G                 
└─nvme0n1p1 465.8G ext4   CyberLab /home/stone/CyberLab
```

Huzzah. Our previously jobless `nvme0n1` drive now has a juicy 465.8G partition with an ext4 filesystem and has been mounted to the CyberLab directory we created in the very first step. We've essentially moved our CyberLab outside of a shared office space and onto its own patch of land. We should probably make sure we've given ourselves permission to be there.

```bash
stone@stone:~$ sudo chown stone:stone /home/stone/CyberLab/
```

And there we have it! We've successfully created a dedicated storage location for our lab stuff. It's formatted, mounted, and configured to persist across reboots.

