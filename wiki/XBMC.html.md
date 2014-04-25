XBMC
====

XBMC (formerly "Xbox Media Center") is a free, open source (GPL)
multimedia player that originally ran on the first-generation XBox, (not
the newer Xbox 360), and now runs on computers running Linux, Mac OS X,
Windows, and iOS. XBMC can be used to play/view the most popular video,
audio, and picture formats, and many more lesser-known formats,
including:

-   Video - DVD-Video, VCD/SVCD, MPEG-1/2/4, DivX, XviD, Matroska
-   Audio - MP3, AAC.
-   Picture - JPG, GIF, PNG.

These can all be played directly from a CD/DVD, or from the hard-drive.
XBMC can also play multimedia from a computer over a local network
(LAN), or play media streams directly from the Internet. For more
information, see the XBMC FAQ.

As of version 12, it can also be used to play and record live TV using a
tuner, a backend server and a PVR plugin; more information about this
can be found on the XBMC wiki.

Contents
--------

-   1 Installation
-   2 Configuration
    -   2.1 Autostarting at boot
    -   2.2 Sharing a Database Between Multiple XBMC PCs
    -   2.3 Using a remote controller
        -   2.3.1 MCE remote with Lirc and Systemd
        -   2.3.2 HDMI-CEC with Pulse Eight USB-CEC
    -   2.4 Fullscreen mode stretches XBMC across multiple displays
    -   2.5 Slowing down CD/DVD drive speed
-   3 See also

Installation
------------

Install xbmc from the official repositories. Optionally install
xbmc-pvr-addons if users wish to use the pvr extensions of xbmc.

Configuration
-------------

> Autostarting at boot

It is desirable to start XBMC automatically on boot. Since version
11.0-11, the xbmc package will automatically create the xbmc group,
user, and provide an xbmc.service so systemd can manage xbmc without the
need for a DE.

To make XBMC start at system boot, enable the service:

    # systemctl enable xbmc

> Sharing a Database Between Multiple XBMC PCs

Provided that a box on the network is running mariadb, one can easily
configure multiple xbmc boxes to share a database. The advantage of this
is that key meta are stored in one place, a show can be paused on one
box and then resumed on another seamlessly, and the record of what has
been watched is unified.

Setup of this is beyond the scope of this article. Consult the Setting
up MySQL for Arch Linux hosted by the XBMC project wiki.

> Using a remote controller

As XBMC is geared toward being a remote-controlled media center; any PC
with a supported IR receiver/remote, can use remote using LIRC or using
the native kernel supported modules. To work properly with xbmc, a file
will be required that maps the lirc events to xbmc keypresses. Create an
XML file at ~/.xbmc/userdata/Lircmap.xml (note the capital 'L').

Note:Users running xbmc from the included service file will find the
xbmc home (~) under /var/lib/xbmc and should substitute this in for the
shortcut above. Also make sure that if creating this file as the root
user, it gets proper ownership as xbmc:xbmc when finished.

Lircmap.xml format is as follows:

    <lircmap>
      <remote device="devicename">
          <XBMC_button>LIRC_button</XBMC_button>
          ...
      </remote>
    </lircmap>

-   Device Name is whatever LIRC calls the remote. This is set using the
    Name directive in lircd.conf and can be viewed by running $ irw and
    pressing a few buttons on the remote. IRW will report the name of
    the button pressed and the name of the remote will appear on the end
    of the line.

-   XBMC_button is the name of the button as defined in keymap.xml.

-   LIRC_button is the name as defined in lircd.conf. If lircd.conf was
    autogenerated using # irrecord, these are the names selected for the
    buttons. Refer back to LIRC for more information.

-   A very thorough Lircmap.xml page over at the XBMC Wiki should be
    consulted for more help and information on this subject as this is
    out of scope of this article.

MCE remote with Lirc and Systemd

Install lirc-utils and link the mce config:

    # ln -s /usr/share/lirc/mceusb/lircd.conf.mceusb /etc/lirc/lircd.conf

Then, make sure the remote is using the lirc protocol:

    $ cat /sys/class/rc/rc0/protocols

If not, issue:

    # echo lirc > /sys/class/rc/rc0/protocols

A udev rule can be added to make lirc the default. A write rule does not
seem to work, so a simple RUN command can be executed instead.

    /etc/udev/rules.d/99-lirc.rules

    KERNEL=="rc*", SUBSYSTEM=="rc", ATTR{protocols}=="*lirc*", RUN+="/bin/sh -c 'echo lirc > $sys$devpath/protocols'"

