# Causes of Concurrency in Userland
- pseudo-concurreny: two things do not actually happen at the same time but interleave with each other
- true concurreny: two processes are executed in a critical region at the exact same time.

# Causes of Concurrency in Kernel
- Interrupts
- Softirq and tasklets
- Kernel preemption
- Sleeping and synchronization with user-space
- Symmetrical multiprocessing

# xxx-safe
- interrupt-safe: Code that is safe from concurrent access from an interrupt handler
- SMP-safe: Code that is safe from concurrency on symmetrical multiprocessing machines
- preempt-safe: Code that is safe from concurrency with kernel preemption

# Whenever you write kernel code, you should ask yourself these questions:
- Is the data global? Can a thread of execution other than the current one access it?
- Is the data shared between process context and interrupt context? Is it shared between two different interrupt handlers?
- If a process is preempted while accessing this data, can the newly scheduled process access the same data?
- Can the current process sleep (block) on anything? If it does, in what state does that leave any shared data?
- What prevents the data from being freed out from under me?
- What happens if this function is called again on another processor?
- Given the proceeding points, how am I going to ensure that my code is safe from concurrency?

# lock contention
The term lock contention, or simply contention, describes a lock currently in use but that
another thread is trying to acquire.A lock that is highly contended often has threads waiting
to acquire it. High contention can occur because a lock is frequently obtained, held for a
long time after it is obtained, or both.

# Atomic operations
- Atomic Integer Operations
- Atomic Bitwise Operations
## Atomic Integer Operations
- `atomic_t` as operand

   `typedef struct {
      int counter;
   } atomic_t;`

   + type checking for atomic operation functions
   + prevent compiler's optimization, such as, alias
   + hide architecture-specific details for implementation
- `atomic_t` has 32 bits
- on most architecture, word-size read is always atomic

|Atomic Integer Operation                     |Description                                                                          |
:---------------------------------------------|:-------------------------------------------------------------------------------------
|`ATOMIC_INIT(int i)`                         | At declaration, initialize to i.                                                    |
|`int atomic_read(atomic_t *v)`               | Atomically read the integer value of v.                                             |
|`void atomic_set(atomic_t *v, int i)`        | Atomically set v equal to i.                                                        |
|`void atomic_add(int i, atomic_t *v)`        | Atomically add i to v.                                                              |
|`void atomic_sub(int i, atomic_t *v)`        | Atomically subtract i from v.                                                       |
|`void atomic_inc(atomic_t *v)`               | Atomically add one to v.                                                            |
|`void atomic_dec(atomic_t *v)`               | Atomically subtract one from v.                                                     |
|`int atomic_sub_and_test(int i, atomic_t *v)`| Atomically subtract i from v and return true if the result is zero; otherwise false.|
|`int atomic_add_negative(int i, atomic_t *v)`| Atomically add i to v and return true if the result is negative; otherwise false.   |
|`int atomic_add_return(int i, atomic_t *v)`  | Atomically add i to v and return the result.                                        |
|`int atomic_sub_return(int i, atomic_t *v)`  | Atomically subtract i from v and return the result.                                 |
|`int atomic_inc_return(int i, atomic_t *v)`  | Atomically increment v by one and return the result.                                |
|`int atomic_dec_return(int i, atomic_t *v)`  | Atomically decrement v by one and return the result.                                |
|`int atomic_dec_and_test(atomic_t *v)`       | Atomically decrement v by one and return true if zero; false otherwise.             |
|`int atomic_inc_and_test(atomic_t *v)`       | Atomically increment v by one and return true if the result is zero; false otherwise.|

## Atomic and Ordering
- Atomicity ensures that instructions occur without interruption and that they
complete either in their entirety or not at all. 
- Ordering, on the other hand, ensures that the
desired, relative ordering of two or more instructions—even if they are to occur in separate
threads of execution or even separate processors—is preserved.

## 64-Bit Atomic Operations
`typedef struct {
	long counter;
} atomic64_t;`

Same as atomic integer, only operations on 64-bit and return `long` as its value. All operations have `atomic64_` prefix.

## Atomic Bitwise Operations
All bitwise operations are in `<asm/bitops.h>`

The bitop functions are defined to work on unsigned longs:
- x64 system the bits numbered:
   + |0..............63|64............127|128...........191|192...........255|
- ppc64 system the bits numbered:
   + |63..............0|127............64|191...........128|255...........192|
- and on ppc32:
   + |31.....0|63....32|95....64|127...96|159..128|191..160|223..192|255..224|

|Atomic Bitwise Operation                       |Description                                                                        |
:-----------------------------------------------|:-----------------------------------------------------------------------------------
|`void set_bit(int nr, void *addr)`             | Atomically set the nr -th bit starting from addr.                                 |
|`void clear_bit(int nr, void *addr)`           | Atomically clear the nr -th bit starting from addr.                               |
|`void change_bit(int nr, void *addr)`          | Atomically flip the value of the nr -th bit starting from addr.                   |
|`int test_and_set_bit(int nr, void *addr)`     | Atomically set the nr -th bit starting from addr and return the previous value.   |
|`int test_and_clear_bit(int nr, void *addr)`   | Atomically clear the nr -th bit starting from addr and return the previous value. |
|`int test_and_change_bit(int nr, void *addr)`  | Atomically flip the nr -th bit starting from addr and return the previous value.  |
|`int test_bit(int nr, void *addr)`             | Atomically return the value of the nr - th bit starting from addr.                |

`nr` is the bit number, and there are no limitations on the bit number supplied
i. e. `nr` can be greater than word-size on the architecture, but usually we
do not use `nr` greater than word-size.

Atomic bitwise operations and their nonatomic versions are the only way for platform independent bitmap.

# Locks
## Spin Lock
- Spin Lock will disable kernel *PREEMPTION* before acquire lock, and enable it after release lock.
- If acquiring a lock without preemption disabled, the process holding the lock may lose its cpu to other process acquiring that lock too, and this will cause a unexpected long busy-wait.
- On uniprocessor machine, `spin_lock` simply disables kernel preemption
- `spin_lock_irqsave` and `spin_unlock_irqstore` are implemented partially as macros, so the argument acts like passing by reference.
- In `spin_ulock_irq*`, irq is enabled first, otherwise, `preempt_enable` may cause reschedule and switch to another process with irq disabled.
- `CONFIG_DEBUG_SPINLOCK` enables a handful of debugging checks in the spin lock code.
### `spin_lock`
1. disable preemption
2. acquire lock (try lock and spin)
### `spin_unlock`
1. release lock
2. enable preemption
### `spin_lock_irq`
1. disable local interrupt
2. disable preemption
3. acquire lock (try lock and spin)
### `spin_unlock_irq`
1. release lock
2. enable local interrupt
3. enable preemption
### `spin_lock_irqsave` 
1. disable local interrupt and save the interrupt flag
2. disable preemption
3. acquire lock (try lock and spin)
### `spin_unlock_irqrestore` 
1. release lock
2. enable local interrupt and restore the interrupt flag
3. enable preemption
