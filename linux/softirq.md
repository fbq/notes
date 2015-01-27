# Others
percpu data on x86:
   use a segment register to store the offset of each per cpu data

# Softirq
Time when softirq execute:

   - In the return from hardware interrupt code path
   - In the ksoftirqd kernel thread
   - In any code that explicitly checks for and executes pending softirqs, such as the net-
working subsystem  

function `__do_softirq`:

   - iterate, check and run softirq handler
   - called by
    * `do_softirq_own_stack`, which call by `do_softirq` and `invoke_softirq`
    * `invoke_softirq`, when return from hardirq code path
    * `run_ksoftirqd`, `thread_fn` for ksoftirqd
   - called with local irq disabled, gauranteed by caller
   - softirq handlers run with local irq enable, and the same softirq handler can run on other cpus simultaneously
   - softirq handlers cannot sleep

irq that interrupt softirq:
   does NOT call `invoke_softirq` to handle new pending softirq.

   Example:

   - when a cpu is running happily in a user process, an irq occurs(`preempt_count_add(HARDIRQ_OFFSET)`) and does some hardware
related work then call `invoke_softirq` to check and run softirqs, at this time, another irq comes(we call this ORIQ), which
can happen since interrupt is enabled in `__do_softirq`. ORIQ will find itself `in_interrupt` and does not `invoke_softirq`.
See `irq_enter` and `irq_exit` in `kernel/softirq.c` for details.
   - besides, even in process contexts, `__do_softirq` will also disable local bottom halves before softirq `action` via 
`__local_bh_disable_ip`, and enable later via `__local_bh_enable_ip`, and the {dis|en}able operations are called when local
irq disable, so local cpu will always seetheir affect.

   Conclusion:
   - softirq is NEVER nested, so is tasklet.

# Tasklet
State of tasklet:

   - state of tasklet is represented by bits in `state` field in `tasklet_struct`
   - `TASKLET_STATE_SCHED`: this tasklet is scheduled and can not be put on this or other cpu's tasklet queue
   - `TASKLET_STATE_RUN`: this tasklet is running. Even this tasklet is scheduled on other cpus(remember that tasklet can not be nested),
it can't run
   - a tasklet can be both SCHED and RUN(they are on different bits in `state`), or either and neither.
   - details at `tasklet_action` in `kernel/softirq.c`.

# Work queue
Which is totally different now, the kernel thread entry is still `worker_thread`, but `worker` and `worker_pool` is the new strategy.
`worker_pool` is group of workers by cpu. Ref `Documentation/workqueue.txt`

# API
## API for softirq

   - `open_softirq(int nr, void (*action)(struct softirq_action *))`, set the handler for softirq `nr`
   - `raise_softirq(unsigned int nr)`, issue the `nr` softirq
   - `raise_softirq_irqoff(unsigned int nr)`, issue the `nr` softirq in a context that interrupt is already disabled
   - `raise_softirq*` will wake up `ksoftirqd` if called in a process context

## API for tasklet

   - `DECLARE_TASKLET(name, func, data)`, declare a tasklet with `name` and call `func` with argument `data` when issued.
   - `DECLARE_TASKLET_DISABLE(name, func, data)`, declare a tasklet with `name` and call `func` with argument `data` when issued. Disabled defaultly 
   - `tasklet_init(struct tasklet_struct *t, void (*func)(unsigned long), unsigned long data)`, dynamic version for tasklet creation.
   - `tasklet_schedule(struct tasklet_struct *t)`, issue tasklet `t`
   - `tasklet_disable(struct tasklet_struct *t)`, disable tasklet `t`
   - `tasklet_enable(struct tasklet_struct *t)`, enable tasklet `t`
   - `tasklet_kill(struct tasklet_struct *t)`, wait until `t` finished and remove it from tasklet schedule queue, only in process contexts
   - `tasklet_kill_immediate(struct tasklet_struct *t, unsigned int cpu)`, remove `t` from schedule queue anyway, be sure that `t` is not running
and `cpu` is in `CPU_DEAD` state, no sleep

## API for workqueue
   - `DECLARE_WORK(n, f)`, declare a work with name `n` and handler `f`
   - `INIT_WORK(_work, _func)`, initialize `_work` with handler `_func`
   - `bool schedule_work(struct work_struct *work)`, issue `work` in system workqueue, false for already in queue
   - `bool schedule_delayed_work(struct delayed_work *dwork, unsigned long delay)`, issue `work` in system workqueue with `delay` jiffies
   - `flush_scheduled_work()`, wait util all works in system workqueue are done
   - `bool cancel_delayed_work(struct delayed_work *dwork)`, [TODO](need to know kernel timer)
   - `struct workqueue_struct * create_workqueue(name)`, create a workqueue with `name`
   - `bool queue_work(struct workqueue_struct *wq, struct work_struct *work)`, issue `work` in `wq`
   - `bool queue_delayed_work(struct workqueue_struct *wq, struct delayed_work *dwork, unsigned long delay)`, like `schedule_delayed_work`
   - `flush_workqueue(struct workqueue_struct *wq)`, like `flush_scheduled_work`

## API for control
   - `local_bh_disable()`, disable softirq on current cpu and therefore tasklet by adding `preempt_count` in `SOFTIRQ_DISABLE_OFFSET`
   - `local_bh_enable()`, enable softirq on current cpu and therefore tasklet and if there is a pending one and the local cpu is not `in_interrupt`, run it
   - these control api are nested, and like irq control, they are to prevent deadlock mainly.
