pkgsrc-darwin
=============
Additional support files to aid in running pkgsrc under Darwin.
This information has been tested under:

* OS X 10.10 (Yosemite)
* OS X 10.9 (Mavericks)
* OS X 10.8 (Mountain Lion)

## Bootstrapping

1. Install Xcode and start Xcode, accept license agreement, and allow it to install components. 
2. Fetch pkgsrc.tar.xz from ftp://ftp.netbsd.org/pub/pkgsrc/current/pkgsrc.tar.xz
3. Extract as root: ```tar -C /usr --xz -xf pkgsrc.tar.xz```

I prefer to keep as much as possible in the ```/usr/pkg``` tree. If you don't, you can disregard the ```--varbase``` flag and argument.

Bootstrap with:
```
    cd /usr/pkgsrc/bootstrap
    ./bootstrap --compiler=clang --abi=64 --pkgdbdir=/usr/pkg/pkgdb --varbase /usr/pkg/var
```

While bootstrapping:

*  Create and populate ```/etc/paths.d/pkgsrc```:
```
    /usr/pkg/sbin
    /usr/pkg/bin
```

* Create and populate ```/etc/manpaths.d/pkgsrc```:
```
    /usr/pkg/man
```

* [optional] Add to ```/usr/pkg/etc/mk.conf```:
```
# Don't forget to set 'MAKEOBJDIRPREFIX /usr/obj' in your environment with WRKOBJDIR
#WRKOBJDIR=/usr/obj/pkg
#PACKAGES=/usr/pkg/packages/${OPSYS}/${OS_VERSION}/${MACHINE_ARCH}
LOCALPATCHES=/usr/pkg/localpatches
PACKAGES=/usr/pkg/packages
DISTDIR=/usr/pkg/distfiles
RCD_SCRIPTS_DIR=/usr/pkg/etc/rc.d

PKG_DEFAULT_OPTIONS+=-x11
PKG_DEFAULT_OPTIONS+=-xcb

PREFER.openssl=pkgsrc
```

## Starting Apps at Boot

### Install startup framework files

* install ```pkgtools/rc.subr```
* install ```pkgtools/rcorder```
* ```mkdir /usr/pkg/var/run```

>NOTE: I moved ```/etc/rc.subr``` and ```/etc/rc.conf``` to ```/usr/pkg/etc``` and created symlinks back to /etc to keep everything in ```/usr/pkg/```.  Also symlinked ```/etc/rc.d -> /usr/pkg/etc/rc.d```

>**The examples in this document assume these files are in ```/usr/pkg/etc/```.**

* Patch ```rc.subr``` to use ```/bin/echo``` instead of shell built-in echo.
```
cd /usr/pkg/etc
patch -p0 /path_to_repo/patches/rc.subr.patch
```

* Install the scripts in this repo's ```etc``` directory to ```/usr/pkg/etc/```:
```
rc.pkgsrc
rc.d/DAEMON
rc.d/LOGIN
rc.d/NETWORKING
rc.d/SERVERS
rc.d/syslogd
rc.d/mountcritremote
```

>NOTE: The included ```rc.d/NETWORKING``` has been heavily modified from the original NetBSD version to wait for one or more network interfaces to have an IP address assigned to it (or 60 seconds elapses) before proceeding with the rc.pkgsrc startup. This was added to help ensure that the network interfaces were properly configured so that applications that bound to them would not have to be restarted after the interface was configured post application startup.

>**The command ```ipconfig waitall```, while claiming to provide this exact functionality, does NOT work as advertised. Additionally, the man page discourages use of ```ipconfig``` for anything other than testing/debugging indicating that it's deprecated.**

To take advantage of this functionality, add the following to ```rc.conf```:

```
NETWORKING_flags="en0 en6"
```

Replace *en0 en6* with the network interface names appropriate to your installation.

### Create a Launchd User Daemon.

To launch rc.pkgsrc at boot:

* Copy ```LaunchDaemons/org.pkgsrc.rc.plist``` to ```/Library/LaunchDaemons/```.

* Load it with:
```launchctl load /Library/LaunchDaemons/org.pkgsrc.rc.plist```

* Check ```/var/log/system.log``` to see if it ran correctly. If not, correct the problem and then unload with:
```launchctl unload /Library/LaunchDaemons/org.pkgsrc.rc.plist```
and re-issue the load command.

## Troubleshooting

On some systems with IPv6 addresses (I think that's the cause), the command ```pkg_admin fetch-pkg-vulnerabilities``` just hangs.  In these cases, use curl/wget to fetch the vulnerability list via cron around 3am daily:

```
curl -# -o /usr/pkg/pkgdb/pkg-vulnerabilities \
	http://ftp.NetBSD.org/pub/NetBSD/packages/vulns/pkg-vulnerabilities.gz
```

## Other Notes

* ```net/hesiod/Makefile```; add:

    ```LDFLAGS.Darwin+=        -lresolv```

* Since Mavericks, OS X doesn't come with a ```cvs``` binary. ```cvs``` is part of the ```devel/scmcvs``` package.

* ```net/xymon``` needs additional shared memory segments. Create 
  ```/etc/sysctl.conf``` and add the lines:

```
    kern.sysv.shmmax=16777216
    kern.sysv.shmmni=128
    kern.sysv.shmseg=32
    kern.sysv.shmall=4096
```

Then, either reboot or load them each by hand with ```sysctl -w```.
