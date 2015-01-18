---
layout: post
title:  "Binding Threads to Cores on OSX"
date:   2015-01-18 21:52:24
categories:
tags: c, osx, pthread
---

When I was porting the [mbw][1] to the OSX, I found that

1. `sched_getaffinity`
2. `pthread_setaffinity_np`

are missing, so that I could not bind a thread
to a core. The result is the mbw threads could be running on the same core
and compete for per-core resources, although the OSX kernel scheduler is
unlikly to do so. I simply simulate these calls by using native OSX calls.

The first thing to do is to detect the number CPU cores available. On Linux,
the `sched_getffinity` fills the `cpu_set_t` data structure provided and
we can use `CPU_ISSET(id, cpu_set_t)` to determine the number cores. Similarly
on OSX, the function
{% highlight c %}
sysctlbyname(const char *name, void *oldp, size_t *oldlenp,
             void *newp, size_t newlen);
{% endhighlight %}
can be used to get various system informations. In our case, the follwoing
code gets the number of CPU cores. The `cpu_set_t` is just a 32-bit variable
so that the max number cores is 32. Of course the actual
`sched_getaffinity` function is much more complicated, but the current
implementation is enough to suit our purpose. The physical core number
is stored directly in the `core_count` variable.

{% highlight c %}
#define SYSCTL_CORE_COUNT   "machdep.cpu.core_count"

typedef struct cpu_set {
  uint32_t    count;
} cpu_set_t;

static inline void
CPU_ZERO(cpu_set_t *cs) { cs->count = 0; }

static inline void
CPU_SET(int num, cpu_set_t *cs) { cs->count |= (1 << num); }

static inline int
CPU_ISSET(int num, cpu_set_t *cs) { return (cs->count & (1 << num)); }

int sched_getaffinity(pid_t pid, size_t cpu_size, cpu_set_t *cpu_set)
{
  int32_t core_count = 0;
  size_t  len = sizeof(core_count);
  int ret = sysctlbyname(SYSCTL_CORE_COUNT, &core_count, &len, 0, 0);
  if (ret) {
    printf("error while get core count %d\n", ret);
    return -1;
  }
  cpu_set->count = 0;
  for (int i = 0; i < core_count; i++) {
    cpu_set->count |= (1 << i);
  }

  return 0;
}
{% endhighlight %}

The next step is to simulate the function `pthread_setaffinity_np`. This time
we use the [affinity API](https://developer.apple.com/library/mac/releasenotes/Performance/RN-AffinityAPI/)
to bind a thread to a core. The first paramter is the thread you want to
bind. The second parameter specifies the policy you want to set, and
it is `THREAD_AFFINITY_POLICY`. The third one is called the `affinity tag`.
Threads in an affinity set have the same `affinity tag` indicating that the
threads want to share an L2 cache. We set the tags differently
for different threads so that the kernel may try to schedule on different cores.


As you may already notice that the `pthread_t` is not compatible with the first
parameter provided to `thread_policy_set`, which is a `thread_port_t`. The
first step we call `pthread_mach_thread_np` to get a `thread_port_t` form
`pthread_t`. The core number that the thread should be bound to is provided
in the `cpu_set` parameters. The `CPU_SET(int core_num, cpu_set_t *cs)`
is called to set the core number in the `cpu_set_t` data structure.

{% highlight c %}

int pthread_setaffinity_np(pthread_t thread, size_t cpu_size,
                           cpu_set_t *cpu_set)
{
  thread_port_t mach_thread;
  int core = 0;

  for (core = 0; core < 8 * cpu_size; core++) {
    if (CPU_ISSET(core, cpu_set)) break;
  }
  printf("binding to core %d\n", core);
  thread_affinity_policy_data_t policy = { core };
  mach_thread = pthread_mach_thread_np(thread);
  thread_policy_set(mach_thread, THREAD_AFFINITY_POLICY,
                    (thread_policy_t)&policy, 1);
  return 0;
}
{% endhighlight %}

Overall, the implementations for these two missing system calls are quite
simple with limited power. Full implementations complied with the POSIX
sematics are beyond the scope of this blog, and I do not really need all
the functions. If I do, I will update them accordingly :-).



[1]: https://github.com/yyshen/mbw
