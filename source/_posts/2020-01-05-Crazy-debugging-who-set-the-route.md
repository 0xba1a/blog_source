---
title: Crazy debugging - who set the route?
date: 2020-01-05 14:00:10
thumbnail: "/images/debug.jpg"
tags:
 - debug
 - crazy debugging
---

It was a bright winter day. I like winter because winter in Bangalore is like summer in London. We had just completed our biggest sale event of the year. Everything was cool and people were in peace.

I had one small assignment. To make a static route persistent. That route was used to communicate between host and VM. As its not persistent when the VM reboots, the communication was lost. It was a pretty simple task of adding an entry in `/etc/network/interfaces`. I just need to 1. Test the existing behaviour, 2. Make the change, 3. Test new behaviour, 4. Document, 5. Release. All good omen.

```sh
$ ip r s
default via 10.0.0.1 dev eth0
169.254.1.1 dev eth0 scope link
$
```

At the time we were supporting two guest images. Debian-8 and Debian-9. So I should test on both. While testing the existing behaviour Debian-9 behaved as expected but the route was already persistent in Debian-8. As you all know, instead of being half happy I was double worried. I can't simply go ahead without understanding why it was happening.

Lets check the logs.

```sh
$ sudo dmesg | grep route
$
$ sudo cat /var/log/syslog | grep route
$
```

Nothing.

We've a script called `first_run.sh`. It was the script that set this static route. This script would be executed upon first start of a VM. It won't run on subsequent boots. There should be no other script doing this. So I ran a system-wide grep for the static ip.

```sh
$ sudo grep -rn '169.251.1.1' /  2>/dev/null
/usr/sbin/first_run.sh
```

Good. As expected no other file other than `first_run.sh` sets the route. So lets check whether this script is mistakenly called at startup.

```sh
$ sudo grep -rn 'first_run' / 2>/dev/null
$
```

I couldn't think further. There is only one file that sets the route. And that file is not being called.
The result was empty because the script that invoked it on first boot had been deleted. Don't ask me why this file was not deleted too.

To know how important something is, delete it.

I deleted the file and restarted VM. The route was not there.

```sh
$ ip r s
default via 10.0.0.1 dev eth0
$
```

Good. At least I'm half correct. The route was set by the `first_run.sh`. The assumption that it ran only on first boot was wrong. So who calls it?

I did this simple hack. Moved `ip` command inside `/lib/`.
```sh
$ which ip
/bin/ip
$ sudo mv /bin/ip /lib/ip
$ which ip
$
```

And wrote a script that will log before calling `/lib/ip`.
```sh
$ cat /bin/ip
#!/bin/bash
pid=$$
echo $(cat /proc/${pid}/cmdline)
ppid="$(ps -p ${pid} -o ppid | grep -v 'PPID' | tr -d '[:space:]')"
echo $(cat /proc/${ppid}/cmdline)
/lib/ip "$@"
$
```

New `ip` command will echo the command before doing actual job.
```sh
$ ip route show
/bin/bash/sbin/iprouteshow
bash
default via 10.0.0.1 dev eth0
169.254.1.1 dev eth0  scope link
```

After restart
```sh
$ cat /var/log/syslog | grep route
Jan 11 16:07:18 hostname startup_service[726]: /bin/bash/sbin/iprouteaddto169.254.1.1/32deveth0
$
$ cat /var/log/syslog | grep startup_service
Jan 11 16:07:18 hostname startup_service[726]: /bin/bash/sbin/iprouteaddto169.254.1.1/32deveth0
Jan 11 16:07:18 hostname startup_service[726]: /bin/bash/usr/sbin/startup_service
$
```

So `startup_service` was the one that sets this route. But how?
```sh
$ cat /etc/systemd/system/multi-user.target.wants/startup_service.service
[Unit]
Wants=network-online.target
After=network-online.target

[Service]
ExecStart=/usr/sbin/startup_service

[Install]
WantedBy=multi-user.target

$
$ ls -lh /usr/sbin/startup_service
lrwxrwxrwx 1 root root 21 May 23  2019 /usr/sbin/startup_service -> /usr/sbin/first_run.sh
$
```

Wow! Amazing. Somebody has already found the persistent route problem and fixed with this hack. Once again its proved that documents are for sages.
