# Ubuntu-OpenFilesLimit
How to increase the open files limit on Ubuntu?

Increasing file limits might be helpful if you’re running a web server which handles a lot of concurrent connections, because every connection opens a socket and socket is a file.

In my case, I needed to increase the open file limit from 1024 to 65536 to make sure that server won’t drop connections in the future. At first, I added this line to /etc/security/limits.conf:
```
* - nofile 65536
```

Where the first column is the user of the process, in our case wildcard * which makes this a default setting for all processes, the second column specifies the type: soft, hard or - (both). The third column is what should be limited exactly (open files, memory, etc) and the last one is the limit value (in our case the number of open files). The difference between hard and soft limits is that the former can be increased only by a rootuser. Soft limits can be increased by non-root users, though you can’t set the limit to a value higher than the hard limit. The limits from the file are applied once a new login session is started.

Unfortunately, after reboot I still had the old 1024 limit. I was testing both limits with those 2 commands:
```
ulimit -Sn # for soft limit
ulimit -Hn # for hard limit
```

I opened the config file again and found a comment:
```
# NOTE: group and wildcard limits are not applied to root.
# To apply a limit to the root user, <domain> must be
# the literal username root.
```

I’m running a service as a root (yeah, I know I shouldn’t do that, but anyway) and it turns out that in the first column a root username should be specified. OK, the final string looks like this:
```
root - nofile 65536
```

This indeed worked or at least I thought it did. I tested the limits with ulimit and it reported 65536.

After some time, the service I’m working on started to drop connections. The interesting thing was that it worked fine for the first 1K connections and after it reached 1022 connections, it just stopped accepting new ones. After a wasted day of debugging, trying various tricks from profiling and debugging golang codebase to tuning Linux kernel TCP stack, I’ve finally found the problem. I spent a day looking everywhere except of the open file limit because I thought that the limit was high enough. Turns out that systemd ignores the values from the /etc/security/limits.conf. I was also getting the wrong limit value because ulimit displays the limits for the current shell process and its children (not the global value), while the service was started as a daemon by a systemd process. In order to check the limits for a specific process you need to use this command:
```
prlimit -p process_id
```
```
# or looking up a per process limits file
cat /proc/process_id/limits
```

So, after I queried the limits of the service process, it turned out that it had a hard limit of 4096 and a soft limit of 1024. I added this line to a /etc/systemd/system.conf file:
```
DefaultLimitNOFILE=65536
```

…and it worked! I wonder why do we need 2 files to configure the same thing?
Summary

There are 2 scopes of limits: system-wide and per-process.

When a user logs in, their shell process gets the limit configuration from the /etc/security/limits.conf. Limits applied to a process are inherited by its children. You can get or set the limits of the process at runtime (new children of the process will inherit the new limits) with either prlimit command or with a ulimitshell builtin (applies to shell process and its children only):
```
# get limits for specific process
prlimit -p process_id

# get limits for current shell
ulimit -a
```

If your system uses systemd and you want to configure the limits for a process that is going to be used as a daemon you need to configure the limits in the /etc/systemd/system.conf file, you can look up the syntax here (at the bottom of the page)

In order to configure a system-wide limit, you need to edit the /etc/sysctl.conf file. For example, if you want to limit the maximum number of open files at a system level, add the following line to a /etc/sysctl.conf:
```
fs.file-max = 65536
```

The settings will be applied on the next reboot or you could reload them at runtime with sysctl -p. That’s it.
