---
layout: post
title:  "How do I do reliable benchmarks on this bloody laptop ?"
---

So you want to benchmark a long running CPU intensive application on your laptop? 
That's cool, I have a neat trick for you if you want to do that: don't. Just 
don't. Don't run CPU intensive benchmarks on your laptop. Run them anywhere 
else: on a workstation, on a server, on your smartphone—well maybe not. Ok, you 
are doing it anyway, right? Well, I am doing it anyway. Let me describe the world 
of pain you are about to enter.

**Update 2015-08-09:** I describe a somehow acceptable solution to issues listed
here in this [new post]({% post_url 2015-08-08-reliable-laptop-benchmarks-solution %}),
so you might want to jump to it directly. Unless you want to know all the
different ways in which you can fail.

* TOC
{:toc}

## Setup ##

I am benchmarking the speed of a *single-core* algorithm implemented in C++
which varies depending on a parameter. I expect the speed to increase *very slightly* 
with the parameter value until a point where it collapses. The goal is to find 
the turning point.

My laptop has an Intel [Core i7 4810MQ](http://ark.intel.com/products/78937/Intel-Core-i7-4810MQ-Processor-6M-Cache-up-to-3_80-GHz)
built in which is perfect to run into issues. Its frequency goes from 2.8Ghz
all the way up to 3.8GHz (+1Ghz !) in turbo mode. So with this processor, I can be 
100% sure I will run into frequency scaling issues.

The full set of experiments runs for about 3 hours. Oh! And what's my good 
excuse to do CPU intensive benchmarks on my laptop? It's the only Haswell
processor I easily have access to and it worked well *last time*. So there is 
no reason it wouldn't work this time, right?

## Failure, Pain, Failure ##

### Obvious prerequisites ###

- Shutdown X11
- Do not use your computer while the benchmark runs
- Plug your laptop - Disable energy saving
- Set your frequency scaling governor to performance

To change your frequency scaling governor:

    $ for ((i=0; i<8; i++)); do sudo cpufreq-set -g performance -c $i
    $ cpufreq-info # Check

### Naive approaches ###

Let's start by naively running the full set of experiments

    $ ./runner.py

Oh, and by the way, before you start fainting, the `runner.py` python script is
just a wrapper which runs my algorithm — again, implemented in C++ — for each
value of the parameter. I am *not* benchmarking the CPU run time of a Python script.
And… look at the nice graph I get:

<img src="/assets/laptop-benchmarks/web-keep-0-edited.png" width="500" class="center-image" />

Of course the ugly dent was not expected, and is probably due to the experimental 
conditions — and from what will follow we can call them poor experimental 
conditions.

Ok, no problem, I will re-run the script for a few points (0.7% and 1%), I said. What could 
go wrong?

    $ ./runner.py 0.7 1 # Only a few points this time

It sure "fixed" the graph:

<img src="/assets/laptop-benchmarks/web-keep-2-edited.png" width="500" class="center-image" />

Time to think a little, what could be the causes of this issue? Well, first, laptops are designed to offer 
limited performance bursts, not run at full blast for three hours. The Core i7-4810MQ might be able
to run at 3.8Ghz but it has a TDP of 47W, where desktop processors have a TDP of about ~84W.

So, possible causes:

*Frequency scaling down because of excessive heat*  
However, frequency is expected to stabilize
at some point in the experiment as a function of room temperature and maximum fan speed.
It would explain a continuous decrease in performance during the experiment but
not this dent.

*Process moved to another core by the scheduler*  
It may also have been moved back to the original core after a while.
Moving a process from one core to another causes a temporary performance degradation because
the L1/L2 caches need to be warmed up again, the branch predictor needs to recover 
its state etc. Besides, the new core might run at a lower or higher frequency.

### Using taskset ###

Let's try to eliminate this second cause, and prevent the scheduler from moving 
the process to another core. I remind you that my algorithm uses a *single core*.
The `taskset` command allows binding commands (or PIDs) to a given CPU core by 
setting its CPU affinity:

    $ taskset -c0 ./runner.py

No luck.

<img src="/assets/laptop-benchmarks/web-keep-3.png" width="500" class="center-image" />

We still have this ugly dent in the graph. However notice how the graph is a lot smoother
than previously.

### Reducing the number of points ###

At this point I wanted to validate the idea that dents were caused by frequency scaling/scheduling issues
and not that I had a specific issue at 0.7%. I therefore tried to draw the same graph with less
points, thus reducing the likelihood of a frequency scaling or scheduling event. Moreover, I reduced the size of my dataset:
previously each point of the graph was an average on 10000 inputs, it is now an average of 2000 inputs.
Note that the x-axis of the graph below is different of previous ones. 

<img src="/assets/laptop-benchmarks/web-keep-4.png" width="500" class="center-image" />

So this confirms the hypothesis.

### Using cset ###

`taskset` does force a command to run on a specific CPU core but it does not prevent other
processes (or kernel interrupts) to get scheduled on this core. [Cpusets](http://man7.org/linux/man-pages/man7/cpuset.7.html)
offer a way to reserve cpu cores for a process and prevent other processes from using them. They can
be manipulated using the [`cset`](https://rt.wiki.kernel.org/index.php/Cpuset_Management_Utility/tutorial) userspace utility.

    $ sudo cset shield -k on -c 2,6 # 2 and 6 are hyperthreads of Core 2

This shields CPU cores 2 and 6 (which are two hyperthreads of the same core, see core id `/proc/cpuinfo`) and prevents Linux 
from scheduling any processes on them. 

Let's now execute our benchmark inside the shield: 

    $ sudo cset shield --exec ./runner.py

<img src="/assets/laptop-benchmarks/web-keep-5.png" width="500" class="center-image" />
<img src="/assets/laptop-benchmarks/web-keep-6.png" width="500" class="center-image" />

I ran the experiment twice to get a better understanding of what was going on. 
Graphs look a little bit better but there is still a lot of variability in the experiments.

### Controlling the environment ###

The last try was putting the laptop in a cooler room—yes, for real. I shielded only one hyperthread 
(ie core 2) to avoid the process from jumping from one hyperthread to another:

    $ sudo cset shield -k on -c 2
    $ sudo cset shield --exec ./runner.py

<img src="/assets/laptop-benchmarks/web-keep-7.png" width="500" class="center-image" />

With the exception of the first two points, the graph looks ok. Maybe for the 
first point the CPU was cold enough so that it could run at its full turbo 
frequency while starting from the second point, frequency decreases because of
heat accumulation.

## Conclusion ##

I was thinking: you know why people benchmark CPU cycles instead of run time in seconds? Maybe
because they don't want to deal with this madness. But, I started this way, I am 
not giving up until I get proper timings.

A few takeways:

- Shielding CPUs using cpusets removed dents in the graphs
- Preventing heat accumulation is probably important

Next directions:
    
- CPU warm up experiment. Run the experiment for the first point of the curve twice
    and discard the first run.
- Wait between each point for CPU temperature to decrase
- Monitoring frequency, CPU temperature during the benchmark to correlate with
    performance decrease.

[Comments on Hacker News](https://news.ycombinator.com/item?id=10012933)
