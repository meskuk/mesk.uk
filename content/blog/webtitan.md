+++
title = 'Bypassing WebTitan Cloud OTG 2.1.4'
date = 2026-02-01T14:45:00+00:00
summary = 'Loose computer policies = no more web filter.'
draft = true
+++


Web filtering is very common in organisations. Maybe a library doesn't want you looking for Linux ISOs online,
or your workplace definitely doesn't need you using Instagram on the job.

Filtering can be done in a few ways: transparently by the network's firewall, or on the endpoint[^1]

WebTitan Cloud OTG, or On The Go, is the agent we'll be working with today. There's nothing _wrong_ with agents on the
endpoint, but they need to be paired with good policies - or you'll end up with this blog post.

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

So if my guess is right, I don't need to hunt down the DNS server on this computer. (But if I knew nothing, I'd probably try that first).

## The config
Let's look at the configuration, which is presumably `service.conf`. Some important snippets:

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

This part forwards queries for the root zone (`.`) and the reverse DNS zone (`in-addr.arpa`) to the respective servers.
Basically, WebTitan's filter gets all your DNS queries.

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

This enables 
turns off certificate verification, meaning you don't need to prove your identity to the service at all.

But first, I will try a simple solution: kill unbound.

```
> (Get-Process -Name *unbound*).Kill()
Exception calling "Kill" with "0" argument(s): "Access is denied"
[ insert more here ]
```

It doesn't work. Moving on.

## Using remote control

Time to check that remote control is working, by checking the forward zones.

```
PS C:\Program Files (x86)\WebTitan Cloud OTG\Unbound> .\unbound-control.exe -c .\service.conf list_forwards
. IN forward 52.209.170.167 52.209.115.90
in-addr.arpa. IN forward 156.154.70.1 156.154.71.1
```

Excellent! Time to remove one.

```
PS C:\Program Files (x86)\WebTitan Cloud OTG\Unbound> .\unbound-control.exe -c .\service.conf forward_remove .
ok
PS C:\Program Files (x86)\WebTitan Cloud OTG\Unbound> .\unbound-control.exe -c .\service.conf list_forwards
in-addr.arpa. IN forward 156.154.70.1 156.154.71.1
```

It works! No more filtering. In a nutshell:

```powershell
cd "C:\Program Files (x86)\WebTitan Cloud OTG\Unbound"; .\unbound-control.exe -c .\service.conf forward off
```

[^1]: Oho, what a word. I feel like a real sysadmin now.
[^2]: I realise it's more methodical to confirm with `netstat` and `nslookup` that yes, this is the DNS server, but the block pages and tray icon made it really apparent.