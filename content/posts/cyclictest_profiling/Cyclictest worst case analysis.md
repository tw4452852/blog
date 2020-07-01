+++
title = "Cyclictest worst case analysis"
date = "2020-06-22T14:27:00"
tags = ["linux", "profiling" ]
+++

## Capture the worst case with ftrace enabled

Here's a screenshot of what's happening around the worst case:

```
cyclicte-1213    1....1.. 2563694114us : sys_clock_nanosleep(which_clock: 1, flags: 1, rqtp: 7fea2bcbb900, rmtp: 0)
cyclicte-1213    1....... 2563694114us : hrtimer_init: hrtimer=000000002ad8bc6a clockid=CLOCK_MONOTONIC mode=ABS
cyclicte-1213    1d...1.. 2563694114us : hrtimer_start: hrtimer=000000002ad8bc6a function=hrtimer_wakeup expires=6389900935569 softexpires=6389900935569 mode=ABS <========= prepare to sleep, the next expected waking up time is 6389900935569.
...
  <idle>-0       1d..h1.. 2563695110us : local_timer_entry: vector=236 <=========== tick interrupt comes.
  <idle>-0       1d..h2.. 2563695118us : hrtimer_cancel: hrtimer=000000002ad8bc6a
  <idle>-0       1d..h1.. 2563695119us : hrtimer_expire_entry: hrtimer=000000002ad8bc6a function=hrtimer_wakeup now=6389900945120 <===== timer wakeup handler, current time is 6389900945120.
  <idle>-0       1d..h2.. 2563695119us : sched_waking: comm=cyclictest pid=1213 prio=19 target_cpu=001
  <idle>-0       1dN.h3.. 2563695122us : sched_wakeup: comm=cyclictest pid=1213 prio=19 target_cpu=001
  <idle>-0       1dN.h1.. 2563695122us : hrtimer_expire_exit: hrtimer=000000002ad8bc6a
  <idle>-0       1dN.h1.. 2563695123us : write_msr: 6e0, value 21d331afdacc
  <idle>-0       1dN.h1.. 2563695123us : local_timer_exit: vector=236
...
  <idle>-0       1d...2.. 2563695126us : sched_switch: prev_comm=swapper/1 prev_pid=0 prev_prio=120 prev_state=R ==> next_comm=cyclictest next_pid=1213 next_prio=19 <==== try to wake up cyclictest process
  <idle>-0       1d...2.. 2563695126us : tlb_flush: pages:0 reason:flush on task switch (0)
  <idle>-0       1d...2.. 2563695127us : write_msr: c0000100, value 7fea2bcbc700
  <idle>-0       1d...2.. 2563695127us : x86_fpu_regs_activated: x86/fpu: 00000000e2eb3eca initialized: 1 xfeatures: 2 xcomp_bv: 8000000000000007
cyclicte-1213    1....... 2563695128us : sys_exit: NR 230 = 0
cyclicte-1213    1....1.. 2563695128us : sys_clock_nanosleep -> 0x0 <==== clock_nanosleep syscall returns.

```

From above log, we can see the tick interrupt arrival time is basically right (2563695110 - 2563694114 = 996us, cyclictest sleep interval is 1000us), but timer wakeup handler was delayed around 9us (2563695119 - 2563695110 = 9), why?



## Capture the worst case with function_graph tracer enabled

To find out what's happening between timer wakeup handler and tick interrupt, I just enable function_graph tracer to compare between abnormal and normal cases, here's the screenshot:

Abnormal case:

