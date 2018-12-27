---
layout: post
title: "Enabling `perf` in Kubernetes with Docker’s default seccomp profile"
description: "Using Linux capabilities to profile containerized processes"
tags: [technical, Kubernetes, Docker, containers, perf, FlameGraphs]
modified: 2018-12-21
categories: 
---

Have you been trying to profile your Kubernetes applications with `perf`? Maybe you want to see what [all the FlameGraphs fuss is about](http://blog.alicegoldfuss.com/making-flamegraphs-with-containerized-java/){:target="_blank"}? If your version of Docker was upgraded within the last year, you’ll likely run into issues.

<!-- more -->

Starting in v17.06 of Docker, `perf_event_open` is [blocked by the default seccomp profile](https://docs.docker.com/engine/security/seccomp/#significant-syscalls-blocked-by-the-default-profile){:target="_blank"}. Which means running `perf` inside your container will get you this:

```
perf_event_open(..., PERF_FLAG_FD_CLOEXEC) failed with unexpected error 1 (Operation not permitted)
perf_event_open(..., 0) failed unexpectedly with error 1 (Operation not permitted)
You may not have permission to collect stats.
Consider tweaking /proc/sys/kernel/perf_event_paranoid:
 -1 - Not paranoid at all
  0 - Disallow raw tracepoint access for unpriv
  1 - Disallow cpu events for unpriv
  2 - Disallow kernel profiling for unpriv
```

Trying to alter the suggested `/proc/sys/kernel/perf_event_paranoid` from within the container gets you the expected:

```
bash: /proc/sys/kernel/perf_event_paranoid: Read-only file system
```

What to do? You’ll need to enable `CAP_SYS_ADMIN`. This flag is one of many [Linux capabilities](http://man7.org/linux/man-pages/man7/capabilities.7.html){:target="_blank"}, so named for the extra capabilities they grant. These flags grant scoped permission escalations for threads to perform specific tasks, from changing file attributes to altering the system clock. `CAP_SYS_ADMIN` is a particularly overloaded one, a kitchen sink of permissions escalations mostly geared toward profiling work.

If you’re only working with Docker, you can add `--cap-add SYS_ADMIN` to your `docker run` command, [as explored here](https://docs.docker.com/engine/reference/run/#additional-groups){:target="_blank"}.

However, if you’re living that Kubernetes life, you’ll need to enable it using a `securityContext`. In the container spec of your deployment file, add:

```
securityContext:
    capabilities:
        add: ["SYS_ADMIN"]
```

And you’ll be good to go!

Note that you need to strip the `CAP` prefix when adding capabilities in Kubernetes. You can read more about [container privileges in Kubernetes here](https://kubernetes.io/docs/tasks/configure-pod-container/security-context/#set-capabilities-for-a-container){:target="_blank"}.

**Remember to remove this setting when you’re done using it!** `perf_event_open` is blocked by default because it grants user processes privileged access to the system. Branch deploy your change, use it, then rollback.
