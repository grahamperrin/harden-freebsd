# Harden FreeBSD 

![](images/harden-freebsd-logo.jpg)

FreeBSD officially defaults to [Permanently Insecure Mode](https://man.freebsd.org/cgi/man.cgi?securelevel). This script will duplicate almost all the hardening settings run by `/usr/libexec/bsdinstall/hardening` and more. Any directive can be set and re-set with a customizable `settings.ini` for administering, tuning your system, and easy to use across jails. 

Even though I am by no means a FreeBSD expert, each of the security settings was researched, assessed and chosen as a set of mitigations for maximizing threat reduction while minimizing restriction of system capability and availability. 



## Features

* Makes backups of `rc.conf`, `sysctl.conf`, and `login.conf` on first run
* Sets passwords to blowfish encryption
* Sets passwords to expire at 120 days
* Handy for jail administration
* Disables sendmail completely
* Removes `other` write permissions from key system files
* Allows only root for `cron` and `at`
* Primitive directive and flag verification catches simple errors
* Modularizable within other tools
* Automate any shell script
* System Logging to `/var/log/messages` and Script Logging to `/var/log/harden-freebsd.log`
* Pretty prints color output of script execution to console while running

## Requirements

* FreeBSD 13.2
* Python 3.9.16

## Installation

**WARNING: Once kernel level 1 is set by this script, you will not be able to modify these confs again with this script until it is set to -1 or 0 and rebooted!**

* Set `kernlevel = 0` if you want to test various setting groups with your applications and network
* Customize `settings.ini`  to whatever is needed, the script will change the directive to your flag
* Set permissions `chmod 750 harden-freebsd.py` to prevent shell injection from another account or process
* Set permissions `chmod 640 settings.ini` to prevent shell injection from another account or process
* No `settings.ini` section can be entirely commented out nor be completely empty

## Customization

#### Backups

The very first time the script is run it will make copies of `rc.conf`, `sysctl.conf`, and `login.conf` named `rc.conf.original` etc. If you've already done this yourself you may want to rename or move those files.

If you would like you can set `settings.ini` section `[SCRIPT]`option `first_run` to `True` with capital `T` to make new backups at any time after you've renamed the original backups or the script will overwrite them.

#### Verification

To err on the safe side, the script does primitive verification of the `rc.conf` and `sysctl.conf` flags and some directives it expects. If may error on some abnormal directives which will cause it to put in place the backup *.original files it made. Check the log for what setting caused the validation failure and rewrite the regular expression or make a new check.

* Conformance of `hostname` will check that it does not start with a number. Although it is doable in FreeBSD it can cause trouble in some applications and networking instances 
  * If you must have a hostname starting with a number simply remove the check
* Conformance in use of capital letters in `rc.conf` where it is expected 

If you do get stuck in read-only single-user mode and need to correct a configuration file then use:

```sh
zfs set readonly=false zroot
zfs mount -a
```

#### Chmod-ability

The set of files needed to be secure changed and changed throughout testing and so it ended up as a shell command but an error checked function was provided for the administrator programmer to use instead of appending to the long list in `settings.ini` section `[FILESEC]` if you wish or to work with other software.

Those files are:

`etc/ftpusers /etc/group /etc/hosts /etc/hosts.allow /etc/hosts.equiv /etc/hosts.lpd /etc/inetd.conf /etc/login.access /etc/login.conf /etc/newsyslog.conf /etc/rc.conf /etc/ssh/sshd_config /etc/sysctl.conf etc/syslog.conf /etc/ttys /etc/crontab /usr/bin/crontab /usr/bin/at /usr/bin/atq /usr/bin/atrm /usr/bin/batch /var/log`

#### Secure Password Settings

The newly applied settings will not take affect until you reset your password.

## Automatic Jail Lockdown/Management

1. Copy software to `/root` and have jail start this script at reboot and all settings will be updated upon next reboot. To update all jails simply copy `settings.ini` with your own script to all appropriate locations for intake. 

`crontab -e`

`@reboot harden-freebsd.py`

2. Have all jails pointing to the same rc script via `exec.start` and set paths in the script pointing to the same location modified by the script paths. 
3. Add new jail specific entires to `settings.ini [SYSTEM]` section for sysctl.conf udpate
   - `security.jail.* = 0`
4. Use mutiple copies of the script and settings.ini for each jail
5. Put it in your template

## Setting Descriptors
**Startup**
* kern_securelevel_enable = "YES"
    * Enable access to other than permanently insecure modes
* Disable Sendmail
* syslogd_flags="-ss"
    * Disallow syslogd to bind to a network socket
* clear_tmp_enable = "YES"
    * Clear the /tmp directory on reboot
* icmp_drop_redirect="YES"
    * Disallow redirection of ICMP (ping, echo)
* inetd_enable = "NO"
    * Disallow Network File System to share directories over the network
* portmap_enable = "NO"
    * Disallow portmapping since Network File Systems is disallowed
* update_motd = "NO"
    * Disallow computer system details from being added to /etc/motd on system reboot
**System**
* kern.securelevel = 1 [(*)](https://man.freebsd.org/cgi/man.cgi?securelevel)
    * The system immutable and system append-only flags may
	   not be turned off; disks for	mounted	file systems, /dev/mem and
	   /dev/kmem may not be	opened for writing; /dev/io (if	your platform
	   has it) may not be opened at	all; kernel modules (see kld(4)) may
	   not be loaded or unloaded.  The kernel debugger may not be entered
	   using the debug.kdb.enter sysctl.  A	panic or trap cannot be	forced
	   using the debug.kdb.panic, debug.kdb.panic_str and other sysctl's.
* security.bsd.see_other_uids = 0
    * Disallow users from seeing information about processes that are being run by another user (UID)
* security.bsd.see_other_gids = 0 [(*)](https://docs.freebsd.org/en/books/handbook/mac/#mac-policies)
    * Disallow users from seeing information about processes that are being run by another group (GID)
* security.bsd.see_jail_proc = 0 (Sysctl MIB Entry `sysctl -a | grep security.bsd`)
    * Disallow non-root users from seeing processes in jail
* security.bsd.unprivileged_read_msgbuf = 0 (Sysctl MIB Entry `sysctl -a | grep security.bsd`)
    * Disallow non-root users from reading system message buffer
* kern.randompid = 107 [(*)](https://wiki.freebsd.org/DevSummit/201308/Security)
    * Force kernel to randomize process ID's using above salt value instead of seqential
* net.inet.ip.random_id = 1
    * Randomize IP packet ID
* net.inet.ip.redirect = 0 
    * Disallow ICMP host redirects
* net.inet.tcp.always_keepalive = 0
    * Disallow keeping open idle TCP connections
* net.inet.tcp.blackhole = 2 +(UDP)[(*)](https://man.freebsd.org/cgi/man.cgi?query=blackhole)
    * Packets that are received on a closed port will not initiate a reply
* net.inet.tcp.path_mtu_discovery = 0 [(*)](https://man.freebsd.org/cgi/man.cgi?query=tcp&sektion=4)
    * Disallows TCP to determine the minimum MTU size on any network that is currently in the path between two hosts
* net.inet.icmp.drop_redirect = 1
    * Pairs with rc.conf startup, as once enabled, it is then set
* hw.mds_disable = 1 [(*)](https://www.kernel.org/doc./html/latest/arch/x86/mds.html)
    * Enable Microarchitectural Data Sampling Mitigation version `VERW`
    * Change value to `3` (AUTO) if using a Hypervisor without MDS Patch
* hw.ibrs_disable = 1 [(*)](https://wiki.freebsd.org/SpeculativeExecutionVulnerabilities)
    * Prevent Spectre and Meltdown CPU Vulnerabilities
* kern.elf64.aslr.stack = 3 [(*)](https://wiki.freebsd.org/AddressSpaceLayoutRandomization)
    * Address space layout randomization is used to increase the difficulty of performing a buffer overflow attack
    * For 32bit CPU or VM Architecture switch directive to `kern.elf32.aslr.stack`


## License Summary

Non-Commercial usage, retain and forward author and license data. Modify existing code as needed up to 25% while allowing unlimited new additions. The Software may use or be used by other software.

## Security Guidelines

Since this Software uses shell commands it is required to place it in a secure directory with permissions on the **parent** directory to have no permissions for `other` /all/world group to *execute* and **no network access**. 

Please follow [these guidelines](/docs/SECURITY.md) should you find a vulnerability not addressed in the audit.

## Statement of Security: 

* **Risk** - Low
* **Impact** - Medium

This script has no networking, accesses no sockets, and uses only standard libraries.

Although this script is using `subprocess.run(shell=True)` the only possibility of shell injection is from the paths customized by the Licensee or unauthorized access to the filesystem the script resides on in order to perform unauthorized modifications to `settings.ini`or the Software which is not a vulnerability of the Software. 

### Latest Development Version

[Quadhelion Engineering Code Repository](https://got.quadhelion.engineering)



![quadhelion engineering logo](images/quadhelionEngineeringIcon.jpg)