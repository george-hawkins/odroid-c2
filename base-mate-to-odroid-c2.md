Packages installed on top of the base Ubuntu MATE to match the normal Odroid C2 setup
-------------------------------------------------------------------------------------

Using the `do-pass` script the packages were installed one by one on the basis of which would pull in the most dependencies first:

6 synaptic
6 kodi-bin
5 unity-settings-daemon
5 python-cryptography
5 nautilus-sendto
5 mesa-common-dev
5 lintian
5 kodi
5 graphviz
4 upstart-monitor
4 unity-greeter
4 ssh
4 qtbase5-dev
4 linux-tools-generic
4 gnome-software
3 murrine-themes
3 build-essential
2 va-driver-all
2 ubuntu-mate-wallpapers-complete
2 qt-at-spi
2 lirc
2 indicator-applet
2 gnome-themes-standard
1 vim
1 systemd-shim
1 smartmontools
1 rhythmbox-plugin-zeitgeist
1 python-ndg-httpsclient
1 myspell-en-za
1 myspell-en-gb
1 libreoffice-help-en-gb
1 libhtml-format-perl
1 gnome-system-monitor
1 chromium-browser
1 aria2

Each line starts with the number of additional packages pulled in by each package.

The following packages were then installed in one go as none of them pull in additional packages:

cpufrequtils
cracklib-runtime
desktop-base
dns-root-data
enchant
fbset
fonts-stix
gdbserver
gnome-icon-theme-gartoon
gnome-user-guide
hexchat-perl
hexchat-plugins
hexchat-python
hunspell-en-ca
ippusbxd
joe
kernel-common
libc6-dbg
libclutter-1.0-common
libcogl-common
libcrossguid0
libhtml-form-perl
libhttp-daemon-perl
liblouis-bin
libnet-libidn-perl
libqt4-sql-mysql
libqt5svg5
libreoffice-l10n-en-za
libreoffice-math
libreoffice-style-breeze
libreoffice-style-tango
libtie-ixhash-perl
libvdpau-va-gl1
libwacom-bin
libwebkit2gtk-4.0-37-gtk2
libxml-xpathengine-perl
light-themes
lm-sensors
menu
mesa-utils-extra
myspell-en-au
mythes-en-au
notification-daemon
pastebinit
plymouth-themes
policykit-1-gnome
python3-bs4
python3-html5lib
python3-smbc
python3-uno
qt5-default
qttranslations5-l10n
signon-ui
thunderbird-locale-en-gb
u-boot-tools
ubuntu-ui-toolkit-theme
vlc-plugin-samba
xvt
zram-config

The following packages were not installable (with the reason given by apt-get):

Couldn't find any package by glob or regex...

	linux-image-3.14.79-104
	linux-tools-4.4.0-62
	linux-tools-4.4.0-62-generic

Package has no installation candidate...

	fonts-droid
	u-boot

Unable to locate package...

	aml-libs
	apt-fast
	arm-mali-examples
	bootini
	fix20160327
	fix20160525
	libaccount-plugin-facebook
	libaccount-plugin-flickr
	libdrm-freedreno1
	libdrm-tegra0
	libp8-platform2
	linux-image-c2
	mali-x11
	xserver-xorg-video-mali
