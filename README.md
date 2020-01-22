# Containers

## Setup

### Preqs

* Ensure you have `node --version >= 12.14`.
* Ensure you have `bakerx --version >= 0.6.3`.
* Ensure you have VirtualBox installed.

## Creating a simple container with chroot

### Create a headless micro VM with bakerx

Pull an 3.9 alpine image and create a micro VM called, `con0`.

```bash
bakerx pull ottomatica/slim#images alpine3.9-simple

bakerx run con0 alpine3.9-simple
```

### Prepare a simple rootfs with busybox.

```bash
mkdir -p rootfs/bin rootfs/sbin rootfs/usr/bin rootfs/usr/sbin

wget https://www.busybox.net/downloads/binaries/1.31.0-defconfig-multiarch-musl/busybox-i686 -O rootfs/bin/busybox
chmod +x rootfs/bin/busybox
```

Install symlinks inside the rootfs
```bash
chroot rootfs /bin/busybox --install -s
```

### Playing with container

Let's try it out!

```bash
PS1="C-$ " chroot rootfs /bin/busybox sh
C-$ touch HELLO.txt
C-$ ls
C-$ exit
```

There is a minor problem. We have not perserved the isolation property for our filesystem, as the filesystem is still mutable.

You can see new file exists in the file system, and worse, we can delete and mess up things.
```bash
PS1="C-$ " chroot rootfs /bin/busybox sh
ls
rm HELLO.txt
```

### Introducing overlay

Create a new [container.sh](container.sh) file inside your micro VM and make it exectuable.

```bash
#!/bin/ash

# Expect rootfs as script argument
ROOTFS=$1

# Create a random unique string
nonce=$(</dev/urandom tr -dc A-Za-z0-9-_ | head -c 10)

# Prepare our filesystem for our container.
mkdir -p /tmp/$nonce/upper /tmp/$nonce/workdir /tmp/$nonce/overlay
# Create an overlay filesystem based on our read-only root filesystem.
mount -t overlay -o lowerdir=$ROOTFS,upperdir=/tmp/$nonce/upper,workdir=/tmp/$nonce/workdir none /tmp/$nonce/overlay

# Create symlinks in our container for convience.
chroot /tmp/$nonce/overlay/ /bin/busybox --install -s

# Launch container with custom prompt in ash shell.
PS1="$nonce-# " chroot /tmp/$nonce/overlay /bin/busybox sh
```

Using the overlay filesystem, we can keep our rootfs "read-only", while allowing new changes to be made. The read-only portion is denotated by the "lower" directory. The change states are maintained in the "upper" and "work" directories, and the merged/unified filesystem is available in the "overlay" directory.

This script will create a new snapshot of the filesystem everytime is it launched in order to run the process.

A short demo of the script illustrates our ability to perserve the rootfs each time we create a new container.

```
nanobox:~# ./container.sh rootfs
TWIYg-MJOP-# ls
bin      linuxrc  sbin     usr
TWIYg-MJOP-# touch FOO.txt
TWIYg-MJOP-# ls
FOO.txt  bin      linuxrc  sbin     usr
TWIYg-MJOP-# exit
nanobox:~# ./container.sh rootfs
ZC4RCytZpg-# ls
bin      linuxrc  sbin     usr
```







