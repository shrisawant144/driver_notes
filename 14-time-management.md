# Time Management in Linux Kernel

## 🎯 Layman's Explanation

**Why Time Management in Kernel?**
Drivers often need to:
- Wait for hardware to be ready
- Schedule periodic tasks
- Measure time intervals
- Implement timeouts

**Key Concepts:**

**1. Jiffies - The Kernel's Clock Tick**
```
Every few milliseconds: TICK!
Jiffies counter increments
```

**Analogy:**
- Jiffies = Heartbeat of the kernel
- Each tick = One heartbeat
- Count heartbeats to measure time

**Example:**
```c
unsigned long start = jiffies;
// Do something
unsigned long elapsed = jiffies - start;  // Time elapsed in ticks
```

**Typical values:**
- x86: 1 jiffy = 4ms (250 Hz)
- ARM: 1 jiffy = 10ms (100 Hz)

**2. Delays - Making Code Wait**

**Two Types:**

**a) Busy-Wait (CPU keeps checking "is time up?")**
```c
udelay(10);   // Wait 10 microseconds
mdelay(10);   // Wait 10 milliseconds
```

**Analogy:**
You're waiting for pizza:
- Busy-wait = Stand at door, keep checking every second
- CPU is BUSY, can't do other work

**Use when:** Very short delays (microseconds), in interrupt context

**b) Sleep (CPU does other work while waiting)**
```c
msleep(100);  // Sleep 100 milliseconds
ssleep(1);    // Sleep 1 second
```

**Analogy:**
- Sleep = Sit on couch, watch TV while waiting for pizza
- CPU can do other tasks

**Use when:** Longer delays, in process context (not interrupts!)

**3. Timers - Schedule Future Work**

**Concept:**
"Do this function after X milliseconds"

```c
Setup timer → Start it → Timer expires → Your function called
```

**Analogy:**
Kitchen timer:
- Set timer for 10 minutes
- Go do other things
- Timer rings → Check oven

**Example Use Cases:**
- Timeout for device response
- Periodic polling
- Watchdog timers
- LED blinking

**4. High-Resolution Timers (hrtimers)**

**Regular timers:**
- Resolution = 1 jiffy (4-10ms)
- Like a clock with only minute hand

**hrtimers:**
- Resolution = nanoseconds
- Like a stopwatch with millisecond precision

**Use when:** Need precise timing (audio, video, real-time control)

**5. Workqueues - Deferred Work**

**Concept:**
"Do this work later, when convenient"

```
Schedule work → Added to queue → Worker thread runs it later
```

**Analogy:**
- You write a to-do note
- Put it in inbox
- Later, when free, you process inbox

**Use when:** Work that can sleep, doesn't need exact timing

**Time Measurement - Different Scales:**

```
nanoseconds (ns)  = 0.000000001 sec  (very precise)
microseconds (μs) = 0.000001 sec     (precise)
milliseconds (ms) = 0.001 sec        (common)
jiffies           = ~0.004 sec       (kernel tick)
seconds (s)       = 1 sec            (coarse)
```

**Choosing the Right Tool:**

```
Need to wait?
├─ Very short (< 1ms) → udelay() [busy-wait]
├─ Short (1-10ms) → mdelay() or msleep()
└─ Long (> 10ms) → msleep() [sleep]

Need to schedule future work?
├─ Precise timing → hrtimer
├─ Approximate timing → kernel timer
└─ No timing requirement → workqueue

In interrupt context?
├─ Yes → udelay/mdelay only (can't sleep!)
└─ No → Can use sleep functions
```

**Common Patterns:**

**1. Timeout Pattern:**
```c
unsigned long timeout = jiffies + msecs_to_jiffies(1000);  // 1 sec timeout
while (!device_ready()) {
    if (time_after(jiffies, timeout)) {
        return -ETIMEDOUT;  // Timeout!
    }
    msleep(10);  // Check every 10ms
}
```

**Analogy:**
Waiting for friend, but only for 10 minutes max.

**2. Periodic Task Pattern:**
```c
Setup timer for 100ms
Timer expires → Do task → Restart timer for another 100ms
```

**Analogy:**
Checking mailbox every day.

**3. Debouncing Pattern:**
```c
Button pressed → Start timer (50ms)
Timer expires → Check if still pressed → Valid press!
```

**Analogy:**
Wait a moment to confirm button press (not accidental).

**Important Rules:**

1. **Never busy-wait for long** - Wastes CPU
2. **Can't sleep in interrupt context** - Use timers instead
3. **Use appropriate precision** - Don't use hrtimer if jiffy timer is enough
4. **Handle timer cleanup** - Delete timers when driver unloads

**Real-World Example - LED Blink Driver:**
```
Setup timer for 500ms
Timer expires:
  - Toggle LED
  - Restart timer for 500ms
Result: LED blinks every 500ms
```

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
