pkgsrc-darwin
=============
This repository contains documentation, support files, and troubleshooting tips for using the [pkgsrc](http://pkgsrc.org) package management system under OS X/macOS version 10.8 and later.

## Using Binary Packages

The `pkgin` tool provides a `yum/dnf/apt`-like interface for managing packages downloaded from one or more binary package repositories. Check out the excellent [`pkgin` bootstrapping instructions](https://pkgsrc.joyent.com/install-on-osx/) from Joyent.

## Compiling Packages From Source

### Note regarding System Integrity Protection (SIP)

With the addition of System Integrity Protection in OS X 10.11, the ability to modify most everything under ```/usr``` is now restricted, even as root, with the exception of ```/usr/local```. These instructions and support files have been modified to use the prefix of ```/opt``` instead of the more lengthy ```/usr/local```.

### Bootstrapping

1. Install the Command Line Tools by runnning ```xcode-select --install``` from a terminal window. A GUI window will appear asking for confirmation to install the command line tools.
The tools can also be downloaded manually; see [Apple Techical Note TN2339](https://developer.apple.com/library/content/technotes/tn2339/_index.html).
2. Fetch `pkgsrc.tar.xz` from `ftp://ftp.netbsd.org/pub/pkgsrc/current/pkgsrc.tar.xz`
3. Extract as root: ```tar -C /opt --xz -xf pkgsrc.tar.xz```

I prefer to keep as much as possible in the ```/opt/pkg``` tree. If you don't, you can disregard the ```--varbase``` flag and argument.

Bootstrap with:
```
cd /opt/pkgsrc/bootstrap
./bootstrap --compiler clang --abi 64 --prefix /opt/pkg --pkgdbdir /opt/pkg/.pkgdb --varbase /opt/pkg/var
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
#PACKAGES=/opt/pkg/packages/${OPSYS}/${MACHINE_ARCH}/${OS_VERSION}
LOCALPATCHES=/opt/pkg/localpatches
PACKAGES=/opt/pkg/packages
DISTDIR=/opt/pkg/distfiles
RCD_SCRIPTS_DIR=/opt/pkg/etc/rc.d

PKG_DEFAULT_OPTIONS+=-x11
PKG_DEFAULT_OPTIONS+=-xcb

PREFER.openssl=pkgsrc
```

### Starting Apps Automatically at Boot

#### Install startup framework files:

* install ```pkgtools/rc.d-boot```
* ```mkdir /opt/pkg/var/run```

>NOTE: I moved ```/etc/rc.subr``` and ```/etc/rc.conf``` to ```/opt/pkg/etc``` and created symlinks back to `/etc` to keep everything in ```/opt/pkg/```. Also symlinked ```/etc/rc.d -> /opt/pkg/etc/rc.d```

## Troubleshooting

### BigSur/XCode 12 - version:1:1: error: expected unqualified-id

According to [this](https://trac.macports.org/ticket/62784) posting, XCode 12 contains a "C++20-capable compiler and the VERSION file included in the source is conflicting with the version header of your compiler because you are on a case-insensitive filesystem."

One work-around is to create an APFS container, formatted case-sensitive, and mount that container on `/opt` by adding the UUID of the container in `/etc/fstab`:

	UUID=A6BBE2B5-7395-49A5-94AB-3934323D50285 /opt apfs rw,auto

The UUID can be obtained using `diskutil info {disk|partition|volume|container}`. `Diskarbitrationd(8)` will automatically mount the case-sensitive volume at boot.

Be sure to copy any existing data out of `/opt` before you reboot or manually mount the new filesystem over `/opt`.

### Xcode 8 - ERROR: This package has set PKG_FAIL_REASON:<br>ERROR: No suitable Xcode SDK or Command Line Tools installed.

First, check that you've installed the command line tools (step #1 in the Bootstrapping section above).

With Xcode 8 it may be due to the following:

* http://mail-index.netbsd.org/pkgsrc-users/2016/09/16/msg023742.html

Due to the way pkgsrc checks OS and SDK versions, and changes Apple made in Xcode 8, building with Xcode 8 on non-macOS Sierra machines will fail.

Until this is worked around in pkgsrc, you might:

* Install macOS Sierra.
* Revert to Xcode 7 (Download from https://developer.apple.com/download/more/)
* Wait patiently (fix probably not going to be in 2016Q3 release)

### **NOT** Xcode 8 - ERROR: This package has set PKG_FAIL_REASON:<br>ERROR: No suitable Xcode SDK or Command Line Tools installed.

This has cropped up again with macOS 10.14 and Xcode 11. 

* ```xcode-select --install``` says the command line tools are already installed; checking ```/Library/Developer/CommandLineTools``` seems to confirm this.  
* ```xcode-select -p``` returns ```/Library/Developer/CommandLineTools```; looks good.
* Removing ```/Library/Developer/CommandLineTools``` and re-installing doesn't fix the issue.

**_FIX_** for me was to remove ```/Applications/Xcode.app``` and ```/Library/Developer/CommandLineTools``` and re-install the Command Line Tools with ```xcode-select --install```, which is fine since I don't use the Xcode app anyway. No idea what's going on here. Possibly some bogosity with Xcode 11 and non-Catalina macOS.

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

* When running OS X Server, several common web ports (80, 443, 8008, 8800, and 8443) are taken by the "service proxy". Since it was in my way I disabled it by unloading the following LaunchDaemon services: 

```
launchctl unload /Applications/Server.app/Contents/ServerRoot/System/Library/LaunchDaemons/com.apple.serviceproxy.plist
launchctl unload /Applications/Server.app/Contents/ServerRoot/System/Library/LaunchDaemons/com.apple.service.ACSServer.plist
```

* Since 10.15 Catalina, the default shell has been changed form Bash (3.x) to Zsh. You can install ```shells/bash``` and then ```chsh -s /opt/pkg/bin/bash``` to restore bash as your default shell _(assuming if you haven't already been running bash from pkgsrc to get all the version 4.x and later goodies like ```shells/bash-completion```.)_ **You will need to logout and back in for the shell change to take effect.**

* Since 10.14 Mavericks, the CVS client is no longer part of the base OS. Install the ```devel/scmcvs``` package.

* Since 10.13 High Sierra, telnet and ftp clients are no longer part of the base OS. Replacements can be found in the following packages:

    * ```net/inetutils``` (GNU Inetutils) provides telnet and ftp (among others). The binaries are g-prefixed (gtelnet, gftp). For convenience, I created symlinks from ```/usr/local/bin/{telnet,ftp} -> /opt/pkg/bin/g{telnet,ftp}```.
    * ```net/tnftp``` (portable version of the NetBSD FTP client)


* ```net/hesiod/Makefile```; add the following:
	```LDFLAGS.Darwin+=        -lresolv```

* ```net/xymon``` needs additional shared memory segments. Create ```/etc/sysctl.conf``` and add the lines:

```
kern.sysv.shmmax=16777216
kern.sysv.shmmni=128
kern.sysv.shmseg=32
kern.sysv.shmall=4096
```

Then, either reboot or load them each by hand with ```sysctl -w```.
