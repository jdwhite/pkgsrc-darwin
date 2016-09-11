pkgsrc-darwin
=============
Additional support files to aid in running pkgsrc under Darwin.
This information has been tested under:

* OS X 10.11 (El Capitan)
* OS X 10.10 (Yosemite)
* OS X 10.9 (Mavericks)
* OS X 10.8 (Mountain Lion)

## Notes for OS X 10.11 (El Capitan)

* With the addition of System Integrity Protection (SIP) in 10.11, the creation of files/directories under ```/usr``` is now restricted. These instructions and support files have been modified to use the prefix of ```/opt``` instead of the more lengthy ```/usr/local```.
* When running OS X Server, several common web ports (80, 443, 8008, 8800, and 8443) are taken by the "service proxy". I'm not sure what the purpose of this is, but since it was in my way I disabled it by unloading the following LaunchDaemon services: 
```
	launchctl unload /Applications/Server.app/Contents/ServerRoot/System/Library/LaunchDaemons/com.apple.serviceproxy.plist
    launchctl unload /Applications/Server.app/Contents/ServerRoot/System/Library/LaunchDaemons/com.apple.service.ACSServer.plist
```

## Bootstrapping

1. Install Xcode and start Xcode, accept license agreement, and allow it to install components. 
2. Fetch pkgsrc.tar.xz from ftp://ftp.netbsd.org/pub/pkgsrc/current/pkgsrc.tar.xz
3. Extract as root: ```tar -C /opt --xz -xf pkgsrc.tar.xz```

I prefer to keep as much as possible in the ```/opt/pkg``` tree. If you don't, you can disregard the ```--varbase``` flag and argument.

Bootstrap with:
```
    cd /opt/pkgsrc/bootstrap
    ./bootstrap --compiler clang --abi 64 --prefix /opt/pkg --pkgdbdir /opt/pkg/pkgdb --varbase /opt/pkg/var
```

While bootstrapping:

*  Create and populate ```/etc/paths.d/pkgsrc```:
```
    /opt/pkg/sbin
    /opt/pkg/bin
```

* Create and populate ```/etc/manpaths.d/pkgsrc```:
```
    /opt/pkg/man
```

* [optional] Add to ```/opt/pkg/etc/mk.conf```:
```
PKGSRCDIR=/opt/pkgsrc

# Don't forget to set 'MAKEOBJDIRPREFIX /opt/obj' in your environment with WRKOBJDIR
#WRKOBJDIR=/opt/obj/pkg
#PACKAGES=/opt/pkg/packages/${OPSYS}/${OS_VERSION}/${MACHINE_ARCH}
LOCALPATCHES=/opt/pkg/localpatches
PACKAGES=/opt/pkg/packages
DISTDIR=/opt/pkg/distfiles
RCD_SCRIPTS_DIR=/opt/pkg/etc/rc.d

PKG_DEFAULT_OPTIONS+=-x11
PKG_DEFAULT_OPTIONS+=-xcb

PREFER.openssl=pkgsrc
```

## Starting Apps at Boot

### Install startup framework files

* install ```pkgtools/rc.subr``` (rc.subr-20150510 or later)
* install ```pkgtools/rcorder```
* ```mkdir /opt/pkg/var/run```

>NOTE: I moved ```/etc/rc.subr``` and ```/etc/rc.conf``` to ```/opt/pkg/etc``` and created symlinks back to /etc to keep everything in ```/opt/pkg/```.  Also symlinked ```/etc/rc.d -> /opt/pkg/etc/rc.d```

>**The examples in this document assume these files are in ```/opt/pkg/etc/```.**

* Install the scripts in this repo's ```etc``` directory to ```/opt/pkg/etc/```:
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
    curl -# -o /opt/pkg/pkgdb/pkg-vulnerabilities \
	    http://ftp.NetBSD.org/pub/NetBSD/packages/vulns/pkg-vulnerabilities.gz
```

### Large delay before graphs render when using rrdtool.

**Note: This isn't an issue with pkgsrc or even unique to Darwin, but since I've observed it on Darwin and it took a while to figure out the cause I'm documenting it here in the hopes it will save others valuable time.**

The delay in graph rendering is caused by the fontconfig library enumerating all the fonts on the system. Typically this only happens once and the font list is stored in a cache.  A persistant delay indicates that the process using the fonconfig library doesn't have permission to create the font cache and so the fontconfig library must enumerate the the fonts every time it's called.

In the author's case, the process using the fontconfig library was httpd running as user _www, which does not have a writable home directory and, thus, was trying to create a ```~/.cache/fontconfig``` directory in ```/``` where it did not have write access.

The author chose to create ```/.cache/``` with the following permissions and ownership:
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

* Since Mavericks, ```cvs``` is no longer part of the base OS X installation and must be installed form pkgsrc. ```cvs``` is part of the ```devel/scmcvs``` package.

* ```net/xymon``` needs additional shared memory segments. Create ```/etc/sysctl.conf``` and add the lines:

```
    kern.sysv.shmmax=16777216
    kern.sysv.shmmni=128
    kern.sysv.shmseg=32
    kern.sysv.shmall=4096
```

Then, either reboot or load them each by hand with ```sysctl -w```.
