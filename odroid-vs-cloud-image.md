First turn off the Odroid C2's intense blue LED:

    $ sudo vi /etc/rc.local

And add the following line before the `exit` line:

    echo none > /sys/class/leds/blue\:heartbeat/trigger

This has the nice side effect that you can see when `rc.local` has completed as the blue LED will go from solid to beating and then go out at this point.

---

Finding Odroids
---------------

[Fing](https://www.fing.io/) for iOS or Android is great for this. A command line alternative is nmap. For a quick scan do:

    $ nmap -T4 -F 192.168.1.133/24

Where `192.168.1.133` is your current machine and `/24` indicates you want to search for devices that differ from your IP only in there last 8 bits.  This runs in around 5s on my network.

If you have only one Odroid C2 on the local network then you'll most probably be able to use the nice name `odroid64.lan` rather than an IP address.

---

Default states
--------------

Odroid C2:

    $ df -h
    Filesystem      Size  Used Avail Use% Mounted on
    udev            732M     0  732M   0% /dev
    tmpfs           172M   14M  159M   8% /run
    /dev/mmcblk0p2   15G  4.4G  9.7G  32% /
    tmpfs           859M  100K  859M   1% /dev/shm
    tmpfs           5.0M  4.0K  5.0M   1% /run/lock
    tmpfs           859M     0  859M   0% /sys/fs/cgroup
    /dev/mmcblk0p1  128M   23M  106M  18% /media/boot
    cgmfs           100K     0  100K   0% /run/cgmanager/fs
    tmpfs           172M   16K  172M   1% /run/user/118
    tmpfs           172M     0  172M   0% /run/user/1000

Ubuntu arm64 cloud server:

    $ df -h        
    Filesystem      Size  Used Avail Use% Mounted on
    udev            451M     0  451M   0% /dev
    tmpfs            93M  2.9M   90M   4% /run
    /dev/vdb1       2.0G  1.3G  768M  62% /
    tmpfs           462M     0  462M   0% /dev/shm
    tmpfs           5.0M     0  5.0M   0% /run/lock
    tmpfs           462M     0  462M   0% /sys/fs/cgroup
    /dev/vdb15       98M  514K   97M   1% /boot/efi
    tmpfs            93M     0   93M   0% /run/user/1000

Get current packages:

    $ ssh odroid@odroid64.lan 'apt list --installed' > installed-odroid-c2.txt

    $ ssh -p 2222 ghawkins@localhost 'apt list --installed' > installed-ubuntu-cloud-arm64.txt

Give the Odroids there own RSA key (so it can be revoked separately if needed):

    $ ssh-keygen -t rsa -b 4096 -C 'odroid@odroid64.lan'
    $ cat ~/.ssh/id_rsa.pub 

Go to <https://github.com/settings/keys> and click "New SSH key" button and add the public part of the key.

Install Xenial backports to fix a bug with `appstreamcli` that causes `apt-get update` to complain about apparent metadata errors (see [Ask Ubuntu](http://askubuntu.com/q/854168)):

    $ sudo apt install appstream/xenial-backports
    $ appstreamcli --version

Note: with my original Odroid images it was only possible to install `xenial-backports` after doing an initial update (and seeing it complain) as the un-upgraded system didn't already know about `xenial-backports`.

Get rid of OpenJDK (actually just the JRE is installed by default) and install the Oracle JDK:

    $ sudo apt-get purge 'openjdk-*'
    $ sudo add-apt-repository ppa:webupd8team/java
    $ sudo apt-get update
    $ sudo apt-get install oracle-java8-installer

The full details can be found here - <http://www.webupd8.org/2012/09/install-oracle-java-8-in-ubuntu-via-ppa.html>

Install git and maven:

    $ apt-get install git
    $ sudo apt-get install maven

Maven needs `JAVA_HOME` to be set:

    $ export JAVA_HOME=/usr/lib/jvm/java-8-oracle
    
Now check that Maven is using the expected JDK:

    $ mvn -version

Clone the benchmarks and build:

    $ git clone git@github.com:george-hawkins/naive-benchmarks.git
    $ cd naive-benchmarks
    $ mvn clean compile

See how much memory is available:

    $ free -m
                  total        used        free      shared  buff/cache   available
    Mem:           1717         184        1229          19         302        1477
    Swap:           858           0         858

The `available` value rather than the `free` value is the crucial number here (see [SO](http://stackoverflow.com/a/30772733/245602)).

You can talk pretty much all of this (suspending the benchmarks, at a point when they should have allocated all the memory they can, and running `free` again shows a higher `available` value than you'd expect).

Edit `benchmarks.conf` and disable all the benchmarks except for the `memory` ones. Set `MAVEN_OPTS` using a value near that of `available` and run the benchmarks:

    $ export MAVEN_OPTS='-Xms1400m -Xmx1400m'
    $ mvn exec:java@benchmarks

The first lines of output should be something like:

    12:10:16.546 ... INFO ...Benchmarks - total free JVM memory - 1.2 GiB
    12:10:16.564 ... INFO ...Benchmarks - file system "/" has 9.3 GiB usable space

Most probably the benchmarks will now bail with an OOM exception. The above values though give you an idea of the upper limits for the `memory.size` and `disk.size` values in `benchmarks.conf`.

Note: these values are not for telling the benchmarks the actual memory and disk sizes, they're for telling the `memory` and `disk` benchmarks how much space to use.

I had to set `memory.size` to a size surprisingly lower than the 1.2GiB of reported free JVM memory - `memory.size` had to be set to just 900MiB in order for things to run without an OOM exception.

I then reenabled all the other benchmarks, ran the `network-server` job on another machine and updated the `network.server` value in `benchmarks.conf` with the IP address of that machine.

Then I ran the benchmarks again:

    $ mvn exec:java@benchmarks

From an initial run the value were:

    12:30:13.363 ...Benchmarks - total free JVM memory - 1.2 GiB
    12:30:13.382 ...Benchmarks - file system "/" has 9.3 GiB usable space
    12:30:13.383 ...Benchmarks - running processor benchmark
    12:30:50.939 ...AbstractBenchmark - processor load - min=9079ms, max=9483ms, median=9311ms, std-dev=217.313905
    12:30:50.941 ...Benchmarks - running memory benchmark
    12:31:22.039 ...AbstractBenchmark - memory read - min=452ms, max=474ms, median=452ms, std-dev=3.034033
    12:32:04.319 ...AbstractBenchmark - memory write - min=659ms, max=669ms, median=660ms, std-dev=1.552568
    12:32:04.320 ...Benchmarks - running disk benchmark
    12:39:15.073 ...AbstractBenchmark - file write - min=106170ms, max=108339ms, median=107600ms, std-dev=932.175010
    12:43:29.213 ...AbstractBenchmark - file read - min=63463ms, max=63707ms, median=63479ms, std-dev=117.358070
    12:43:29.274 ...Benchmarks - running network benchmark
    12:43:29.278 ...Network - client will write to 8001 and read from 8000 on 192.168.1.133
    12:43:29.280 ...Network - warming up server
    12:43:30.645 ...Network - completed server warmup
    12:43:44.581 ...AbstractBenchmark - network write - min=3476ms, max=3489ms, median=3480ms, std-dev=5.567764
    12:43:44.582 ...Network - network write speed is 941.6 Mb
    12:43:58.560 ...AbstractBenchmark - network read - min=3480ms, max=3498ms, median=3485ms, std-dev=7.889867
    12:43:58.562 ...Network - network read speed is 940.3 Mb
    12:43:58.563 ...Benchmarks - finished

So 37s for `processor`, 74s for `memory`, 11m 25s for `disk` and 28s for `network`.

Then for all of them, except `disk`, I adjusted the `cycles` values, using the times above, to try and achieve a run time of 5m for each. For `disk` I adjusted down the `disk.size` to try and also achieve a 5m run time.

The resulting `benchmark.conf` was:

    processor.enabled = true
    processor.cycles = 32
    processor.width = 6000

    memory.enabled = true
    memory.cycles = 260
    memory.size = 900MiB

    disk.enabled = true
    disk.cycles = 4
    disk.size = 3588MiB

    network.enabled = true
    network.cycles = 42
    network.count = 50000
    network.server = 192.168.1.133
    network.read.port = 8000
    network.write.port = 8001

The results were:

    13:02:06.557 ...Benchmarks - total free JVM memory - 1.2 GiB
    13:02:06.575 ...Benchmarks - file system "/" has 9.3 GiB usable space
    13:02:06.576 ...Benchmarks - running processor benchmark
    13:07:10.210 ...AbstractBenchmark - processor load - min=9056ms, max=9696ms, median=9483ms, std-dev=115.471330
    13:07:10.211 ...Benchmarks - running memory benchmark
    13:09:17.542 ...AbstractBenchmark - memory read - min=481ms, max=501ms, median=481ms, std-dev=1.471091
    13:12:13.060 ...AbstractBenchmark - memory write - min=674ms, max=685ms, median=675ms, std-dev=1.237345
    13:12:13.061 ...Benchmarks - running disk benchmark
    13:15:21.522 ...AbstractBenchmark - file write - min=46462ms, max=47240ms, median=46863ms, std-dev=317.730887
    13:17:12.437 ...AbstractBenchmark - file read - min=27598ms, max=27858ms, median=27724ms, std-dev=138.014492
    13:17:12.487 ...Benchmarks - running network benchmark
    13:17:12.491 ...Network - client will write to 8001 and read from 8000 on 192.168.1.133
    13:17:12.499 ...Network - warming up server
    13:17:13.799 ...Network - completed server warmup
    13:19:40.567 ...AbstractBenchmark - network write - min=3481ms, max=3504ms, median=3494ms, std-dev=5.094234
    13:19:40.576 ...Network - network write speed is 937.8 Mb
    13:22:06.884 ...AbstractBenchmark - network read - min=3480ms, max=3488ms, median=3482ms, std-dev=2.668263
    13:22:06.886 ...Network - network read speed is 941.1 Mb
    13:22:06.887 ...Benchmarks - finished

Even during the middle of running the processor benchmark (which uses all four cores) the air temperature just a few millimeters above the CPU end of the heatsink was just 6C higher than the room temperature.

Running the same configuration on a DELL OptiPlex 3020 with a four core 3.2GHz i5-4570 CPU with 15.6GiB RAM and a 500GB Samsung 850 EVO Basic running Ubuntu 16.04 LTS the results were:

    $ export MAVEN_OPTS='-Xms5400m -Xmx5400m'
    mvn exec:java@benchmarks -Dconfig.file=benchmarks-odroid-c2.conf
    21:12:15.547 ...Benchmarks - total free JVM memory - 4.8 GiB
    21:12:15.549 ...Benchmarks - file system "/" has 279.4 GiB usable space
    21:12:15.550 ...Benchmarks - using config ConfigOrigin(merge of system properties,benchmarks-odroid-c2.conf: 1)
    21:12:15.550 ...Benchmarks - running processor benchmark
    21:13:49.083 ...AbstractBenchmark - processor load - min=2791ms, max=3508ms, median=2894ms, std-dev=122.734892
    21:13:49.083 ...Benchmarks - running memory benchmark
    21:14:04.314 ...AbstractBenchmark - memory read - min=55ms, max=66ms, median=57ms, std-dev=1.662364
    21:14:28.316 ...AbstractBenchmark - memory write - min=90ms, max=121ms, median=91ms, std-dev=3.004350
    21:14:28.317 ...Benchmarks - running disk benchmark
    21:14:29.892 ...Disk - was able to allocate 3.5 GiB of 3.8 GiB apparently free memory
    21:15:01.015 ...AbstractBenchmark - file write - min=6851ms, max=7909ms, median=7423ms, std-dev=525.133237
    21:15:29.307 ...AbstractBenchmark - file read - min=7045ms, max=7112ms, median=7066ms, std-dev=28.711206
    21:15:29.361 ...Benchmarks - running network benchmark
    21:15:29.361 ...Network - client will write to 8001 and read from 8000 on 192.168.1.120
    21:15:29.361 ...Network - warming up server
    21:15:30.585 ...Network - completed server warmup
    21:17:57.000 ...AbstractBenchmark - network write - min=3481ms, max=3499ms, median=3485ms, std-dev=4.015649
    21:17:57.001 ...Network - network write speed is 940.3 Mb
    21:20:23.480 ...AbstractBenchmark - network read - min=3485ms, max=3495ms, median=3487ms, std-dev=2.577937
    21:20:23.480 ...Network - network read speed is 939.7 Mb
    21:20:23.480 ...Benchmarks - finished

The Odroid C2 is an order of magnitude slower - 304s vs 94s for `processor`, 303s vs 39s for `memory` and 299s vs 60s for `disk`. And this despite the fact that no effort was made to bring the OptiPlex to a state suitable for benchmarking, i.e. lots of other processes were running, including Chrome and Eclipse.

The network performance for both however isn't just close - it's identical at 293s for both.

The network itself is presumably the bottleneck - one might also expect disk performance to be an area with a similar issue.

However the Odroid C2 with a 16GB eMMC module performs dramatically slower for the disk benchmark despite claims that [eMMC](https://en.wikipedia.org/wiki/MultiMediaCard#eMMC) can rival SATA SSDs.

Presumably using a MicroSD card with the Odroid C2 would reduce performance even further.

Note: the OptiPlex ran the `network-server` for the Odroid C2 tests, while a MacBook Air (with a thunderbolt wired ethernet adapter) ran it for the OptiPlex tests.

---

Trimming down Odroid Ubuntu
---------------------------

First I got my QEMU emulate arm64 cloud server and the Odroid C2 into a fully upgraded and cleaned up state for comparison.

### QEMU emulate arm64 cloud server cleanup and upgrade

On my QEMU emulated arm64 cloud server I did:

    $ sudo apt install appstream/xenial-backports

See elsewhere for why backports was installed.

    $ sudo apt-get update
    $ sudo apt-get dist-upgrade

Aside: contrary to what I'd previously imagine `dist-upgrade` is just a more aggressive version of `upgrade` and not related in anyway to doing a major releasy upgrade of the distro.

    $ cat /var/run/reboot-required
    *** System restart required ***
    $ sudo reboot now

You are logged out - you can watch the shutdown and reboot in the terminal where you started the QEMU session. Oddly after the upgrade instead of a nice noisy text based boot sequence it did a kind of pulsing dots boot sequence - which didn't work very well in the non-graphicss console. Only the final login prompt and the final cloud-init output appeared properly.

Also oddly `sudo` started complaining about being `unable to resolve host ubuntu.localdomain` - I resolve this in the same way as the complaint about host `ubuntu` (see writeup of QEMU arm64 emulation).

Then on with the process...

    $ dpkg --list | grep 'linux-image' | awk '{ print $2 }' | sort -V | sed -n '/'"$(uname -r | sed "s/\([0-9.-]*\)-\([^0-9]\+\)/\1/")"'/q;p' | xargs sudo apt-get -y purge
    $ dpkg --list | grep 'linux-headers' | awk '{ print $2 }' | sort -V | sed -n '/'"$(uname -r | sed "s/\([0-9.-]*\)-\([^0-9]\+\)/\1/")"'/q;p' | xargs sudo apt-get -y purge

This remove old kernel images and there related headers (see this [SO answer](http://askubuntu.com/a/254585)).

On the cloud image this step did nothing (there were no old images) - when repeating these steps on the Odroid it cleared off 11 old kernel images (but no headers).

    $ sudo apt-get autoremove
    $ sudo apt-get clean

After these steps `df` shows that disk usage on `/` has actually gone up by 27 MB.

### Repeating these steps on Odroid

The same set of steps on the Odroid went much the same except...
    
During the upgrade it warns you about changes to `boot.ini` - if you compare the old and new `boot.ini` after the changes you can see that both the old and new file are from Odroid, i.e. there's no issue of some Odroid specific version being overwritten with an Ubuntu generic version, and if you look at the changes there's nothing very interesting.

Running the `clean` step resulted in the output:

    $ sudo apt-get clean
    W: Problem unlinking the file apt-fast - Clean (21: Is a directory)

This seems to be an `apt-fast` issue ([#100](https://github.com/ilikenwf/apt-fast/issues/100)) that can be resolved like so:

    $ sudo rm -r /var/cache/apt/archives/apt-fast
    $ sudo vi /etc/apt-fast.conf

And change the `DLDIR` value to `/var/cache/apt/apt-fast` (the comments in the issue suggest a different value - but this is the value you see in the commit to fix the issue).

Once complete `df` shows no change between the before and after state.

---

The Odroid C2 runs [Ubuntu MATE 16.04 LTS](https://ubuntu-mate.org/download/) - I downloaded the x64_86 image and installed it as a virtual machine using VirtualBox.

Note: VirtualBox defaults to an 8GB disk image - MATE requires at least 8.6GB. I set the disk size to 16GB and the memory to 2GB. Normally I'd opt for LVM but I didn't as the Odroid doesn't use it.

Once installed I added the guest additions:

    $ sudo apt-get install virtualbox-guest-dkms
    $ sudo apt-get install virtualbox-guest-x11

To me it looks like the `virtualbox-guest-x11` package should have pulled in all necessary dependencies but it did not pull in `virtualbox-guest-dkms` and (unsurprisingly) the guest additions failed to work until this was done.

After installing `virtualbox-guest-dkms` the system is very slow to reboot - it seems it's waiting 90s for a process to die - see Ubuntu MATE issue [#5524](https://ubuntu-mate.community/t/ubuntu-mate-16-04-sometimes-needs-long-to-shut-down/5524).

After going through the first three steps of the cleanup process described above (installing `backports` and doing an `update` and `dist-upgrade`) this continued to happen but at least stopped looking like a hung system - if you pressed escape (to flip to the text based shutdown output) it at least told you it was waiting 90s on a particular process.

---

See [minimum-install.md](minimum-install.md) for details on installing a minimal Odroid image.

---

TODO:

* Tuned is Redhat specific - there's no Ubuntu equivaluent.
* `sudo apt-get autoclean`
* Do an apt-get update and upgrade all.
* Reduce installed packages down to just those used by cloud images.
* Remove unused kernels - see <http://askubuntu.com/questions/2793/how-do-i-remove-old-kernel-versions-to-clean-up-the-boot-menu>
* Compare filesystem of Odroid after these steps with QEMU arm64 cloud server.
* Think about ansible, docker and maybe dnsmasq and maybe Ambari.
* See if you can read voltage from somewhere under `/dev` and see that working off power hub doesn't affect performace.

The Odroid C2 seems to have no hardware monitor files under `/sys/class/hwmon` (see [sysfs-interface](https://www.kernel.org/doc/Documentation/hwmon/sysfs-interface)) and the following finds no sensors:

    $ sudo apt-get install lm-sensors
    $ sudo sensors-detect
