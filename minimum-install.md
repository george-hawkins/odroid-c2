Minimal Odroid image
====================

The normal Odroid image is Ubuntu MATE with a whole load of extras (Kodi etc.) installed.

On Aug 4th, 2016 Hardkernel [announced](https://plus.google.com/102407583165771869503/posts/XSnZxZxNz2A) minimal images.

Before intalling such an image to an eMMC module (or Micro SD card) let's first...

Determine the correct device
----------------------------

Before you insert the eMMC module that you intend to write to (using a USB card reader or whatever) do:

    $ ls /dev/disk/by-id

Now insert the USD card reader with eMMC module and do the same again:

    $ ls /dev/disk/by-id

You should see three new files corresponding to the new device and its two partitions (the same filename as the device but with the suffixes `-part1` and `-part2`).

The name of the file corresponding to the device depends on the make and model of reader.

In my case the name was `usb-Generic_STORAGE_DEVICE_000000000251-0:0`. Now remember that name for use in subsequent commands:

    $ DISK='/dev/disk/by-id/usb-Generic_STORAGE_DEVICE_000000000251-0:0'

If you do `ls -l` on this file you'll see it's a softlink back to something fairly standard like `/dev/sdb`.

---

If you eject removable devices via the file manager this will also power off the device - which we don't want.

We can eject our two partitions without powering off the main device like so:

    $ udisksctl unmount --block-device "$DISK-part1"
    $ udisksctl unmount --block-device "$DISK-part2"

As seen USB drives etc. are automatically mounted by default. Using `udisksctl` one can unmount them without powering them off.

But simply mounting a drive alters its content (I would guess the mount time is maybe written somewhere) and this is enough to ruin part of the verification process we'll want to do as part of the following install.

So temporarily disable this:

    $ gsettings set org.gnome.desktop.media-handling automount false

Installing the image
--------------------

I downloaded the minimal image via the closest mirror listed at <http://odroid.com/dokuwiki/doku.php?id=en:c2_release_linux_ubuntu> and unpacked it:

    $ mv ~/Downloads/ubuntu64-16.04-minimal-odroid-c2-20160815.img.xz .
    $ unxz ubuntu64-16.04-minimal-odroid-c2-20160815.img.xz

    $ IMAGE=$PWD/ubuntu64-16.04-minimal-odroid-c2-20160815.img

Then wrote it to the eMMC module and safely ejected it:

    $ sudo dd if=$IMAGE of=$DISK bs=1M conv=fsync
    $ udisksctl power-off --block-device $DISK

`udisksctl` will make sure that everything is synced to the drive before powering it off. Now the step which would fail if automount were still enabled...

Unplug and reinsert the USB device - nothing should automount. Now generate an md5 sum for the contents of the disk:

    $ sudo dd if=/dev/sdb bs=512 count=$(($(stat -c%s $IMAGE)/512)) | md5sum

And verify this against the downloaded image:

    $ dd if=$IMAGE bs=512 count=$(($(stat -c%s $IMAGE)/512)) | md5sum

The values shown should match. Now reenable automounting:

    $ gsettings set org.gnome.desktop.media-handling automount true

Logging in
----------

See instructions on how to find the IP address of a new Odroid elsewhere here. Once you've found the IP address log in like so:

    $ ssh root@192.168.1.206

Note that unlike a normal Odroid C2 Ubuntu install (where the username would be `odroid`) you have to log in as `root` (the password is still `odroid`).

    # df -h
    Filesystem      Size  Used Avail Use% Mounted on
    udev            736M     0  736M   0% /dev
    tmpfs           172M  7.8M  165M   5% /run
    /dev/mmcblk0p2   15G  830M   13G   6% /
    tmpfs           860M     0  860M   0% /dev/shm
    tmpfs           5.0M     0  5.0M   0% /run/lock
    tmpfs           860M     0  860M   0% /sys/fs/cgroup
    /dev/mmcblk0p1  128M   15M  114M  12% /media/boot

---

Setup walkthru from start to finish
-----------------------------------

After going through the above process a couple of times, and adding some additional steps to resolve issues that occurred while trying to upgrade the image, the following is a summary of doing a full installation and upgrade from start to finish.

Most of the steps have already been explained above - and the few additional steps are explained after the walk thru here.

---

First mount the eMMC module (or MicroSD card) and setup the disk:

    $ IMAGE=$PWD/ubuntu64-16.04-minimal-odroid-c2-20160815.img
    $ DISK='/dev/disk/by-id/usb-Generic_STORAGE_DEVICE_000000000251-0:0'
    $ sudo dd if=$IMAGE of=$DISK bs=1M conv=fsync
    $ udisksctl mount --block-device "$DISK-part2"
    $ cd /media/$USER/rootfs
    $ sudo mv etc/apt/apt.conf.d/20auto-upgrades root
    $ sudo rm boot/initrd.img-4.4.0-31-generic boot/uInitrd-4.4.0-31-generic var/lib/initramfs-tools/4.4.0-31-generic
    $ cd
    $ udisksctl unmount --block-device "$DISK-part2"
    $ udisksctl power-off --block-device $DISK

For an explanation of the `mv` and `rm` steps see later.

Attach the eMMC module to the Odroid C2, connect it to the network and power it up. Then find it:

    $ nmap -T4 -F 192.168.1.133/24
    $ ssh root@192.168.1.240

First create an `odroid` user (for an explanation see later):

    # useradd odroid --create-home --shell /bin/bash
    # echo odroid:odroid | chpasswd
    # usermod --append --groups adm,disk,lp,dialout,fax,voice,cdrom,floppy,sudo,audio,operator,src,video,plugdev,staff,users,input odroid

Note: `useradd` has a `--password` option but this expects an already encrypted password (see this [Ask Ubuntu answer](http://askubuntu.com/a/668134) and this [UNIX Stack Exchange answer](http://unix.stackexchange.com/a/81248/111626)).

Unlike the non-minimum distribution backports is already installed - you can confirm like so:

    # apt-cache policy | fgrep backports

So let's get straight on with getting everything up-to-date:

    # apt-get update
    # apt-get dist-upgrade

    # ls -l /var/run/reboot-required

The `reboot-required` file exists (but unlike similar situation with the non-minimal image it contains nothing).

Let's check the current kernel version and then reboot:

    # uname -r
    3.14.65-73
    # reboot now

After logging in again after the reboot:

    # uname -r
    3.14.79-105

So the kernel was upgraded. Now let's purge any old kernels:

    # dpkg --list | grep linux-image
    # dpkg --list | grep linux-headers
    # apt-get purge --yes linux-image-c2 linux-image-3.14.65-73

And do a final cleanup:

    # sudo apt-get autoremove
    # sudo apt-get clean

Unlike the non-minimal image there was no complaint during the `clean` step about `apt-fast` as `apt-fast` isn't installed.

After all this `df` shows that there's 22MB less available on `/`.

If you want you can restore the apt file we moved earlier:

    # mv 20auto-upgrades /etc/apt/apt.conf.d

This will cause it to do a daily update and automatically install security updates (see link up above for more details).

---

Explanations
------------

Why did we move the `20auto-upgrades` when setting up the eMMC module?

By default this file specified that the system should do a daily update and apply security updates.

If the file is left in place then (assuming you're relatively quick to log into the newly powered up system) these actions will have been kicked off and you won't be able to do anything with `apt-get`.

Instead you'll get the following error:

    E: Could not get lock /var/lib/dpkg/lock - open (11: Resource temporarily unavailable)
    E: Unable to lock the administration directory (/var/lib/dpkg/), is another process using it?

Aside: I diagnosed this by copying an arm64 `lsof` binary onto the eMMC module when setting it up and using it like so:

    # lsof | fgrep /var/lib/dpkg

From the process ID shown I could see it was locked by `unattended-upgrade` and that the parent of this process was `apt.systemd.daily`.

With this information and a Google I got to:

* <https://github.com/boxcutter/ubuntu/issues/73>
* <https://www.hiroom2.com/2016/05/18/ubuntu-16-04-auto-apt-update-and-apt-upgrade/>

---

Next - why did we remove the `4.4.0-31-generic` related files?

They appear to be the remain of a 4.X kernel left around by mistake by Hardkernel. If you don't remove them then the `dist-upgrade` fails with:

    update-initramfs: Generating /boot/initrd.img-4.4.0-31-generic
    WARNING: missing /lib/modules/4.4.0-31-generic
    Ensure all necessary drivers are built into the linux image!
    depmod: ERROR: could not open directory /lib/modules/4.4.0-31-generic: No such file or directory
    depmod: FATAL: could not search modules: No such file or directory

If you then look at what images are installed `4.4.0-31` isn't listed as one of them:

    # dpkg --list | grep linux-image
    ii  linux-image-3.14.65-73     20160802    arm64    Linux kernel binary image for version 3.14.65-73
    ii  linux-image-3.14.79-104    20170119    arm64    Linux kernel binary image for version 3.14.79-104
    ii  linux-image-c2             73-1        arm64    Linux Kernel 3.14 Long Term for ODROID-C2

So I just removed the remains that I could find (which seems to be the suggested solution on Googling).

It's the file under `/var/lib/initramfs-tools` in particular that causes `update-initramfs` to fail.

---

Finally - why did we add the `odroid` user?

If you don't then the post-installation script of the `bootini` package fails with:

    usermod: user 'odroid' does not exist

If you look in `/var/lib/dpkg/info/bootini.postinst` you'll see on the last line that it tries to modify the user `odroid`:

    usermod -a -G sys odroid

So to stop this failing we create this user first. The various options specified for `usermod` and the groups specified with `usermod` try to match as near as possible the `odroid` user found on the non-minimal image.

Further issues
--------------

If you check for failed systemd services you'll see that ureadahead always fails:

```text
# systemctl --failed
UNIT               LOAD   ACTIVE SUB    DESCRIPTION
* ureadahead.service loaded failed failed Read required files in advance
```

If you then look at the status output you'll see more information:

```text
# systemctl status ureadahead.service
...
Feb 11 17:28:00 odroid64 ureadahead[183]: ureadahead: Error while tracing: No such file or directory
```

If you then try to start ureadahead direcly with strace, to see what file was the problem, you see:

```text
# strace /sbin/ureadahead --force-trace --debug --verbose
...
openat(3, "events/fs/do_sys_open/enable", O_RDWR) = -1 ENOENT (No such file or directory)
close(3)                                = 0
write(2, "ureadahead: Error while tracing:"..., 59ureadahead: Error while tracing: No such file or directory
) = 59
```

If you Google for this you'll find the kernel simply doesn't contain the patch necessary for ureadahead to do its work. See:

* <http://unix.stackexchange.com/questions/308801/ureadahead-error-while-tracing-possible-kernel-patch-problem>
* <https://bugs.launchpad.net/ubuntu/+source/ureadahead/+bug/677443>

This is presumably a consequence of running an up-to-date Ubuntu version on increasingly aged kernel while waiting for mainline kernel support for the Odroid C2 CPU.

See this thread for more on progress in getting Amlogic S905 support into the mainline: <http://forum.odroid.com/viewtopic.php?f=135&t=22717>

---

### Blue LED

The intensly bright blue LED on Odroid C2 board is annoying if left to pulse continually but is useful to signal that startup or shutdown has completed.

To enable it, only during these phases, install the [`blue-led.service`](blue-led.service) file that's included here.

If everything is already up and running you can install it like so:

    $ scp blue-led.service root@192.168.1.206:/etc/systemd/system
    $ ssh root@192.168.1.206
    # systemctl daemon-reload

Either way once the system is up and running you can enable the service like so:

    $ ssh root@192.168.1.206
    # systemctl enable blue-led

If you reboot the system then on subsequent startups the blue LED will go from solid to flashing to off - when it goes off the system should be ready for you to login.

Similarly when the system is _subsequently_ shutdown the blue LED will start flashing and then go off (leaving just the red LED indicating that the system still has power).

The shutdown is far quicker than the startup.

When running the non-minimal Odroid C2 image shutting down the system causes you to be logged off immediatelly - for whatever reason this doesn't seem to happen with the minimal image - and you just end up stuck.

To kill your ssh session see this [Ask Ubuntu answer](http://askubuntu.com/a/29952) - in short just press enter, then `~` (tilde) and then `.` (period).

---

### Locale

The minimal image has a very limited number of locales installed. You can list them like so:

    # locale -a

So it's probably best to tell the ssh daemon to ignore the locale sent by remote clients or you may end up with a confused locale setup on logging in via ssh.

To see what locale settings have been propagated via ssh:

    # locale

To ignore the locale from remote clients:

    # sed -i 's/AcceptEnv\s\+LANG/#&/' /etc/ssh/sshd_config
    # systemctl restart sshd

Now the locale settings will be taken from `/etc/default/locale` rather than the remote client. Log out and log back in and enter:

    # locale

Depending on how in sync the remote client and the Odroid C2 are, in terms of locale, you may or may not see a different set of locale values displayed.

### Time zone

By default the time zone is UTC for the minimal image:

    # date +%Z
    UTC

To change this to e.g. 'Europe/Zurich':

    # timedatectl set-timezone Europe/Zurich

---

Summary of additional steps
---------------------------

Assuming the Odroid system is already up and running the additional step to setup the blue LED service, locale and time zone are:

    $ scp blue-led.service root@192.168.1.206:/etc/systemd/system
    $ ssh root@192.168.1.206
    # systemctl daemon-reload
    # systemctl enable blue-led
    # timedatectl set-timezone Europe/Zurich
    # sed -i 's/AcceptEnv\s\+LANG/#&/' /etc/ssh/sshd_config
    # systemctl restart sshd

---

Benchmarks
----------

OK - it's time to stop doing everything as root. This step is done logged in as `odroid`.

Unlike the non-minimal image OpenJDK is not already installed, so the following step isn't needed:

    $ sudo apt-get purge 'openjdk-*'

Installing the Oracle JDK and Maven:

    $ sudo add-apt-repository ppa:webupd8team/java
    $ sudo apt-get update
    $ sudo apt-get install oracle-java8-installer
    $ sudo apt-get install git
    $ sudo apt-get install maven
    $ JAVA_HOME=/usr/lib/jvm/java-8-oracle mvn -version

This pulls in about 400MB. Now to clone and compile the benchmarks:

    $ git clone https://github.com/george-hawkins/naive-benchmarks.git
    $ cd naive-benchmarks/
    $ mvn clean compile

To run the benchmarks first run `free -m` and then set `MAVEN_OPTS` according to the value shown as available:

    $ export MAVEN_OPTS='-Xms1600m -Xmx1600m'
    $ export JAVA_HOME=/usr/lib/jvm/java-8-oracle
    $ time mvn exec:java@benchmarks -Dconfig.file=benchmarks-odroid-c2.conf

The benchmarks ran in much the same time as on the fully loaded non-minimal image.

For record the times were 303s for `processor`, 278s for `memory`, 296s for `disk` and 293s for `network`.

The memory result is slightly faster, about 8% faster, but I haven't investigated if this difference can be consistently demonstrated.

The time for the `network` benchmark is always remarkably consistent - 293s here as it has been on the non-minimal image and my x64_86 box.

---

TODO:

* Disable password based ssh login.
* Disable root ssh login altogether
* Create a spark user and use `ssh-copy-id` (see [Ask Ubuntu question](http://askubuntu.com/q/4830) for alternatives that don't require `ssh-copy-id`).
