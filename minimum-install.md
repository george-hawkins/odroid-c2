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
