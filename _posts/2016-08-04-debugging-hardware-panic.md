---
layout: post
title: "Debugging a Hardware Panic"
description: "Troubleshooting a 'Fatal hardware error!' kernel message"
tags: [technical, kernel panic, fatal hardware error, troubleshooting, crash]
modified: 2016-08-04
categories: 
---

So, a server rebooted.  Perhaps I’ve been lucky in the developers I work with, but whenever a host reboots, I always think hardware failure first. I treated this case no differently and jumped onto the host to run some quick diagnostics.
 
Since I deal in Dell equipment, my first stop is always `check_openmanage`. It gives a quick report on CPUs, DIMMs, chassis, power supplies, etc from the command line and takes about 30 seconds to run. It rarely fails me.
 
However, this time it did. Everything returned `[OK]`. No hardware failures.
 
I double-checked this claim on the iDRAC, just in case this was the one time in history a GUI was more correct than the modules running directly on the box. Nope, everything was green.
 
So I loaded up the kernel dump.

<!-- more -->
 
## Fatal Hardware Error!
 
There are plenty of guides out there on how to setup and run `crash`. If you’re reading this blog post, I assume you already got that far. If not, [this guide is pretty good](http://blog.zedroot.org/linux-kernel-debuging-using-kdump-and-crash/). Of course, if you haven’t configured your system to record dumps in the first place, take this as a lesson learned and do that ASAP.
 
I ran `crash` on the dump and it spat out this:

{% highlight shell %}
KERNEL: /usr/lib/debug/lib/modules/2.6.32-504.16.2.el6.x86_64/vmlinux
.
.
.
RELEASE: 2.6.32-504.16.2.el6.x86_64
VERSION: #1 SMP Wed Apr 22 06:48:29 UTC 2015
MACHINE: x86_64  (2500 Mhz)
MEMORY: 96 GB
PANIC: "[139839.578542] Kernel panic - not syncing: Fatal hardware error!"
{% endhighlight %}
 
`“Fatal hardware error!"` so something had failed. But my diagnostics didn’t agree.
 
I spent the next several hours down in the weeds.
 
Running `bt` gives you the stack trace of the active tasks at the time of the panic:

{% highlight shell %}
crash> bt
PID: 0      TASK: ffffffff81a8d020  CPU: 0   COMMAND: "swapper"
 #0 [ffff880053a06cc0] machine_kexec at ffffffff8103b5bb
 #1 [ffff880053a06d20] crash_kexec at ffffffff810c9942
 #2 [ffff880053a06df0] panic at ffffffff81529723
 #3 [ffff880053a06e70] ghes_notify_nmi at ffffffff8131c691
 #4 [ffff880053a06ea0] notifier_call_chain at ffffffff815304d5
 #5 [ffff880053a06ee0] atomic_notifier_call_chain at ffffffff8153053a
 #6 [ffff880053a06ef0] notify_die at ffffffff810a4f5e
 #7 [ffff880053a06f20] do_nmi at ffffffff8152e1c9
 #8 [ffff880053a06f50] nmi at ffffffff8152da60
    [exception RIP: intel_idle+177]
    RIP: ffffffff812eaae1  RSP: ffffffff81a01e38  RFLAGS: 00000046
    RAX: 0000000000000000  RBX: 0000000000000002  RCX: 0000000000000001
    RDX: 0000000000000000  RSI: ffffffff81a01fd8  RDI: ffffffff81a90500
    RBP: ffffffff81a01ea8   R8: 0000000000000002   R9: 000000000000009c
    R10: 00007f0efb0ee285  R11: 0000000000000000  R12: 0000000000000000
    R13: 145631f3e34bac61  R14: 0000000000000001  R15: 0000000000000000
    ORIG_RAX: ffffffffffffffff  CS: 0010  SS: 0018
--- <NMI exception stack> ---
 #9 [ffffffff81a01e38] intel_idle at ffffffff812eaae1
#10 [ffffffff81a01eb0] cpuidle_idle_call at ffffffff81426117
#11 [ffffffff81a01ed0] cpu_idle at ffffffff81009fc6
{% endhighlight %}
 
In this case, the failure happened in driver method `intel_idle` at line offset 177. This is usually where you decompile the kernel code (or throw the server away and go back to sleep) but I found the code online instead. The `intel_idle` driver forces CPUs into the lowest activity level when not in use instead of keeping the core warm. It’s meant to boost energy efficiency on your hardware, but I don’t care for the performance hit. In this case it looked like the method had failed to store CPU state for some reason, causing the panic. However, the server was running now and this seemed to be a one-off machine goblin occurrence.
 
Except a few hours later, when it crashed again.
 
And again.
 
And again.
 
`“Fatal hardware error!"` each time. Yet everything was coming up `[OK]`. What was failing? Was it the `intel_idle` driver? It ran on other hosts without issue. Why was it breaking here?
 
**Protip:** trust the kernel.
 
## The ESM Log
 
So, let’s cut to the chase. What was the issue and how did I find it?
 
The secret here is the ESM log. ESM stands for Embedded System Management and is sometimes known as the System Event Log. It logs every hardware-level event, even harmless ones, and it showed a critical failure on this host.
 
How did I find this? I didn’t. It turns out, an engineer on a completely different team had seen `esmlog` show up in `dmesg` one day and didn’t know what it was. And as he was learning about it, he just so happened to pick my troubled host to investigate.
 
Curiosity is awesome.

{% highlight shell %}
$ omreport system esmlog
 
Severity      : Critical
Date and Time : -- --  -- --:--:-- ----
Description   : A bus fatal error was detected on a component at bus 0 device 3 function 0.
{% endhighlight %}
 
`bus 0 device 3 function 0` refers to a location on the PCI. You can dig into this further with `lspci`.

{% highlight shell %}
$ lspci -s 00:03.0

00:03.0 PCI bridge: Intel Corporation Xeon E7 v2/Xeon E5 v2/Core i7 PCI Express Root Port 3a (rev 04)
{% endhighlight %}

So the failure was on the PCI itself. The `intel_idle` failure made more sense now: the processes were unable to share state across the bus.

You can also run a super detailed command on this device location:

{% highlight shell %}
$ lspci -vvxs 00:03:0
{% endhighlight %}

Which gave me, among many other things, this:

{% highlight shell %}
03:00.0 PCI bridge: Renesas Technology Corp. SH7757 PCIe Switch [PS] (prog-if 00 [Normal decode])
{% endhighlight %}

I put in a request for a new SH7757 PCIe Switch and that was that.
 
## Futureproofing
 
So, I debugged the kernel panic. But, there were gaps in my diagnostics that had me looking in the wrong places for hours. How do I prevent that from happening again?
 
You can get ESM log data from the `check_openmanage` command, but it isn’t included by default because the log itself is so noisy. However, I wanted visibility into the log, so I wrote a custom `nrpe` check to poll the ESM log and port it into Nagios. It doesn’t give details to the failures themselves, just if there are `CRITICAL` events on a host. It also doesn’t page people; if the failure is bad enough that the host reboots, we’ll know anyway.
 
So, why have the check if you’re not going to page on it? The developers I work with don’t have root access or deep systems knowledge, but they do have Nagios logins. And I would much prefer getting a page that says, “A server rebooted and I see a `CRITICAL` ESM log message in Nagios” vs “A server rebooted and lol idk.” Proper alerting is an art.
 
We haven’t needed the ESM log monitoring yet (fingers crossed!) but I’m glad the developers have access to it anyway. My future wishlist includes writing automation to clear the ESM log after a period of time, so only fresh events appear. Otherwise you need to manually clear the log after an incident.
 
I hope this post have been helpful. Now go back to bed.
