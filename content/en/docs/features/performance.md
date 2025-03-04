---
title: Performance
date: 2020-01-11T14:09:21+09:00
draft: false
weight: 301
collapsible: false
# showTitle: true
---

## Unikraft Performance

Unikraft has been extensively evaluated in terms of performance. Evaluations of
using off-the-shelf applications on Unikraft results in a 1.7x-2.7x performance
improvement compared to Linux guests.  In addition, Unikraft images for these
apps are around 1MB, require less than 10MB of RAM to run, and boot in around
1ms on top of the VMM time (total boot time 2ms-40ms).

{{< alert theme="info" >}}
You can find the full evaluation details and the description of Unikraft in
general in the [EuroSys'21 paper](https://dl.acm.org/doi/10.1145/3447786.3456248) (best
paper award).
{{< /alert >}}

### Image Size

In order to quantify image sizes in Unikraft, we generate a number of images for
all combinations of DCE (Dead Code Elimnation) and LTO (Link Time Optimization),
and for a helloworld VM and three other applications: nginx, Redis and SQLite.
The results in Figure 1 show that Unikraft images are all under 2MBs for all of
these applications. We further compare these results with other unikernels and
Linux in Figure 2. As shown, Unikraft images are smaller than all other
unikernel projects and comparable to Linux userspace binaries (note that the
Linux sizes are just for the application; they do not include the size of glibc
nor the kernel). This is a consequence of Unikraft's modular approach,
drastically reducing the amount of code to be compiled and linked (e.g., for
helloworld, no scheduler and no memory allocator are needed).

{{< img
  class="max-w-xl mx-auto"
  src="https://raw.githubusercontent.com/unikraft/eurosys21-artifacts/master/plots/fig_08_unikraft-image-size.svg"
  title="Figure 1"
  caption="Image sizes of Unikraft applications with and without LTO and DCE"
  position="center"
>}}

{{< img
  class="max-w-xl mx-auto"
  src="https://raw.githubusercontent.com/unikraft/eurosys21-artifacts/master/plots/fig_09_compare-image-size.svg"
  title="Figure 2"
  caption="Image sizes for Unikraft and other OSes, stripped, w/o LTO and DCE"
  position="center"
>}}

## Boot Times

In our evaluation, we use standard virtualization toolstacks instead, and wish
to understand how quickly Unikraft VMs can boot. When running experiments, we
measure both the time taken by the VMM (e.g. Firecracker, QEMU, Solo5) and the
boot time of the actual unikernel/VM, measured from when the first guest
instruction is run until main() is invoked.

{{< img
  class="max-w-xl mx-auto"
  src="https://raw.githubusercontent.com/unikraft/eurosys21-artifacts/master/plots/fig_10_unikraft-boot.svg"
  title="Figure 3"
  caption="Boot time for Unikraft images with different virtual machine monitors."
  position="center"
>}}

The results are shown in Figure 3, showing how long a helloworld unikernel needs
to boot with different VMMs. Unikraft's boot time on QEMU and Solo5 (guest only,
without VMM overheads) ranges from tens (no NIC) to hundreds of microseconds
(one NIC). On Firecracker, boot times are slightly longer but do not exceed 1ms.
These results compare positively to previous work: MirageOS (1-2ms on Solo5),
OSv (4-5ms on Firecracker with a read-only filesystem), Rump (14-15ms on Solo5),
Hermitux (30-32ms on uHyve), Lupine (70ms on Firecracker, 18ms without KML), and
Alpine Linux (around 330ms on Firecracker). This illustrates Unikraft's ability
to only keep and initialize what is needed.

Overall, the total VM boot time is dominated by the VMM, with Solo5 and
Firecracker being the fastest (3ms), QEMU microVM at around 10ms and QEMU the
slowest at around 40ms. These results show that Unikraft can be readily used in
scenarios where just-in-time instantiation of VMs is needed.


## Memory Consumption

How much memory does Unikraft actually need when running real-world
applications? To answer this question we ran experiments to measure the minimum
amount of memory required to boot various applications as unikernels, finding
that 2-6MBs of memory suffice for Unikraft guests (Figure 4).

{{< img
  class="max-w-xl mx-auto"
  src="https://raw.githubusercontent.com/unikraft/eurosys21-artifacts/master/plots/fig_11_compare-min-mem.svg"
  title="Figure 4"
  caption="Minimum memory needed to run different applications using different OSes, including Unikraft."
  position="center"
>}}

## Application Performance

We conduct all measurements with the same application config and where possible
the same application version (this is limited by application support, e.g.,
Lupine, HermiTux, and Rump only support a specific version of Redis), on a
single core. We did not optimize application or kernel configs for performance,
however we took care of removing obvious performance bottlenecks for each
system, e.g., switching on memory pools in Unikraft's networking stack (based on
lwIP), or porting Lupine to QEMU/KVM in order to avoid Firecracker performance
bottlenecks. Unikraft measurements use Mimalloc as the memory allocator.

{{< img
  class="max-w-xl mx-auto"
  src="https://raw.githubusercontent.com/unikraft/eurosys21-artifacts/master/plots/fig_12_redis-perf.svg"
  title="Figure 5"
  caption="Redis perf (30 conns, 100k reqs, pipelining 16) with QEMU/KVM and Firecracker (FC)"  
  position="center"
>}}

The results are shown in Figures 5 and 6. For both apps, Unikraft is around
30%-80% faster than running the same app in a container, and 70%-170% faster
than the same app running in a Linux VM. Surprisingly, Unikraft is also 10%-60%
faster than Native Linux in both cases. We attribute these results to the cost
of system calls (aggravated by the presence of KPTI — the gap between native
Linux and Unikraft narrows to 0-50% without KPTI), and possibly the presence of
Mimalloc as system-wide allocator in Unikraft; unfortunately it is not possible
to use Mimalloc as kernel allocator in Linux without heavy code changes. Note
that we did try to LD_PRELOAD Mimalloc into Redis, but the performance
improvement was not significant. We expect that the improvement would be more
notable if Mimalloc were present at compile time instead of relying on the
preloading mechanism (making the compiler aware of the allocator allows it to
perform compile/link time optimizations) but we could not perform this
experiment since Mimalloc is not natively supported by the current Redis code
base

{{< img
  class="max-w-xl mx-auto"
  src="https://raw.githubusercontent.com/unikraft/eurosys21-artifacts/master/plots/fig_13_nginx-perf.svg"
  title="Figure 6"
  caption="NGINX (and Mirage HTTPreply) performance with wrk (1 minute, 14 threads, 30 conns, static 612B page)"
  position="center"
>}}

Compared to Lupine on QEMU/KVM, Unikraft is around 50% faster on both Redis and
NGINX. These results may be due to overcutting in Lupine's official
configuration, scheduling differences (we select Unikraft's cooperative
scheduler since it fits well with Redis's single threaded approach), or
remaining bloat in Lupine that could not be removed via configuration options.
Compared to OSv, Unikraft is about 35% faster on Redis and 25% faster for nginx.
Rump exhibits poorer performance: it has not been maintained for a while,
effectively limiting the number of configurations we could apply. For instance,
we could not set the file limits because the program that used to do it
(rumpctrl) does not compile anymore. HermiTux does not support nginx; for Redis,
its performance is rather unstable. This is likely due to the absence of virtio
support, as well as performance bottlenecks at the VMM level (HermiTux relies on
uHyve, a custom VMM that, like Firecracker, does not match the performance of
QEMU/KVM). Unlike Lupine, porting it to QEMU/KVM requires significant code
changes.
