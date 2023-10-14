---
title: "Build a bootable qcow2 image with debootstrap"
date: 2023-10-12T00:03:10+08:00
draft: false
---

# Create Image

## Create a empty image

```bash
qemu-img create -f qcow2 debian-buster.qcow2 60G
```

This command creates a `debian-buster.qcow2` image file in the qcow2 format, with a maximum storage capacity of 60GB (
which does not actually consume 60GB of disk space).

💡 Most of the commands listed below require root privileges.

## Mount the image as a device

To begin, load the NBD (Network Block Device) kernel module.

```bash
modprobe nbd
```

Mount the empty qcow2 image onto the NBD (Network Block Device) device.

```bash
qemu-nbd -c /dev/nbd0 debian-buster.qcow2
```

We will attempt to mount the image using `/dev/nbd0`, but if this device is unavailable or being used by another
process,
we will try another device such as `/dev/nbd1` until we find an available one.

Once mounted, you can treat the qcow2 image as a hard drive device, specifically `/dev/nb`. It is an empty hard drive.
The next step is to partition the hard drive and install a Linux filesystem on it.

## Partition

You have the flexibility to choose any tool you prefer for partitioning the hard drive. In this tutorial, we will use
the `fdisk` tool.

```bash
fdisk /dev/nbd0
```

In the interactive mode, please enter the following commands:

```bash
Command (m for help): o

Created a new DOS disklabel with disk identifier 0x3267d9af.

Command (m for help): n
Partition type
   p   primary (0 primary, 0 extended, 4 free)
   e   extended (container for logical partitions)
Select (default p): p
Partition number (1-4, default 1): 1
First sector (2048-125829119, default 2048):
Last sector, +sectors or +size{K,M,G,T,P} (2048-125829119, default 125829119):

Created a new partition 1 of type 'Linux' and of size 60 GiB.

Command (m for help): a
Selected partition 1
The bootable flag on partition 1 is enabled now.

Command (m for help): w
The partition table has been altered.
Calling ioctl() to re-read partition table.
Syncing disks.
```

o: create a new empty DOS partition table

n: add a new partition

a: toggle a bootable flag

w: write table to disk and exit

Or you can use single shell command to do this.

```bash
echo -e "o\nn\np\n1\n\n\na\nw\n" | fdisk /dev/nbd0
```

Use `fdisk -l /dev/nbd0` to view the partition result.

```bash
Disk /dev/nbd0: 60 GiB, 64424509440 bytes, 125829120 sectors
Units: sectors of 1 * 512 = 512 bytes
Sector size (logical/physical): 512 bytes / 512 bytes
I/O size (minimum/optimal): 512 bytes / 512 bytes
Disklabel type: dos
Disk identifier: 0xfd48ea90

Device      Boot Start       End   Sectors Size Id Type
/dev/nbd0p1 *     2048 125829119 125827072  60G 83 Linux
```

## Make filesystem

Format the new created partition with ext4 filesystem.

```bash
mkfs.ext4 /dev/nbd0p1
```

Output

```bash
mke2fs 1.43.4 (31-Jan-2017)
Discarding device blocks: failed - Input/output error
Creating filesystem with 15728384 4k blocks and 3932160 inodes
Filesystem UUID: f5bfdfcd-2074-498a-b4c5-42a9a29b30be
Superblock backups stored on blocks:
	32768, 98304, 163840, 229376, 294912, 819200, 884736, 1605632, 2654208,
	4096000, 7962624, 11239424

Allocating group tables: done
Writing inode tables: done
Creating journal (65536 blocks): done
Writing superblocks and filesystem accounting information: done
```

## Mount partition

```bash
mkdir mnt
mount /dev/nbd0p1 $PWD/mnt/
```

## Use debootstrap to build debian(buster) root filesystem

```bash
apt install -y debootstrap
debootstrap buster $PWD/mnt https://mirrors.tuna.tsinghua.edu.cn/debian/
```

I uses a mirror(`https://mirrors.tuna.tsinghua.edu.cn/debian/`) to download debian software. You can omit this if you
don’t need.

## Mount proc and device

```bash
mount -o bind /dev "$PWD/mnt/dev"
mount -o bind /dev/pts "$PWD/mnt/dev/pts"
mount -t proc none "${PWD}/mnt/proc"
mount -t sysfs none "${PWD}/mnt/sys"
```

You can bind more directory if you need such as `/sys/fs/cgroup`

