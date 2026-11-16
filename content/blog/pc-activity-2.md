+++
title = 'Improving the chart'
date = 2026-11-16T13:13:00Z
summary = 'The side project survived the pile'
draft = false
+++

I wrote a [brief post](/blog/pc-activity.md) about trying to make a fancy chart to gauge the 'activity' across classroom computers at my campus.

To reiterate, what I started with was:
- PowerShell to collect data
- `pyplot` to generate a chart

It looked rather ugly. But it worked:

![Old chart for one of the labs on campus](/img/pc-activity/chart.png)

Recently, I've done something of an overhaul. You can see it at [/charts](/charts) on this website!

![New chart for one of the labs on campus](/img/pc-activity/new-chart.png)

## Python

I made major changes to the Python side, like dropping `pyplot` in favor of [Jinja](https://palletsprojects.com/projects/jinja) and some templates. It's a great library, but I honestly didn't feel like changing more and more about the chart type I was using. I also had an easier time controlling the look of a webpage!

Doing this made me finally find out normalisation exists. I use "[min-max normalisation]" on timestamps I collect, and map it to the Lightness of a HSL colour, which makes a lovely black to white to green scale.

[min-max normalisation]: https://www.geeksforgeeks.org/data-analysis/normalization-and-scaling/#types-of-normalization-techniques

Personally, it's nicer on the eye than my previous idea of using saturation:

![Comparison of colour scale by lightness vs colour scale of saturation](/img/pc-activity/lightness_vs_saturation.png)

It also generates pages that are linked between eachother, for easy navigation.

## PowerShell

On the PowerShell side, I did work to support collecting the data for classrooms at once.

One interesting part was deciding on how to do LDAP filters. I wanted the script to be easy to invoke, but not expose the naming convention here, which might be
unwanted by IT.
f
So instead of using raw filters, the script takes room numbers (`-Rooms` argument) and makes a filter like this:

```powershell
$searcher.Filter = "(&(objectClass=computer)(name=*-LAB-$room))"
```

It's very restrictive compared to before, but I'll expand it when I need to.

## Data

The data format used by both scripts has now changed. At the start, it was something like this:

```json
{
    "COMPUTERNAME": 1794869466 // Unix seconds
}
```

Very obviously inflexible as it doesn't name the classroom, let alone when the data is from.

Since I needed to split the computers into groups (classrooms), I went with this across multiple files:

```json
{
    "name": "Classroom name",
    "values": {
        "COMPUTERNAME": 1794869466
    }
}
```

Excellent! If I don't like dealing with several files, I can even merge these into an array, which I [also tried].

[also tried]: https://github.com/meskuk/pcmap/blob/d97977174972f88257f7db8f53d0fc379d09374f/data/test.json

But without context on _when_ the data was taken, the colours will change _every time_ the site is generated. Not good.

I did much agonising before ending up with this:

```json
{
    "collected_at": 1794869466,
    "rooms": [
        {
            "name": "Classroom name",
            "times": {
                "COMPUTERNAME": 1794869466
            }
        }
    ]
}
```

The only reason things ended up this way is that I didn't take the time to think things over while iterating. It's great to hit the ground running - but only in moderation!

## What's next?

- **Automation:** The overall goal is to have enough data to actually see change over time. Sadly, a daily overview requires criminal amounts of shadow IT.

- **Not using a static site:** I might stand up a service in the cloud. The biggest bottleneck right now is publishing the data as it requires access to my GitHub account. If I build a service, I can just upload times 
to a server from any university PC.

- **Other data:** I also want to look into other stats. For example, how many students logged in today? What's the market share of Gmail in the student body, according to Active Directory? And other very ethical ideas.

- **More cleanup:** My code is admittedly not at its best. It needs some more love.

Hopefully this takes me to a good place! Thanks for reading! :)