```
1)               |  smp_apic_timer_interrupt() {
1)               |    irq_enter() {
1)   8.986 us    |    }
1)               |    hrtimer_interrupt() {
1)               |      ...
1)               |      __hrtimer_run_queues() {
1)               |        ...
1)               |        hrtimer_wakeup() {
1)               |          wake_up_process() {
1)               |            try_to_wake_up() {
1)               |              ...
1)               |              ttwu_do_activate() {
1)               |                activate_task() {
1)               |                  enqueue_task_rt() {
1)               |                    dequeue_rt_stack() {
1)   0.423 us    |                      dequeue_top_rt_rq();
1)   1.196 us    |                    }
1)   0.434 us    |                    cpupri_set();
1)   0.371 us    |                    update_rt_migration();
1)               |                    _raw_spin_lock() {
1)   0.384 us    |                      preempt_count_add();
1)   1.145 us    |                    }
1)   0.470 us    |              +---->ktime_get();
1)   0.394 us    |              |     hrtimer_forward();
1)               |              |     hrtimer_start_range_ns() {
1)               |              |       lock_hrtimer_base.isra.0() {
1)               |              |         _raw_spin_lock_irqsave() {
1)   0.396 us    |              |           preempt_count_add();
1)   1.211 us    |              |         }
1)   1.973 us    |              |       }
1)   0.382 us    | additional part      preempt_count_sub();
1)               |   (~7.5us)   |       _raw_spin_lock() {
1)   0.383 us    |              |         preempt_count_add();
1)   1.129 us    |              |       }
1)   0.412 us    |              |       enqueue_hrtimer();
1)               |              |       _raw_spin_unlock_irqrestore() {
1)   0.379 us    |              |         preempt_count_sub();
1)   1.170 us    |              |       }
1)   7.505 us    |              +---->}
1)   0.399 us    |                    preempt_count_sub();
1)               |                    enqueue_top_rt_rq() {
1)   0.377 us    |                      sched_can_stop_tick();
1)   0.380 us    |                      tick_nohz_dep_clear_cpu();
1)   1.943 us    |                    }
1) + 18.024 us   |                  }
1) + 18.785 us   |                }
1)               |                ...
1) + 22.754 us   |              }
1)               |              ...
1) + 29.839 us   |            }
1) + 30.588 us   |          }
1) + 31.336 us   |        }
1)               |        ...
1) + 36.056 us   |      }
1)               |      ...
1) + 46.372 us   |    }
1)               |    irq_exit() {
1)               |     ...
1)   6.826 us    |    }
1) + 63.996 us   |  }

```

Normal case:

```
1)               |  smp_apic_timer_interrupt() {
1)               |    irq_enter() {
1)               |      ...
1)   8.901 us    |    }
1)               |    hrtimer_interrupt() {
1)               |      ...
1)               |      __hrtimer_run_queues() {
1)               |        ...
1)               |        hrtimer_wakeup() {
1)               |          wake_up_process() {
1)               |            try_to_wake_up() {
1)               |              ...
1)               |              ttwu_do_activate() {
1)               |                activate_task() {
1)               |                  enqueue_task_rt() {
1)               |                    dequeue_rt_stack() {
1)   0.391 us    |                      dequeue_top_rt_rq();
1)   1.150 us    |                    }
1)   0.416 us    |                    cpupri_set();
1)   0.380 us    |                    update_rt_migration();
1)               |                    _raw_spin_lock() {
1)   0.385 us    |                      preempt_count_add();
1)   1.128 us    |                    }
1)   0.376 us    |                    preempt_count_sub();
1)               |                    enqueue_top_rt_rq() {
1)   0.373 us    |                      sched_can_stop_tick();
1)   0.366 us    |                      tick_nohz_dep_clear_cpu();
1)   1.889 us    |                    }
1)   8.032 us    |                  }
1)   8.811 us    |                }
1)               |                ...
1) + 12.782 us   |              }
1)               |              ...
1) + 19.906 us   |            }
1) + 20.648 us   |          }
1) + 21.393 us   |        }
1)               |        ...
1) + 26.086 us   |      }
1)               |      ...
1) + 36.506 us   |    }
1)               |    irq_exit() {
1)   0.429 us    |      ...
1)   6.732 us    |    }
1) + 53.940 us   |  }
```

After digging into the source code, the additional part in the trace is due to starting rt period throttling timer, the call trace is: `enqueue_task_rt->enqueue_rt_entity->__enqueue_rt_entity->inc_rt_tasks->inc_rt_group->start_rt_bandwidth` where if `rt_period_active == 0`, it will start the timer.

**Note:** Because rt task throttling is global for all cores, so any core can restart this timer, and all of throttling related stuff are protected by a global spin lock.