## Chroot

```bash
chroot $PWD/mnt bash
```

## Configure apt mirror(optional)

```bash
cat > /etc/apt/sources.list << EOF
deb https://mirrors.tuna.tsinghua.edu.cn/debian/ buster main contrib non-free

deb https://mirrors.tuna.tsinghua.edu.cn/debian/ buster-updates main contrib non-free

deb https://mirrors.tuna.tsinghua.edu.cn/debian/ buster-backports main contrib non-free

deb https://mirrors.tuna.tsinghua.edu.cn/debian-security buster/updates main contrib non-free
EOF
```

Configure apt mirror to make `apt install` faster.

## Install kernel

Debootstrap is a tool used to create a Debian root filesystem without including the kernel. After using debootstrap, you
will need to manually install the kernel using the apt package manager.

```bash
apt update
apt install -y linux-image-amd64
```

You can install other version’s kernel by apt. Use `apt search linux-image` to search more kernel image.

## configure fstab

Get UUID of hard drive.

```bash
UUID=$(blkid |grep '^/dev/nbd0p1'|grep -o ' UUID="[^"]\+"'|sed -e 's/^ //')
echo $UUID
```

Output(UUID might be different on your disk)

```bash
UUID="f5bfdfcd-2074-498a-b4c5-42a9a29b30be"
```

Write to fstab

```bash
cat >/etc/fstab <<EOF
${UUID} / ext4 defaults 0 1
EOF
```

## Install bootloader(grub)

```bash
apt install -y grub2
```

## Configure grub(optional)

```bash
# Print grub log in tty when boot.
sed -i 's/GRUB_CMDLINE_LINUX=\"/GRUB_CMDLINE_LINUX=\"console=tty0 console=ttyS0,115200 /g' /etc/default/grub
sed -i 's/GRUB_TIMEOUT=.*/GRUB_TIMEOUT=0/g' /etc/default/grub
# Disable graphic reqirement.
sed -i 's/^#GRUB_TERMINAL=console$/GRUB_TERMINAL=console/' /etc/default/grub
```

Uncomment `GRUB_TERMINAL=console` to disable graphic requirement. Or you must pass video device in qemu (
such `-device VGA`)

If you want to make boot faster, you can disable more feature in `GRUB_CMDLINE_LINUX` such
as `GRUB_CMDLINE_LINUX="console=tty0 console=ttyS0,115200 noibrs noibpb nopti nospectre_v2 nospectre_v1 l1tf=off nospec_store_bypass_disable no_stf_barrier mds=off tsx=on tsx_async_abort=off mitigations=off"`

## Install grub onto for disk.

```bash
# Remove os-prober config, don't need to detect other os on the machine. Or the host boot config will write to grub config, it's not useful.
rm -f /etc/grub.d/*os-prober
grub-install --no-floppy "/dev/nbd0"
update-grub
```

## Another useful command

In this environment, it’s better to install necessary software or config such as

- ssh server and it’s config;
- cloud-init and it’s config;
- root password;
- hostname, timezome;
- etc.

```bash
# Change root password.
passwd root

```

## Unmount and clean

```bash
# exit from chroot
exit
rm -rf "${PWD}/mnt/tmp/*"
umount "$PWD/mnt/dev/pts"
umount "$PWD/mnt/dev"
umount "${PWD}/mnt/proc"
umount "${PWD}/mnt/sys"
fstrim -v "$PWD/mnt"
umount "$PWD/mnt"
e2fsck -p -f "/dev/nbd0p1"
qemu-nbd --disconnect "/dev/nbd0"

```

## Output final image

# Test the image

## Start vm

```bash
qemu-system-x86_64 --enable-kvm --nodefaults --nographic -display none -machine type=pc,usb=off -smp 6,sockets=1,cores=6,threads=1 -m 2024M -device virtio-balloon-pci,id=balloon0 -device virtio-blk-pci,drive=drive0,serial=d547d26900a8e36b0198 -drive file=debian-buster.qcow2,format=qcow2,id=drive0,if=none,aio=threads,media=disk,cache=unsafe,snapshot=on -chardev socket,id=serial0,path=/tmp/console.sock,server=on,wait=off -serial chardev:serial0 -vnc unix:/tmp/vnc.sock -device VGA,id=vga,vgamem_mb=64

```

## Connect VM by console

```bash
apt install -y minicom
minicom -D unix#/tmp/console.sock
```