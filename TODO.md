## TODO

* An installer script.

* Add IPv6 support to ```rc.d/NETWORKING``` script.

* A version of NetBSD's ```/etc/daily``` to take care of actions such as
    * ```pkg_admin fetch-vulnerability-list```
    * ```pkg_admin audit``` and mail the output to the admin

* Investigate using ```pkgtools/rc.d-boot```. The ```rc.d-boot``` script does not seem to include the setting of certain vars such as ```autoboot``` and ```rc_fast``` before running rc.d scripts, which have been needed for compatability. Perhaps a pull request is in order.

