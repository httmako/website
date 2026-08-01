---
title: systemd services
toc: true
lang: en
description: Shows examples for secure systemd .service files
header-includes:
    <link rel="stylesheet" href="/style.css">
---


# Securing services with systemd

This blog documents my findings with making a linux application as secure as possible using systemd services.


## The problem

Imagine this scenario: You have a linux server and want to host an application that no other user has access to.  
You create a new user for this and start the application inside a "screen" or in the background using a shell script.

This enables multiple attack vectors (or configuration problems):

 - Maybe the user has SSH enabled, which makes it attackable from the outside
 - The user has a login shell, which makes its password attackable from within the server
 - If the application is insecure it could read any file from the file system via bugs
 - A hacker could have access to your linux user if the application is insecure (e.g. exploits from log4j, activemq, etc.)


## Solution via systemd services

By changing your application from a simple user-started `./myapp` into a systemd service you can use the security features enabled by systemd.  

Documentation for this:

 - [https://www.freedesktop.org/software/systemd/man/latest/systemd.resource-control.html#IPAccounting=](https://www.freedesktop.org/software/systemd/man/latest/systemd.resource-control.html#IPAccounting=)
 - [https://www.man7.org/linux/man-pages/man5/systemd.exec.5.html](https://www.man7.org/linux/man-pages/man5/systemd.exec.5.html)
 - [https://www.man7.org/linux/man-pages/man5/systemd.service.5.html](https://www.man7.org/linux/man-pages/man5/systemd.service.5.html)

An example, not-yet-secured systemd .service file (e.g. `/etc/systemd/system/restapi.service`):

```bash
[Unit]
Description=RestAPI
After=network.target iptables.service mariadb.service

[Service]
Type=simple
User=restapi
Group=restapi
WorkingDirectory=/opt/restapi
ExecStart=/opt/restapi/bin/rapi
Restart=always
RestartSec=1m

[Install]
WantedBy=multi-user.target
```

(After creating new service files you have to run `systemctl daemon-reload`)

This creates a service for an application named `RestAPI`.  
It automatically restarts it 1 minute after it crashed (if it crashes) and starts it after the network, iptables and mariadb services loaded.  
This application is started with the "restapi" user, which means it has no root privileges when running.

Running a service as a non-root user has many side effects, for example:

 - Missing capabilities, which means you can't bind to a port <1024 and you can't send raw network packets
 - Missing read/write permissions for specific folders (e.g. /var/log/)

Running the service as a user has many advantages, for example:

 - Higher security thanks to less privileges
 - Easier management of permissions
 - More security compliant thanks to less attack surface

SystemD, as mentioned in the documentation above, has many more security features than just user services.  
To analyze your systemd service files you can use the `systemd-analyze security` command. To analyze a single service file in detail you can use `systemd-analyze security <svcname>`.  
This is a fully enhanced systemd service file with comments to explain each feature:

```bash
[Unit]
Description=MyRestAPI
After=network.target mariadb.service
# Location of this file would be /etc/systemd/system/myrestapi.service (on redhat and debian)

[Service]
# Type simple is default, service is considered started after process has been started
Type=simple
# Set the user and group that runs this process, highly recommended
User=myuser
Group=myuser
# You could also generate a random user that starts this process.
# Warning: user-file ownership will probably be a problem if using this!
# DynamicUser=true

WorkingDirectory=/srv/myrestapi
# The starting command for this service, e.g. full path to an executable file
ExecStart=/srv/myrestapi/bin/myrestapi
# When to restart, always = if service stopped we restart, no matter the exit code. on-failure = only restart if exit code is non-zero
Restart=always
# Delay between restarts, 1m = 1minute
RestartSec=1m
# You could also set environment variables like this:
Environment=USER=x HOME=/home/x ES_HOME=/opt/es

# Hardening from now on

# Enable counting of CPU ms used
CPUAccounting=yes
# Limit the CPU usage to 50% of a SINGLE CORE, set it to e.g. 200% to only use 2 cores max. of your system
CPUQuota=50%
# Enable counting of IP bytes in and out for the service
IPAccounting=yes
# Only allow sockets, IPv4 and IPv6 as network protocols
RestrictAddressFamilies=AF_UNIX AF_INET AF_INET6
# The following 2 lines enable an IP whitelist for this service
# This allows this application to only communicate with the localhost, neither requests from nor to the internet are allowed
IPAddressDeny=any
IPAddressAllow=127.0.0.0/8
# You can also disable all network traffic with PrivateNetwork
# PrivateNetwork=true
# Service can no longer gain privileges via e.g. setuid or fs caps
NoNewPrivileges=true
# Creates new /tmp and /var/tmp directories for this process. After this systemd service stops these tmp folders will be removed!
PrivateTmp=true
# Filesystem namespacing, doesn't propagate mounts to this process
PrivateMounts=true
# Disables /dev/sda,mem,port,etc.
PrivateDevices=true
# Makes /usr,/boot,/efi,/etc read-only, but can still write to /tmp,/srv
# Set this to "strict" to make the whole filesystem read-only
ProtectSystem=full
# Hide /home,/root and /run/user (inaccessible and empty)
ProtectHome=true
# Make cgroups read-only (only container managers use write) and disallow namespace creation
ProtectControlGroups=true
RestrictNamespaces=true
# Prevents hostname and clock changes
ProtectHostname=true
ProtectClock=true
# Set umask of files created by service's application (u=rwx, go=nothing)
UMask=0077
# Remove IPC after service is stopped
RemoveIPC=true
# Protect kernel variables and modules
ProtectKernelTunables=true
ProtectKernelModules=true
# Only lets the process see itself in /proc/
ProcSubset=pid
ProtectProc=noaccess
# Disables setting SUID/SGID on files
RestrictSUIDSGID=true
# Remove all root capabilities (bind port < 1024, ignore file perms, etc.)
CapabilityBoundingSet=
# Files not owned by you appear to be owned by "nobody" (or root)
PrivateUsers=true
# Disallow access to syslog
ProtectKernelLogs=true
# Makes directories invisible for the process. Warning, some processes need those! (or use ReadOnlyPaths=)
InaccessiblePaths=/bin /boot /lib /lib64 /media /mnt /opt /root /sbin /usr /var
# Disable specific syscalls (setting clock, tracing, kernel-modules, mounting, rebooting, root-only, changing sawp, emulating other CPUs, obsolete calls)
SystemCallFilter=~@clock @debug @module @mount @reboot @privileged @swap @cpu-emulation @obsolete
# Disable changing kernel-exec-domain and realtime execution (could hog cpu)
LockPersonality=true
RestrictRealtime=true


# Redirect the stdout and stderror to a file
StandardOutput=append:/srv/myrestapi/stdout.log
StandardError=inherit

# Chroot's the process into the directory.
# Warning: This doesn't work well for non-static binaries because of missing /lib!
# RootDirectory=/srv/myrestapi

[Install]
WantedBy=multi-user.target
```

The above is an example of a systemd service file with nearly all of the security features activated.  
If you use the chroot (RootDirectory=) and User= settings above then your application is (nearly) as secure as possible:

 - No SSH or shell login possible, because of unreadable /bin
 - No network access to or from the outside, only to localhost
 - No ability to see any folder on the system thanks to chroot / unreadable folders

Even if someone hacks the application and can execute code, there is no easy way forward as the attacker can neither launch a program/shell (no /bin folder accessible) nor connect to other systems from your server (`IPAddressAllow=127.0.0.0/8`).

This chroot and dynamic-user setup mainly works for Golang REST-APIs only!  
Golang applications can be built as a static executable (which don't need the /lib folder) and a rest api doesn't need to access files (mainly needs port 80 http and port 3306 mysql).  
You can verify if your executable is static by using the `ldd` tool (e.g. `ldd myapp`).  
If your application is not static or needs connectivity to the outside world without a proxy then it is a bit more complex to setup.  
Example: An application needs the /etc/hosts file for DNS resolutions and the /lib folder for e.g. libc libraries. It would also need connectivity to the internet, which means configuring "IPAddressAllow=" will be nearly impossible.


# FAQ

## About chroot / RootDirectory=

The main problem with chroot: The `/etc/resolv.conf` file is not available. This makes DNS resolutions impossible, which means you have to enter `127.0.0.1` in your configs instead of `localhost`, because your application won't be able to resolve localhost to the IP address anymore.

If you need DNS then I recommend to use the `InaccessiblePaths=` configuration and not use chroot anymore.

Chroot is also going to create a few new folders in your target directory (`dev,etc,proc,root,sys,tmp,usr,var`). They are all empty, but systemd seems to create them automatically when using `RootDirectory=` and not clean them up after stopping the application.

I highly recommend to only use chroot if you have a very simple and static application.


## Troubleshooting exited processes

(All these commands require root)  

After creating your service file you have to run `systemctl daemon-reload` to load them initially.  
To start them use `systemctl start <myapp>`. To check their status (e.g. if they are running) you use `systemctl status <myapp>`.  
To see more of the status log output you can use `journalctl -xeu <myapp>`.  

If the process exited with e.g. status 2,203 or similar then check if your service file is correct and if your directories and ExecStart application actually exists.  
Check the stdout.log file (set by `StandardOutput=append:/srv/myrestapi/stdout.log`) to check for any application errors.  
Common mistakes are:

 - Wrong permissions of files (e.g. no read/write permission)
 - Unable to access IPs because of `IPAddressAllow=127.0.0.0/8`
 - Unable to resolve DNS (e.g. localhost->127.0.0.1) because of `RootDirectory=` (=chroot)

Logs are available by either accessing the stdout.log file or by using `journalctl -xe` to see the systemd stop reason.