Note:If this does not work, follow the suggestion to use tmpfiles.d as
specified in the LIRC wiki to set the remote to the lirc protocol at
boot time.

Next, specify the lirc device. This varies with kernel version. As of
3.6.1 /dev/lirc0 should work with the default driver.

    /etc/conf.d/lircd.conf

    #
    # Parameters for lirc daemon
    #

    LIRC_DEVICE="/dev/lirc0"
    LIRC_DRIVER="default"
    LIRC_EXTRAOPTS=""
    LIRC_CONFIGFILE=""

The default service file for lirc ignores this conf file. So we need to
create a custom one.

    /etc/systemd/system/lirc.service

    [Unit]
    Description=Linux Infrared Remote Control

    [Service]
    EnvironmentFile=/etc/conf.d/lircd.conf
    ExecStartPre=/usr/bin/ln -sf /run/lirc/lircd /dev/lircd
    ExecStartPre=/usr/bin/ln -sf /dev/lirc0 /dev/lirc
    ExecStart=/usr/sbin/lircd --pidfile=/run/lirc/lircd.pid --device=${LIRC_DEVICE} --driver=${LIRC_DRIVER}
    Type=forking
    PIDFile=/run/lirc/lircd.pid

    [Install]
    WantedBy=multi-user.target

Finally, enable and start the lirc service:

    # systemctl enable lirc
    # systemctl start lirc

This should give a fully working mce remote.

HDMI-CEC with Pulse Eight USB-CEC

An elegant way of getting remote functions in XBMC is using CEC, a
protocol that is part of the HDMI specification. Most modern TVs support
CEC, although some manufacturers advertise the feature under other
names. Apart from a CEC-enabled TV some hardware that takes the CEC
signals coming from the TV and present them in a way that XBMC can
understand is also needed. One such device is the USB-CEC adapter from
Pulse Eight. Hooking up the USB-CEC is pretty simple, but in order for
it to work in Arch we have to do a few things.

First of all, make sure libcec is installed.

    # pacman -S libcec

When connected, the USB-CEC's /dev entry (usually /dev/ttyACM*) will
default to being owned by the uucp group, so in order to use the device
the user running XBMC needs to belong to that group. The user also needs
to belong to the lock group, otherwise XBMC will be unable to connect to
the device. To add a user to both groups, run

    # usermod -aG uucp,lock [username]

If you have more than one user that uses XBMC, repeat the command for
all those users. If, for example, one is using xbmc-standalone, the
relevant command is

    # usermod -aG uucp,lock xbmc

Remember that modifying the groups of any logged in users means those
users need to log out and login again in order for the changes to take
effect.

Note:Trying to use the USB-CEC without belonging to above groups may
lead to problems, including XBMC crashes, so make sure the correct user
belongs to both groups.

> Fullscreen mode stretches XBMC across multiple displays

For a multi-monitor setup, XBMC may default to stretching across all
screens. One can restrict the fullscreen mode to one display by setting
the environment variable SDL_VIDEO_FULLSCREEN_HEAD to the number of the
desired target display. For example, having xbmc show up on display 0,
add the following line to the xbmc user's Bashrc:

    SDL_VIDEO_FULLSCREEN_HEAD=0

Note:Mouse cursor will be held inside screen with XBMC.

> Slowing down CD/DVD drive speed

The eject program from the util-linux package does a nice job for this,
but its setting is cleared as soon as the media is changed.

This udev-rule reduces the speed permanently:

    /etc/udev/rules.d/dvd-speed.rules

    KERNEL=="sr0", ACTION=="change", ENV{DISK_MEDIA_CHANGE}=="1", RUN+="/usr/bin/eject -x 2 /dev/sr0"

Replace sr0 with the device name of your optical drive. Replace -x 2
with -x 4 if you prefer 4x-speed instead of 2x-speed.

After creating the file, reload the udev rules with

    # udevadm control --reload

See also
--------

-   XBMC Wiki - Excellent resource with much information about Arch
    Linux specifically

Retrieved from
"https://wiki.archlinux.org/index.php?title=XBMC&oldid=296963"

Category:

-   Player

-   This page was last modified on 12 February 2014, at 07:36.
-   Content is available under GNU Free Documentation License 1.3 or
    later unless otherwise noted.
-   Privacy policy
-   About ArchWiki
-   Disclaimers