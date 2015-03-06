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
    * This issue has been submitted to the NetBSD gnats bug database as [problem report #49724](http://gnats.netbsd.org/cgi-bin/query-pr-single.pl?number=49724).


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

### Command ```pkg_admin fetch-pkg-vulnerabilities``` hangs indefinitely.

On some systems, possibly those with an IPv6 addresses (probably a red herring), the ```pkg_admin fetch-pkg-vulnerabilities``` hangs.  In these cases, use curl/wget to fetch the vulnerability list via cron around 3am daily:

```
    curl -# -o /usr/pkg/pkgdb/pkg-vulnerabilities \
	    http://ftp.NetBSD.org/pub/NetBSD/packages/vulns/pkg-vulnerabilities.gz
```

### Large delay before graphs render when using rrdtool.

**Note: This isn't an issue with pkgsrc or even unique to Darwin, but since I've observed it on Darwin and it took a while to figure out the cause I'm documenting it here in the hopes it will save others valuable time.**

The delay in graph rendering is caused by the fontconfig library enumerating all the fonts on the system. Typically this only happens once and the font list is stored in a cache.  A persistant delay indicates that the process using the fonconfig library doesn't have permission to create the font cache and so the fontconfig library must enumerate the the fonts every time it's called.

In the author's case, the process using the fontconfig library was httpd running as user _www, which does not have a writable home directory and, thus, was trying to create a ```~/.cache/fontconfig``` directory in ```/``` where it did not have write access.
: The author chose to create ```/.cache/``` with the following permissions and ownership:
```
    drwxrwxrwx  3 root  wheel  102 Feb 16 11:51 /.cache
```
On the next call to the fontconfig library user _www was able to create and populate the cache directory.
```
    /.cache/fontconfig:
    total 1400
    -rw-r--r--  1 _www  wheel     144 Feb 16 11:52 2643cf6afbfb317cb17c1d21dc458165-x86_64.cache-4
    -rw-r--r--  1 _www  wheel  977944 Feb 16 11:52 84c0f976e30e948e99073af70f4ae876-x86_64.cache-4
    -rw-r--r--  1 _www  wheel     200 Feb 16 11:51 CACHEDIR.TAG
    -rw-r--r--  1 _www  wheel  400608 Feb 16 11:51 b0a71e6bf6a8a1a908413a823d76e21f-x86_64.cache-4
    -rw-r--r--  1 _www  wheel   43856 Feb 16 11:51 c94fc4589f4e4f179bb7abc5ef634560-x86_64.cache-4
```

Thereafter the graphs were rendered without (significant) delay.

## Other Notes

* ```net/hesiod/Makefile```; add:

    ```LDFLAGS.Darwin+=        -lresolv```

* Since Mavericks, OS X doesn't come with a ```cvs``` binary. ```cvs``` is part of the ```devel/scmcvs``` package.

* ```net/xymon``` needs additional shared memory segments. Create ```/etc/sysctl.conf``` and add the lines:

```
    kern.sysv.shmmax=16777216
    kern.sysv.shmmni=128
    kern.sysv.shmseg=32
    kern.sysv.shmall=4096
```

Then, either reboot or load them each by hand with ```sysctl -w```.
