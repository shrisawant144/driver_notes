# Time Management in Linux Kernel

## Overview

Time management in the kernel involves timers, delays, and time-related operations for driver development.

## Jiffies

Global variable tracking timer ticks since boot:

```c
#include <linux/jiffies.h>

unsigned long j = jiffies;  // Current jiffies value
```

Conversion macros:
```c
msecs_to_jiffies(ms)
jiffies_to_msecs(j)
usecs_to_jiffies(us)
jiffies_to_usecs(j)
```

## Delays

### Busy-Wait Delays

```c
#include <linux/delay.h>

udelay(microseconds);   // Microsecond delay
mdelay(milliseconds);   // Millisecond delay
ndelay(nanoseconds);    // Nanosecond delay
```

### Sleep Delays

```c
#include <linux/delay.h>

msleep(milliseconds);           // Uninterruptible sleep
msleep_interruptible(milliseconds);  // Interruptible sleep
ssleep(seconds);                // Second sleep
```

## Kernel Timers

### Timer Structure

```c
#include <linux/timer.h>

struct timer_list my_timer;

void timer_callback(struct timer_list *t)
{
    pr_info("Timer expired\n");
    // Restart timer if needed
    mod_timer(t, jiffies + msecs_to_jiffies(1000));
}

// Initialize timer
timer_setup(&my_timer, timer_callback, 0);
my_timer.expires = jiffies + msecs_to_jiffies(1000);
add_timer(&my_timer);

// Modify timer
mod_timer(&my_timer, jiffies + msecs_to_jiffies(2000));

// Delete timer
del_timer(&my_timer);
del_timer_sync(&my_timer);  // Wait for completion
```

## High-Resolution Timers

```c
#include <linux/hrtimer.h>
#include <linux/ktime.h>

struct hrtimer hr_timer;

enum hrtimer_restart hrtimer_callback(struct hrtimer *timer)
{
    pr_info("HR Timer expired\n");
    return HRTIMER_NORESTART;  // or HRTIMER_RESTART
}

// Initialize
hrtimer_init(&hr_timer, CLOCK_MONOTONIC, HRTIMER_MODE_REL);
hr_timer.function = hrtimer_callback;

// Start timer (1 second)
ktime_t ktime = ktime_set(1, 0);  // seconds, nanoseconds
hrtimer_start(&hr_timer, ktime, HRTIMER_MODE_REL);

// Cancel timer
hrtimer_cancel(&hr_timer);
```

## Work Queues

Deferred work execution:

```c
#include <linux/workqueue.h>

struct work_struct my_work;

void work_handler(struct work_struct *work)
{
    pr_info("Work executed\n");
}

// Initialize
INIT_WORK(&my_work, work_handler);

// Schedule work
schedule_work(&my_work);

// Delayed work
struct delayed_work my_delayed_work;
INIT_DELAYED_WORK(&my_delayed_work, work_handler);
schedule_delayed_work(&my_delayed_work, msecs_to_jiffies(1000));

// Cancel work
cancel_work_sync(&my_work);
cancel_delayed_work_sync(&my_delayed_work);
```

## Getting Current Time

```c
#include <linux/timekeeping.h>

// Wall clock time
struct timespec64 ts;
ktime_get_real_ts64(&ts);

// Monotonic time
ktime_get_ts64(&ts);

// Using ktime
ktime_t kt = ktime_get();
s64 ns = ktime_to_ns(kt);
```

## Example: Timer-Based Driver

```c
#include <linux/module.h>
#include <linux/timer.h>

static struct timer_list my_timer;
static int count = 0;

static void timer_callback(struct timer_list *t)
{
    pr_info("Timer fired: count = %d\n", count++);
    mod_timer(&my_timer, jiffies + msecs_to_jiffies(1000));
}

static int __init timer_init(void)
{
    timer_setup(&my_timer, timer_callback, 0);
    mod_timer(&my_timer, jiffies + msecs_to_jiffies(1000));
    pr_info("Timer module loaded\n");
    return 0;
}

static void __exit timer_exit(void)
{
    del_timer_sync(&my_timer);
    pr_info("Timer module unloaded\n");
}

module_init(timer_init);
module_exit(timer_exit);
MODULE_LICENSE("GPL");
```

## Best Practices

- Use sleep functions in process context
- Use busy-wait only for very short delays (<10μs)
- Always cleanup timers in exit functions
- Use `del_timer_sync()` to ensure timer completion
- Prefer high-resolution timers for precise timing
- Use work queues for non-atomic operations

## Common Pitfalls

- Calling sleep functions in atomic context
- Forgetting to delete timers on module unload
- Using busy-wait for long delays
- Not handling timer races properly
