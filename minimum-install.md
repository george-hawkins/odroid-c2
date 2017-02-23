Minimal Odroid image
====================

The normal Odroid image is Ubuntu MATE with a whole load of extras (Kodi etc.) installed.

On Aug 4th, 2016 Hardkernel [announced](https://plus.google.com/102407583165771869503/posts/XSnZxZxNz2A) minimal images.

Before intalling such an image to an eMMC module (or Micro SD card) let's first...

Determine the correct device
----------------------------

Before you insert the eMMC module that you intend to write to (using a USB card reader or whatever) do:

    $ ls /dev/disk/by-id

Now insert the USB card reader with eMMC module and do the same again:

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

    $ sudo dd if=$DISK bs=512 count=$(($(stat -c%s $IMAGE)/512)) | md5sum

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

It can take a while the device starts up for the first time (it has to resize disks and other things on the first boot).

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

Armbian optimizations
---------------------

The [Armbian project](https://www.armbian.com/) produces lightweight distributions based on Xenial for ARM development boards including the Odroid C2. I haven't tried the distribution itself - but the [board specific notes](https://docs.armbian.com/board_details/odroidc2/) for the Odroid C2 includes some usefule information.

Running in headerless mode:

    # sed -i 's/setenv nographics "0"/setenv nographics "1"/' /media/boot/boot.ini

This frees up a really substantial amount of RAM - 267MB according to `free` (comparing the difference between a clean booted system before and after the change).

Turn off the USB hub:

    # echo 126 > /sys/class/gpio/export
    # /bin/sh -c 'echo 0 > /sys/class/gpio/gpio126/value'

This should save 170mW - my power monitor showed about 155mW - a 7% power saving. Turning off graphics should apparently have saved a similar amount, but while I saw the expected memory saving, I didn't see any change in power consumption.

Note: turning off USB does not persist across reboots, i.e. it's back on after a reboot. I haven't written a unit file, like the one for the blue LED, to apply this change on every boot.

---

Backup
------

Once you've created the perfect setup you can back it up and restore it to other eMMC modules.

You could use `dd` for this but a better tool is FSArchiver. Ideally you could backup both partitions on the eMMC module with a single command (as described in the [FSArchiver quick start guide](https://www.fsarchiver.org/quickstart.html)). This should be possible with the version of FSArchiver that comes with Yakkety and later but the version on Xenial doesn't support vfat and this is the file system type used for the Odroid C2 boot partition. So you still need to use `dd` for the boot partition and `sfdisk` needs to be used to create the partitions. I've rolled this all up into the two small scripts here [`fs-backup`](fs-backup) and [`fs-restore`](fs-restore).

First:

    $ sudo apt install fsarchiver

Then to backup insert the eMMC module and:

    $ ./fs-backup "$DISK" my-backup-dir

This takes about a minute and creates `my-backup-dir` and stores the partition table and the archived contents of the partitions there.

Then swap in another eMMC module and clone the result to it:

    $ ./fs-restore "$DISK" my-backup-dir
    $ udisksctl power-off --block-device "$DISK"

Spark slave setup
-----------------

On the Odroid C2 install the Oracle JDK:

    # add-apt-repository ppa:webupd8team/java
    # apt-get update
    # apt-get install oracle-java8-installer

Add a user called `spark`:

    # useradd spark --create-home --shell /bin/bash
    # echo spark:spark | chpasswd
    # adduser spark sudo
    # exit

On your local system copy you public key to the `spark` user and then return to the Odroid one last time as root:

    $ ssh-copy-id spark@192.168.1.240
    $ ssh root@192.168.1.240

Disable password based login via ssh and disable root login via ssh altogether:

    # sed -i -e 's/PermitRootLogin yes/PermitRootLogin no/' -e 's/#PasswordAuthentication yes/PasswordAuthentication no/' /etc/ssh/sshd_config
    # systemctl restart sshd

Clean out setup steps (including passwords) from history and exit:

    # set +o history
    # rm .bash_history
    # exit

Download a prebuilt Spark distribution and install it on the Odroid:

    $ scp spark-2.1.0-bin-hadoop2.7.tgz spark@192.168.1.240:
    $ ssh spark@192.168.1.240
    $ tar -xf spark-2.1.0-bin-hadoop2.7.tgz
    $ rm spark-2.1.0-bin-hadoop2.7.tgz
    $ cd spark-2.1.0-bin-hadoop2.7
    $ ln -s $PWD ~/spark-home
    $ ./sbin/start-slave.sh spark://192.168.1.133:7077
    $ cat logs/*worker*.out

Note: `spark-2.1.0-bin-hadoop2.7` is fairly substantial - it contains lots of examples etc. However the `jars` subdirectory dominates everything else - so there's not much to be gained by pruning the examples and such like. If you look at the log output you'll see that the slave uses the classpath:

    .../spark-2.1.0-bin-hadoop2.7/conf/:.../spark-2.1.0-bin-hadoop2.7/jars/*

I.e. it depends on multiple jars in the the `jars` directory, rather than using e.g. a fat jar. So in the end all I remove is the original tar file.

Note: the soft link `spark-home` is used by the `spark-slave.service` unit file installed later.

Now stop things, exit and copy over the Spark slave systemd unit file and set things up for automatic startup on boot:

    $ ./sbin/stop-slave.sh 
    $ exit

The systemd unit file specifies that the Spark master is running on `spark-master.local`. You can setup the master to advertise this mDNS name (this is described elsewhere). But if the master is not advertising this name then you'll need to edit `spark-slave.service` to replace `spark-master.local` with the IP address of the master (or its DNS resolvable name if it has one) before copying it to the Odroid C2.

    $ scp spark-slave.service spark@192.168.1.240:
    $ ssh spark@192.168.1.240

    $ sudo mv spark-slave.service /etc/systemd/system
    $ sudo systemctl daemon-reload

If you are using an mDNS name, i.e. `spark-master.local`, in `spark-slave.service` then you'll need to install `avahi-daemon` so the name can be resolved:

    $ sudo apt-get install avahi-daemon

Start the service, check the status, stop it and again check the status:

    $ sudo systemctl start spark-slave
    $ sudo journalctl --unit spark-slave
    $ sudo systemctl stop spark-slave
    $ systemctl status spark-slave

Enable it for automatic start on reboot and then reboot:

    $ sudo systemctl enable spark-slave
    $ sudo reboot now

If you open the web UI for the master in your browser you should see the slave connect - the Spark slave takes quite a while to startup so this happens a noticeable amount of time after the Odroid has booted.

Final steps
-----------

Before cloning and copying to other eMMC modules:

    $ sudo systemctl stop spark-slave
    $ rm spark-home/logs/*
    $ set +o history
    $ rm .bash_history
    $ sudo shutdown now

Once you've ditached the eMMC module and inserted into a card reader on your main machine backup the image (that will then be cloned to further modules) and then change the hostname from `odroid64` to the first in our sequence of names:

    $ ./fs-backup "$DISK" spark-slave-odroid-backup
    $ ./set-hostname "$DISK" odroid64 spark-slave-1

Label the eMMC module with a sticker marked "1" and then proceed inserting further modules and cloning the image to those modules, giving each a name (and then a sticker):

    $ ./fs-restore "$DISK" spark-slave-odroid-backup
    $ ./set-hostname "$DISK" odroid64 spark-slave-2
    $ ./fs-restore "$DISK" spark-slave-odroid-backup
    $ ./set-hostname "$DISK" odroid64 spark-slave-3
    $ ./fs-restore "$DISK" spark-slave-odroid-backup
    $ ./set-hostname "$DISK" odroid64 spark-slave-4

Once their all powered up and you've seen them in action you can shut them all down like so:

    $ for i in 1 2 3 4
    do
        ssh -t spark@spark-slave-$i.local sudo shutdown now
    done

You have to enter the password for sudo for each system (and press enter, tilde, dot to kill each ssh sesssion in turn).

TODO:

* Describe briefly advertising `spark-master.local` with link to <https://github.com/george-hawkins/avahi-aliases-notes>
* Run benchmark one more time once everything is in octogon - to check power supply and longer cables (without shielding) don't affect things.

---

Spark standalone cluster
------------------------

TODO: this section used to go before the bit about setting up the Spark slave - it probably needs some restructuring to reflect its new position.

See <http://spark.apache.org/docs/latest/spark-standalone.html>

Using the Spark download unpacked earlier:

    $ cd .../spark-2.1.0-bin-hadoop2.7
    $ export SPARK_MASTER_HOST=192.168.1.133
    $ ./sbin/start-master.sh
    $ fgrep spark: logs/*master*.out                                                                             
    17/02/15 18:02:33 INFO Master: Starting Spark master at spark://...:7077

There's various other interesting information in the logs, such as web UI and REST endpoints - and various warnings that might be worth examining, e.g. the lack of Hadoop native libraries (see [SO](http://stackoverflow.com/q/19943766/245602)).

Note: I had to explicitly set `SPARK_MASTER_HOST` (to the non-loopback address for my machine, found with `ip -4 -br addr`) as otherwise it defaulted to a loopback address that couldn't be accessed by the Spark slaves (see [SPARK-19657](https://issues.apache.org/jira/browse/SPARK-19657)).

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

Spark
-----

There are no end of web pages on getting started with Spark - many of them already going stale.

The best initial course of actions seems to be:

* Download Spark and go through the included README.
* Work through <http://spark.apache.org/docs/latest/quick-start.html>
* Work through the introductory notebooks which can be accessed as part of the Databricks [Community Edition](https://community.cloud.databricks.com/).

The Databricks notebooks aren't terribly well maintained, there are broken links, spelling mistakes and a mix of Scala 1.X and 2.X usages.

### Notes

Download the latest version from <http://spark.apache.org/downloads.html>, i.e. the default download option that's shown.

Look through the `README`.

Running `spark-shell` for the first time will create the directory `metastore_db`, along with the file `derby.log`, in your current directory (and your history ends up in `~/.scala_history`).

Running something like `run-example SparkPi` results in the directory `spark-warehouse` being created in your current directory.

Note: the Scala REPL supports tab-completion for methods etc.

Then work through <http://spark.apache.org/docs/latest/quick-start.html>

Note: using `local[4]` with the `--master` argument to `spark-submit` (the final example) means run the job locally with four worker threads - see [master URLs](http://spark.apache.org/docs/latest/submitting-applications.html#master-urls).

When `spark-shell` starts up it shows a URL for a Web UI (that is only available while `spark-shell` itself is running).

If you run something like this:

    scala> val textFile = sc.textFile("README.md")
    scala> val wordCount = textFile.flatMap(_.split(" ")).map((_, 1)).reduceByKey(_ + _)
    scala> wordCount.collect()

Then this results in a job that you can look at in the Web UI - if you click on the job you can then look at a DAG visualization of it and click on the various stages for more information.

You can use the information shown to help improve the performance of your jobs.

Note: there's no job in this example until the `collect()` step - everything previously is just lazy setup.

---

Once you've completed the quick start you can move on to the Databricks notebook "A Gentle Introduction to Apache Spark on Databricks".

Various introductory sites link directly to this notebook but it makes more sense to access it as a notebook within Databricks.

It's the first notebook you see in the featured notebooks section of the <https://community.cloud.databricks.com/> main page. If you access it this way then it's already imported into your workspace.

The workspace layout seemed a little confusing initially - just go to users, there you see the email address you registered with, right click on this and you can create or import additional notebooks here.

If you carry on from the "gentle" notebook to the data scientist and data engineer notebooks (mentioned in the introductory paragraph) you will have to import these explicitly.

Initially the "gentle" notebooks is fairly slow going. The section on the RDD, DataFrame and Dataset clarifies how these three ideas relate. RDD is the original, then came DataFrames (similar to the same named concept in Pandas (Python) and R) and finally Dataset is the newest collection - according to the notebook it "provides the typed interface that is available in RDDs while providing a lot of conveniences of DataFrames. It will be the core abstraction going forward."

The Databricks "gentle" introduction is part of a series that includes two further notebooks one for data engineers, that's worth going through, and one for data scientists which is of less value (unless you are a data scientist).

---

### Notes on the Databricks notebooks

Within a notebook double click any textual cells to see and edit the markdown.

Notebooks support tab completion but it seems a bit flakey, pressing tab a few times often seems necessary and sometimes it just gets things completely wrong.

It turns out that in the `org.apache.spark.sql` package class one finds that [`DataFrame` is defined in terms of `Dataset`](https://spark.apache.org/docs/latest/api/scala/index.html#org.apache.spark.sql.package@DataFrame=org.apache.spark.sql.Dataset[org.apache.spark.sql.Row]):

    type DataFrame = Dataset[Row]

You can see a nice little graphic covering this at the end of the [Databricks Quick Start](https://databricks.com/product/getting-started-guide/quick-start) that also introduces the idea of `DataFrame`, i.e. `Dataset[Row]`, being the untyped API while `Dataset[T]` is the typed API.

From the "gentle" notebook:

> This is one of the core reasons that users should be focusing on using DataFrames and Datasets instead of the legacy RDD API.
> With DataFrames and Datasets, Apache Spark will work under the hood to optimize the entire query plan and pipeline entire steps together.

Prior to Spark 2.X you had `SparkContext` and `SQLContext` - they still exist in 2.X but you now have `SparkSession` (which has fields `sparkContext` and `sqlContext`) which tries to provide a unified interface.

Many examples (including those on the Databricks site) still refer to the pre-2.X entities (usually as `sc` and `sqlContext`) while newer examples work with `SparkSession` (usually as `spark`).

Note: in any Databricks notebook `sc`, `sqlContext` and `spark` are immediately available, in `spark-shell` just `sc` and `spark` are immediately available (but as noted `spark.sqlContext` gets you to an `SQLContext`).

`SparkContext` works in terms of the older RDD entities while `SparkSession` works with datasets and dataframes. So:

    sc.parallelize(Seq(1, 2, 3)) // Returns an RDD instance.
    spark.createDataset(Seq(1, 2, 3)) // Returns a Dataset instance.

The `dbfs:` scheme seems to be optional when using `%fs`, i.e. `%fs ls /...` generates the same result as `%fs ls dbfs:/...`.

Further getting started resources
---------------------------------

<https://www.simple-talk.com/cloud/cloud-data/start-big-data-apache-spark/> is a more wordy walkthru of:

* <http://spark.apache.org/docs/latest/quick-start.html>
* The "Gentle Introduction to Apache Spark on Databricks" notebook.

<https://databricks.com/product/getting-started-guide/quick-start> similarly mainly rehashes information from the above two - adding a few extra useful details.

The full Databricks documentation can be found at <https://docs.cloud.databricks.com/docs/latest/databricks_guide/index.html>

[Apache Zeppelin](https://zeppelin.apache.org/) provides a web based notebook implementation that you can use locally that looks extremely similar to the notebook implementation provided by Databricks.

The Hortonworks [Spark in 5 minutes](http://hortonworks.com/hadoop-tutorial/hands-on-tour-of-apache-spark-in-5-minutes/) uses Zeppelin (and the [Hortonworks Sandbox](http://hortonworks.com/hadoop-tutorial/learning-the-ropes-of-the-hortonworks-sandbox/#what-is-the-sandbox) that includes [Ambari](https://ambari.apache.org/) and many other things ready setup) to introduce Spark much as the Databricks "gentle" notebook does. The introduction ends with a link to the more advanced tutorials from Hortonworks.

MapR has a nice [multi-page overview](https://www.mapr.com/ebooks/spark/) of Spark, e.g. the [architectural page](https://www.mapr.com/ebooks/spark/03-apache-spark-architecture-overview.html) provides lots of interesting details, and it has nice sections like ["Hadoop vs. Spark - An Answer to the Wrong Question"](https://www.mapr.com/ebooks/spark/04-hadoop-and-spark-benefits.html). However it doesn't seem to have been updated for Spark 2.X.

The slides for Databricks one day workshop introducing Spark are available - unfortunately there seem to be different versions floating about, there's the [Databricks version](http://training.databricks.com/workshop/itas_workshop.pdf) and there's one [hosted by a professor at Stanford](https://stanford.edu/~rezab/sparkclass/slides/itas_workshop.pdf). The Stanford is from 2014 (and uses Spark 0.9.1) while the one from Databricks is from 2015 (and uses Spark 1.2.0). The Stanford one is interesting as it contains an "Advanced Topics" section that's not in the Databricks version (and despite being earlier includes details about projects like Shark that are mentioned as in-progress in the Databricks version).

The slides are actually readable, i.e. they're not just vague bullet points, and while they only cover low level RDD usage, they're interesting as they go fairly in-depth and cover the whole Spark ecosystem, i.e. all the other projects you're likely to come in contact with when looking at Spark.

Another light introduction is <https://www.infoq.com/articles/apache-spark-introduction>. While it just rehashes the same examples seen elsewhere, when in comes to Spark itself, it (like the Databricks workshop slides) provides a broader view and briefly covers some of the Spark ecosystem and makes some effort to explain the relationship between Spark and Hadoop. Note: the Spark examples use version 1.2.0.

The MapR overview, the Databricks workshop slides and the InfoQ introduction all provide a broader view rather than just providing a straight dive into Spark.

As already noted up above many older Spark related pages involve creating `SparkConf`, `SparkContext` and other entities individually. In Spark 2.X these are all rolled up into a unified entry point called `SparkSession`. Databricks have a nice [blog entry](https://databricks.com/blog/2016/08/15/how-to-use-sparksession-in-apache-spark-2-0.html) introducing `SparkSession` (and discusses how to use it in place of the Spark 1.X constructs).

Moving from `SparkContext` etc. to using `SparkSession` is also covered in a bit more detail by <http://vishnuviswanath.com/spark_session.html>

Next steps
----------

See <http://spark.apache.org/docs/latest/#where-to-go-from-here>

In particular the [Programming Guide](http://spark.apache.org/docs/latest/programming-guide.html) and the [Cluster Overview](http://spark.apache.org/docs/latest/cluster-overview.html) (in particular the [standalone section](http://spark.apache.org/docs/latest/spark-standalone.html)).

---

Building Spark
--------------

    $ git clone git@github.com:apache/spark.git
    $ cd spark
    $ git tag
    $ git checkout v2.1.0
    $ ./dev/change-scala-version.sh 2.11
    $ build/mvn -T4 -DskipTests clean package

Note: I did not need, as suggested, to increment my maximum JVM heap size or set `ReservedCodeCacheSize`. I think the default heap size with Java 8 on a 64 bit machine with a reasonable amount of memory (16GB in my case) is already very large. However I did initially run out of memory during the build - but this wasn't a JVM issue, it was Chrome (as usual) hogging all the memory - once I quit out of Chrome and released enough memory it built fine.

The use of `change-scala-version` was because Maven errored out with "Failed to execute goal net.alchim31.maven:scala-maven-plugin" and this was the solution (that I found on [SO](http://stackoverflow.com/a/32686011/245602)).

Running in IntelliJ
-------------------

Importing into IntelliJ is described in the "IDE Setup" section of <http://spark.apache.org/developer-tools.html>

Despite having built everything with Maven it seems necessary on importing it into IntelliJ to rebuild it (Build / Rebuild Project).

On doing so I hit this problem described in the [SO question "Error:... not found: type EventBatch"](http://stackoverflow.com/questions/33311794/import-spark-source-code-into-intellj-build-error-not-found-type-sparkflumepr). The accepted answer worked but it's a little unclear - here's exactly what I did in the end:

Went to File / Project Structure... Then to Project Settings / Modules. Selected spark-streaming-flume-sink. Selected `target` - which was marked as excluded. I untoggled the Excluded button for it. Then I selected all the directories immediately below `target`, except for `scala-2.11`, and marked them as excluded. I then expanded `scala-2.11` and marked its immediate directories, except for `src_managed`, as excluded. I then expanded `src_managed` and selected `compiled_avro` and toggled the Sources button for it to mark it as a source directory. Then once I'd pressed OK I went back to Build / Rebuild Project - and everything compiled fine.

I then navigated to the `org.apache.spark.deploy.master.Master` companion object and ran it.

You then see:

    Exception in thread "main" java.lang.NoClassDefFoundError: com/google/common/cache/CacheLoader

The reason for this is fairly clear - if on the command line you do:

    $ cd core
    $ mvn dependency:tree

You'll see all the dependencies are marked as `compile`, i.e. they shouldn't be used at runtime.

Oddly I couldn't find any clear answer to what to do about this - one could edit the pom (or more precisely its parent pom) to change the scope from `compile` but this is called the naive approach on the JetBrains/intellij-scala wiki page [How to use provided libraries in run configurations](https://github.com/JetBrains/intellij-scala/wiki/%5BSBT%5D-How-to-use-provided-libraries-in-run-configurations).

While this page specifically mentions the Spark project it describes an SBT solution - as the Spark project uses Maven it's not clear how one can apply the approach described there.

In the end I just searched to see if these jars had been grouped up together by the Maven process in any subdirectory of the Spark project:

    $ find . -name 'guava*.jar'
    ./assembly/target/scala-2.11/jars/guava-14.0.1.jar
    ./core/target/jars/guava-14.0.1.jar

I chose the directory under `assembly` as it appeared to have all the jars of interest. Then in the Project window I right clicked on core, went to Open Module Settings, selected the Dependencies tab, selected the last entry in the list and pressed `+` to add an entry below it. On pressing `+` I selected JARs or directories... and navigated to the `jars` folder down below `assembly` and then after pressing OK changed its scope to Runtime.

After a long indexing pause, I ran `Master` again - this kicked off a rebuild again.

Then I went to Run / Edit Configurations and set the Program Arguments to `--host <MyHostName> --port 7077 --webui-port 8080`, i.e. the arguments you see if you run `start-master.sh` and then use `ps -ef` to see the arguments passed the JVM that's started.

Note that without any arguments it uses the public IP of your machine for the Spark master URI and uses 6066 rather than 7077 for the REST server port.

So it's the `start-master.sh` explicitly passing in a hostname that causes the issue that it uses a name that resolves to the loopback address on Ubuntu and Debian systems (that have no FQDN and/or static IP).
