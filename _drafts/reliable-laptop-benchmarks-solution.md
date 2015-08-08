---
layout: post
title:  "Repeatable single-core CPU intensive benchmarks — laptop edition"
---

<!--

- Running CPU intensive benchmarks in a reliable and repeatable manner on laptops
is not easy.
- discrépancies
- Save
- In this post, I give directions that should help you
-->

When benchmarking execution time of CPU intensive programs on laptops,
*scheduling events* and *frequency scaling events* *will* cause discrepancies in
your experimental results. One solution to this issue is to measure CPU cycles
instead of execution time, but execution time is often a more meaningful
measure. In this post, I give advice to mitigate these two factors and thus
benchmark execution times on laptops in a
*reliable* and *repeatable* way. Advice on preventing scheduling events remains
relevant when benchmarking execution times on workstations or servers.

* TOC
{:toc}

## Issue description ##

It is often useful to measure the execution time of CPU intensive programs,
for instance to evaluate the sensitivity of an algorithm to a parameter, or
compare two algorithms. If you don't take into account the variability caused
by *scheduling events* and *frequency scaling events*, you will obtain
inconsistencies in your results, as shown below:

<img src="/assets/laptop-benchmarks/web-keep-0-edited.png" width="500" class="center-image" />

*Scheduling events* and *frequency scaling events* impact performance in two
different ways:

- *Scheduling events:* The Linux scheduler will *sometimes* move your process
from one core to another. When a process is moved to a differents core, L1/L2 caches
need to be filled again and the branch predictor needs to recover its state. This usually
causes a drop in performance. The Linux scheduler avoids moving processes around
for theses reasons but if your benchmark runs for long enough, theses events
will still occur. This is not a laptop specific issue, and it will also happen
on workstations and servers.

- *Frequency scaling events:* All modern Intel processors — not only laptop
ones — change the frequency of each of their cores based on their current temperature
so that they don't exceed their TDP (Thermal Design Power), the
maximum amount of heat they are allowed to dissipate. Although this issue affects
all processors, this issue is more salient of laptops for two reasons. Laptop cooling
systems don't dissipate heat as effectively as those of workstations or servers, and
laptop processors have lower TDPs that workstation or server ones. In other words,
your laptop is not designed to run at full speed for long periods — only to offer
short performance bursts — but we will work this around.

These events tend to combine in nasty ways. For instance, the Linux scheduler may
realize that the core you are running your benchmark on is getting hot and
is thus decreasing its frequency. It will therefore move your process to another
cooler and idling core, which can run at a higher frequency. Overall, this causes (i) the frequency to
change, (ii) the branch predictor to be reset, (iii) caches to be flushed. No
wonder you get inconsistencies!

This post focuses on benchmarks which are:

- CPU-intensive (and optionally memory intensive)
- Single-core
- Long-running, ie, *total* benchmark time (for all algorithms/parameters) exceeds
30 minutes.

Shorter run times diminish the likelihood of *scheduling events* and
*frequency scaling events* but you may observe them anyway.

Applying this post advice should allow you to obtain nice graphs:

<img src="/assets/laptop-benchmarks/web-keep-8.png" width="500" class="center-image" />

## Requirements ##

- Shutdown X11
- Do not use your computer while the benchmark runs
- Plug your laptop - Disable energy saving

Shutting down X11 helps ensuring that not too many services and apps are running.
This reduces the likelihood that one will interfere with the benchmark — ie, your mail client
won't decide it is the perfect moment to start compressing and re-indexing all your mails.

## Preventing scheduling events ##

The first thing to do is to reserve one core for your benchmark, and prevent
the scheduler to schedule anything but your benchmark on this core. This can be done using
[cpusets](http://man7.org/linux/man-pages/man7/cpuset.7.html) which can be
handled using the [`cset`](https://rt.wiki.kernel.org/index.php/Cpuset_Management_Utility/tutorial)
command line tool.

If your laptop has hyperthreading enabled (if it is equipped with a Core i7), you
need to reserve a *full physical* core, and not only one logical core, or hyperthread.
If your reserve only one hyperthread, the scheduler may schedule other processes
than your benchmark on the other hyperthread of the physical core, which will
pollute L1/L2 caches, the branch predictor and the pipeline — you don't want
that.

To do so, have a look at `/proc/cpuinfo`

    $ cat /proc/cpuinfo
    processor       : 0
    [...]
    model name      : Intel(R) Core(TM) i7-4810MQ CPU @ 2.80GHz
    [...]
    siblings        : 8
    core id         : 0
    cpu cores       : 4
    [...]

    processor       : 1
    [...]
    model name      : Intel(R) Core(TM) i7-4810MQ CPU @ 2.80GHz
    [...]
    siblings        : 8
    core id         : 1
    cpu cores       : 4
    [...]

    processor       : 2
    [...]
    model name      : Intel(R) Core(TM) i7-4810MQ CPU @ 2.80GHz
    [...]
    siblings        : 8
    core id         : 2   # Core 3
    cpu cores       : 4
    [...]


    processor       : 6
    [...]
    model name      : Intel(R) Core(TM) i7-4810MQ CPU @ 2.80GHz
    [...]
    siblings        : 8
    core id         : 2   # Core 3
    cpu cores       : 4
    [...]


It has one entry per logical core (`processor: [id]`), or hyperthread. Each
entry indicates the physical core (`core id: [id]`) the logical core belongs to.
Here, we can see that logical cores 2 and 6 correspond to the physical core 2,
the third core of my quad core processor.

Let's reserve, or *shield* this third core:

    $ sudo cset shield -k on -c 2,6 # With hyperthreading
    $ sudo cset shield -k on -c 2   # Without hyperthreading

To execute commands on the shielded core, you can use `cset shield --exec [command]`.
However, if your processor has hyperthreading enabled, you should furthermore force
your benchmark to execute on one specific hyperthread of the physical core
you shielded. This can be done using `taskset` in addition to `cset`

    # Execute benchmark inside the shield and force it to always
    # execute on logical core 2 (never logical core 6)
    $ sudo cset shield --exec taskset -- -c 2 ./benchmark # With hyperthreading

    $ sudo cset shield --exec ./benchmark # Without hyperthreading


## Preventing frequency scaling events ##

First, ensure your laptop is plugged and that your CPU frequency scaling governor
is set to performance:

    for ((i=0; i<8; i++))
    do
        sudo cpufreq-set -g performance -c $i
    done
    cpufreq-info # Check

The next — and last — thing to do is to ensure that your processor does not heat
too much. And the only way to do this is to stop your benchmark at different times
during the course of its execution. Make sure to do this at moments were it
does not interfere with your results. Also, do not stop and wait too often
because it would prevent your processor to reaching its highest frequency.

In my case, each point is the median of 10000 experiments, which is long enough.
I therefore decided to stop and wait for the processor to cool down before
each point. This ensures that all points where benchmarked in the same experimental
conditions.

The next question is how long to wait for the processor to cool down. If you measure
the temperature of the CPU core you execute your benchmark on, for instance using
the `sensors` (lm-sensors package), you will see that it decreases very quickly
(< 1 seconds) after you stop doing calculation. However, there is still heat
accumulated in your laptop case and you need to wait more.

I therefore decided to wait for my laptop fan to go below 3600 RPM — you may
need to adjust this value to your laptop.

You can find [in this gist](https://gist.github.com/Xion345/af6ee03942ed4c8d511e),
the Python script I used to parse `sensors` command output.


This advice should be enough to get you sorted — unless you have a NUMA laptop, and in this
case… Wow. Just wow.

