---
title: selinux
toc: true
lang: en
description: Shows common selinux commands and troubleshoot tips
header-includes:
    <link rel="stylesheet" href="/style.css">
---


# SELinux quick intro

This is a blog article and neither official nor verified by redhat.  
This site documents 

Main information source:  
 - [https://access.redhat.com/documentation/en-us/red_hat_enterprise_linux/7/html/selinux_users_and_administrators_guide/sect-security-enhanced_linux-troubleshooting-fixing_problems](https://access.redhat.com/documentation/en-us/red_hat_enterprise_linux/7/html/selinux_users_and_administrators_guide/sect-security-enhanced_linux-troubleshooting-fixing_problems)  
 - [https://access.redhat.com/documentation/en-us/red_hat_enterprise_linux/7/html/selinux_users_and_administrators_guide/chap-managing_confined_services-the_apache_http_server](https://access.redhat.com/documentation/en-us/red_hat_enterprise_linux/7/html/selinux_users_and_administrators_guide/chap-managing_confined_services-the_apache_http_server)


# Warning and Enabling selinux

**Never disable SELinux!**  
It is always better to just set it to "Permisive", which means "not actively blocking, only logging". (By default SELinux is set to "Enforcing", which means it blocks illegal accesses)

To verify if SELinux runs you can use the `getenforce` or `sestatus` command. The `getenforce` command will return the current mode of SELinux:

 - `Disabled` , SELinux is completely disabled and no features work  
 - `Permissive` , SELinux is active but doesn't do anything, it only logs if it would've blocked a call  
 - `Enforcing` , SELinux is active and working, it blocks illegal calls/accesses  

To disable SELinux temporary use the `setenforce 0` command.  
To disable it permanently edit the `/etc/selinux/config` file and set the state to `SELINUX=permissive`.

To enable it (as in, actively blocking illegal accesses) temporarily use `setenforce 1`.  
To enable it permanently set `SELINUX=enforcing` in the `/etc/selinux/config`.

The `setenforce` command only set the state until next reboot. You can check this with the `sestatus` command.


## Enable selinux from a disabled state

1. Check if `getenforce` does NOT return `Disabled`. If it says Permissive/Enforcing then it is active, else continue.
2. Check if the setting in `/etc/selinux/config` does NOT say `Disabled`. If it does, change it to `Permissive`.
3. Check if there is a grub boot config that disables it (`grep -rin "selinux=0" /boot`), if there is use the command in the `/etc/selinux/config` file to enable it again
4. Check if a file in `/etc/audit/rules.d/` does NOT exclude selinux avc records. If a line like `-a always,exclude -F msgtype=AVC` exists you HAVE TO COMMENT IT OUT by placing a `#` infront of the line and rebooting the server afterwards.

If it was Disabled then you have to reboot to enable it again. Before that create the autorelabel file (`touch /.autorelabel`) and then restart the server. The boot may take a bit longer because SELinux applies the context to the filesystem again.

After booting the mount options will have an additional one named `seclabel`, if you have strict monitorint (e.g. checkmk) then update the setting there to include this.

If you have custom software installed (e.g. a custom .rpm package that installed into /opt or /srv) then you maybe have to reinstall that software for it to work correctly.  
During software installations the SELinux labels and context will be set for the installed files, if selinux is disabled during the installation then these labels won't be set correctly and the application maybe will not run after enabling selinux again afterwards.



# What is selinux

SELinux stands for "Security Enhanced Linux" and is basically a list of rules and definitions that secure your server and filesystem accesses.  
For example, there exist boolean configs for disabling certain httpd server features like 

```bash
semanage boolean -l | grep httpd_can_network_connect_db
# httpd_can_network_connect_db   (on   ,   on)  Allow httpd to can network connect db
```

and there are also special attributes on files that show what exactly they are for

```bash
ls -Zahl /var/www/html/index.php
# -rw-r--r--. 1 root root **unconfined_u:object_r:httpd_sys_content_t:s0** 7.1K Feb 26 17:40 /var/www/html/index.php
```

There are also contexts on users and processes. These can be viewed with the `-Z` flag, for example `ps -Zex` or `ls -Z` or `id -Z`.

According to the documentation, the following daemons are protected by the default targeted policy: dhcpd,httpd,named,nscd,ntpd,portmap,snmpd,squid,syslogd.



# Commands and Usage

To see the state of selinux you can use `getenforce` or `sestatus`.  
To enable or disable (`Enforcing` or `Permissive`) selinux you can use the `setenforce` command with either a 0 (Permissive) or 1 (Enforcing) behind it.  
To set the selinux state on boot (=permanent) edit the `/etc/selinux/config` file.


## Logs

The logs for selinux are stored in `/var/log/audit/audit.log`.  
To search for violations you can use `ausearch -m avc`.  
To get a summary of violations you can use `aureport -a`.  
The main type of selinux log you want is called `AVC`. This is the type that reports failed access because of selinux enforcing policies.


## Booleans

To list all booleans (=configs of selinux) you can use `semanage boolean -l`. The first value of the output is the current state, the second one is the default state. To install the semanage command (if missing) install `policycoreutils-python` for RHEL7 or `policycoreutils-python-utils` for RHEL8/9.  
Another command for this is `getsebool -a`.  
To set a bool you use `setsebool httpd_can_network_connect_db on` (or off). To make it permanent between reboots use the `-P` flag, example: `setsebool -P httpd_can_network_connect_db on`.


## List contexts

To view contexts for files or processes you can use the `-Z` flag.
```bash
ps -Ze | grep httpd
# system_u:system_r:httpd_t:s0      19177 ?        00:02:41 httpd
ls -Z /var/www/html/index.php
# unconfined_u:object_r:httpd_sys_content_t:s0 index.php
```

If you move a file with `mv` it moves the selinux context with it. (=keep context)  
If you copy a file with `cp` the file will have the context of the target folder. (=refresh context)  
If you want to copy a file with the same context use the `cp --preserve=context` flag.


## Set contexts

To temporary change the type context of a file (until a relabel) use `chcon`:
```bash
chcon -t bin_t /bin/myapp
```

To permanently change it use the `semanage fcontext` command:
```bash
semanage fcontext -a -t bin_t /bin/myapp
restorecon -v /bin/myapp
```

To view more help with the semanage command you can use `man semanage-fcontext`.



# Troubleshooting

## Verifying selinux problem

To make sure that your problem occurs because of SELinux:  

 - If `getenforce` returns `Enforcing` then disable it with `setenforce 0` and check if the problem persists. If the problem is now gone then it is an SELinux problem. If the problem still occurs then it is NOT an SELinux problem.  
 - If `getenforce` returns `Permissive` or `Disabled` then the problem is not because of SELinux.


## Exec - permission denied

Example error: `exec /opt/bin/file - Permission denied`.  
This is a common error for systemd services that use custom applications, e.g. software installed in /srv or a /home directory.  
This problem occurs because the binary file (in this case the file named `/opt/bin/file`) has a type that is not `bin_t`.

Check if this problem is the same as yours:  

 - `ls -ahl /opt/bin/file` shows that the user that wants to start the executable has read&execute permissions (e.g. `-rwxr-xr-x`)  
 - `ls -Z /opt/bin/file` shows the type as not `bin_t` (e.g. `system_u:object_r:unconfined_t:s0`)

If the selinux type of the file is not bin_t then you can use the following command to set it temporary:

```bash
chcon -t bin_t /opt/bin/file
```

Try to restart the application and check if it works with the new context.
  
If it works then you can now set the new context permanently.  
First you set a custom context with the `semanage fcontext` command (-a adds a new entry) and then load this context with the `restorecon` command.  
This way of setting the context makes it survive a filesystem relabel.  
WARNING: You have to use the full path with the semanage command!

```bash
# Create a new context for the file
semanage fcontext -a -t bin_t /opt/bin/file
# Load the new context for the file
restorecon -v /opt/bin/file
# Output:
# Relabeled /opt/bin/file from unconfined_u:object_r:user_home_t:s0 to unconfined_u:object_r:bin_t:s0
```

To see all your custom contexts you can use the `semanage fcontext -C -l` command.

```bash
semanage fcontext -C -l
# SELinux fcontext                                   type               Context
#
# /opt/bin/file                                      all files          system_u:object_r:bin_t:s0
```


## HTTPD errors

By default the standard apache2 webserver, called httpd, is not allowed to do much.  
This is the cause of many errors, and there are many fixes for all the small problems.

**database fails to connect**  
```bash
setsebool -P httpd_can_network_connect_db on
```

**connection to remote host fails**  
```bash
setsebool -P httpd_can_network_connect on
```

**httpd reverse proxy not working**  
```bash
setsebool -P httpd_can_network_relay on
```

**php is not working**  
```bash
setsebool -P httpd_builtin_scripting on
```

**forbidden / can't read file**  
Make sure the folder and files of your web directory (e.g. `/var/www/html`) have the correct selinux context.  
To set it to a context that the httpd webserver can read:
```bash
chcon -R -t httpd_sys_content_t /var/www/html
```
(This type is the default of the web folder if you use restorecon)

**can't write file**  
To let the apache user write to a file you have to set a different context for your web folder (e.g. `/var/www/html`) like this:
```bash
chcon -R -t httpd_sys_rw_content_t /var/www/html
```
Then enable the service to write to files with:
```bash
setsebool -P httpd_anon_write on
setsebool -P httpd_sys_script_anon_write on
```
Also make sure that the folder is owned by the user `apache` with:
```bash
chown -R apache:apache /var/www/html
```




## Common troubleshooting tips

The command `ausearch -m avc` or `aureport -a` shows you violations where selinux blocked access to something.  
With these commands you can check what process tried to access/write to what file/function.  
Make sure that the process that runs (`ps -Ze | grep myprocess`) has the correct permission to access the ressource (`ls -Zahl /opt/myapp`).

You could also use the `audit2why` command to get a nice description of why an access was denied like this:

```bash
audit2why < /var/log/audit/audit.log
```

For application specific troubleshooting you can check the second link at the top of this page for a redhat documentation about specific services and their booleans / contexts.
