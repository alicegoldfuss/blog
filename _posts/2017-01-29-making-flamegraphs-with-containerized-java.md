---
layout: post
title: "Making FlameGraphs with Containerized Java"
description: "Linux performance profiling with perf and Docker"
tags: [technical, flamegraph, perf, java, cassandra, docker, containers]
modified: 2017-01-29
categories: 
---

About a month ago, I had the pleasure of taking a tutorial led by the fantastic Brendan Gregg on creating [FlameGraphs](https://github.com/brendangregg/FlameGraph){:target="_blank"} using the Linux `perf` toolset. I recommend reading his many [blog posts](http://www.brendangregg.com/flamegraphs.html){:target="_blank"} on the subject, but in short: while `perf` is an excellent resource for debugging kernel and user space processes, FlameGraphs make the data even easier to consume.

Now, if the process you’re trying to profile is Java, there are some extra hoops to jump through, which Brendan has [also detailed online](http://techblog.netflix.com/2015/07/java-in-flames.html){:target="_blank"}.

But if the Java process is in a container, it's even more annoying. That’s where this post comes in.

<!-- more -->

## Some context

As explained in Brendan’s blog post [here](http://techblog.netflix.com/2015/07/java-in-flames.html){:target="_blank"}, `perf` doesn’t work out of the box on Java, because Java doesn’t automatically expose stacks and method names. Running `perf` without these gives you something like this:

<img src="/images/flamegraph1.png" alt="">

Notice the nondescript frame dedicated to "java"? Not very helpful.

Running Java with the option `-XX:+PreserveFramePointer` (starting in JDK8u60) will expose the stacks. However, without the method name symbols, you get this:

<img src="/images/flamegraph2.png" alt="">

You need to also collect and dump the symbols of the running Java process, so `perf` can apply them to the correct stacks. This is made easier by Johannes Rudolph's [perf-map-agent](https://github.com/jrudolph/perf-map-agent){:target="_blank"} repo. It has some scripts that will dump the Java process symbols and even integrate with the [FlameGraph](https://github.com/brendangregg/FlameGraph){:target="_blank"} repo to make the graphs for you with one command. It's pretty slick.

Enter containers.

## Containers

Containers, for all their hype and mystery, are still processes on a host. Run a `ps` and you can see all container processes running the same as noncontainerized ones.

{% highlight shell %}
$ ps -ef | grep java
103  88834  88800 33 Jan27 ?        10:05:13 /usr/java/default/bin/java
{% endhighlight %}

That Java process is running inside a Docker container, and from the point of view of the host, it has PID `88834` and UID `103`.

Inside the container, that Java process has PID `27` and is owned by the `cassandra` user.

{% highlight shell %}
$ ps -ef | grep java
cassand+     27      1 33 Jan27 ?        10:05:20 /usr/java/default/bin/java
{% endhighlight %}

Herein lies the issue. Due to a bug in Java, you must dump the process symbols while operating as the owner of the Java process. The `perf-map-agent` scripts require it. But the process owner (`cassandra`) only exists within the container. Meanwhile, the `perf` toolkit must be run as root, and it’s common practice not to allow root within running containers.

So, how can you dump the symbols?

## The hack

The hack ("workaround" is too elegant a word) is to run `perf` outside on the host, dump the symbols inside the container, and marry the two resulting files in the same space to make a FlameGraph.

More specifically:

1.  Setup the FlameGraph repo on your host and the `perf-map-agent` repo inside the container where the Java process owner can access it. I also had to alter `/etc/passwd` inside the container to give my `cassandra` user a shell (use `vipw` for safety).
2. Capture a system profile on the host with something like
    ```
    sudo perf record -F 99 -a -g -- sleep 30
    ```
3. From inside the container (easier to have this running already in another shell) dump the symbols for the Java process with
    ```
    java -cp attach-main.jar:$JAVA_HOME/lib/tools.jar \
    ```
    ```
    net.virtualvoid.perf.AttachOnce PID
    ```
3. You will now have a `perf-PID.map` file inside `/tmp` of the container. Move this file to the host (I used a mounted volume).
4. Now on the host, rename the `perf-PID.map` file to match the PID of the Java process *as seen by the host*. For example, my file was named `perf-27.map` but the host has that PID as `88834`, so I renamed it to `perf-88834.map`
5. Move the re-named `perf-PID.map` file to your host's `/tmp` directory and `chown` it to `root`
6. You can now proceed with the directions as though containers are not involved. So, create a FlameGraph with
    ```
    sudo perf script | stackcollapse-perf.pl | flamegraph.pl \
    ```
    ```
    --color=java --hash > flamegraph.svg
    ```

You will need to alter this command depending on where your `perf.data` file resides in relation to the FlameGraph repo.

*Voila!* A containerized Java FlameGraph.

<img src="/images/flamegraph3.png" alt="">

**Tips:**

- Let Java warm up before profiling it to ensure less churn in symbol creation. I let mine run for 15 minutes.
- Run the `perf` profile before dumping the symbols. Switching the order might result in empty stacks, because the symbols were created in the JVM after the `perf-PID.map` file.


## Why?

Why is this hack needed? Why can't we dump the symbols outside the container?

At first glance, it seems easy enough to just create a `cassandra` user on the host with UID 103. But trying to dump the Java symbols gives us an error:

{% highlight shell %}
[cassandra@hostname]$ java -cp attach-main.jar:$JAVA_HOME/lib/tools.jar net.virtualvoid.perf.AttachOnce 88834
Exception in thread "main" com.sun.tools.attach.AttachNotSupportedException: Unable to open socket file: target process not responding or HotSpot VM not loaded
    at sun.tools.attach.LinuxVirtualMachine.<init>(LinuxVirtualMachine.java:106)
    at sun.tools.attach.LinuxAttachProvider.attachVirtualMachine(LinuxAttachProvider.java:63)
    at com.sun.tools.attach.VirtualMachine.attach(VirtualMachine.java:208)
    at net.virtualvoid.perf.AttachOnce.loadAgent(AttachOnce.java:37)
    at net.virtualvoid.perf.AttachOnce.main(AttachOnce.java:33)
{% endhighlight %}

This is the same behavior you get if you try to dump the symbols as a user who doesn't own the Java process. So, the host's `cassandra` user can't attach to a socket. What kind of socket? JMX or UNIX? Not sure. [The documentation isn't super clear.](https://docs.oracle.com/javase/6/docs/jdk/api/attach/spec/com/sun/tools/attach/VirtualMachine.html#attach(java.lang.String)){:target="_blank"}

Even `nsenter` fails here:

{% highlight shell %}
[root@hostname]$ nsenter -t 88834 -n java -cp attach-main.jar:$JAVA_HOME/lib/tools.jar net.virtualvoid.perf.AttachOnce 88834
Exception in thread "main" com.sun.tools.attach.AttachNotSupportedException: Unable to open socket file: target process not responding or HotSpot VM not loaded
    at sun.tools.attach.LinuxVirtualMachine.<init>(LinuxVirtualMachine.java:106)
    at sun.tools.attach.LinuxAttachProvider.attachVirtualMachine(LinuxAttachProvider.java:63)
    at com.sun.tools.attach.VirtualMachine.attach(VirtualMachine.java:208)
    at net.virtualvoid.perf.AttachOnce.loadAgent(AttachOnce.java:37)
    at net.virtualvoid.perf.AttachOnce.main(AttachOnce.java:33)
{% endhighlight %}

Walking the same network namespace as process `88834` still doesn't access the socket.

I talked to several people about this and each conversation ended in puzzlement. Usually I would only post once I had all the answers, but I think it's good to illustrate that everyone gets stuck sometimes. And it's better to get the hack out there as a stopgap in the meantime, clunky though it might be. I look forward to a more elegant solution.

## Special thanks

I want to thank Brendan Gregg, Johannes Rudolph, and Nitsan Wakart for creating and maintaining the FlameGraph and perf-map-agent repos, as well as helping me initially troubleshoot. Thank you to Jérôme Petazzoni for his unique container systems knowledge and my colleague Mike Hix for poking at namespaces. I am proud to work with all of you and delighted to occasionally stump you.
