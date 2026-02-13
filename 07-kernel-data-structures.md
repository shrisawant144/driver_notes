# Kernel Data Structures

## Overview

The Linux kernel provides efficient, well-tested data structures for common programming tasks. Using these structures ensures compatibility, performance, and proper memory management.

## Linked Lists

### Structure

```c
#include <linux/list.h>

struct list_head {
    struct list_head *next, *prev;
};
```

### Defining a List

```c
struct my_data {
    int value;
    char name[32];
    struct list_head list;
};

// Static initialization
LIST_HEAD(my_list);

// Dynamic initialization
struct list_head my_list;
INIT_LIST_HEAD(&my_list);
```

### List Operations

```c
// Add to list
struct my_data *data = kmalloc(sizeof(*data), GFP_KERNEL);
data->value = 42;
strcpy(data->name, "example");
list_add(&data->list, &my_list);        // Add to head
list_add_tail(&data->list, &my_list);   // Add to tail

// Delete from list
list_del(&data->list);

// Check if empty
if (list_empty(&my_list))
    printk("List is empty\n");

// Replace entry
list_replace(&old->list, &new->list);

// Move entry
list_move(&data->list, &another_list);
list_move_tail(&data->list, &another_list);
```

### Iterating Over Lists

```c
struct my_data *entry;
struct list_head *pos;

// Basic iteration
list_for_each(pos, &my_list) {
    entry = list_entry(pos, struct my_data, list);
    printk("Value: %d, Name: %s\n", entry->value, entry->name);
}

// Simpler iteration
list_for_each_entry(entry, &my_list, list) {
    printk("Value: %d, Name: %s\n", entry->value, entry->name);
}

// Safe iteration (allows deletion)
struct my_data *tmp;
list_for_each_entry_safe(entry, tmp, &my_list, list) {
    if (entry->value == 0) {
        list_del(&entry->list);
        kfree(entry);
    }
}

// Reverse iteration
list_for_each_entry_reverse(entry, &my_list, list) {
    printk("Value: %d\n", entry->value);
}
```

### Complete Example

```c
#include <linux/module.h>
#include <linux/kernel.h>
#include <linux/list.h>
#include <linux/slab.h>

struct person {
    char name[32];
    int age;
    struct list_head list;
};

static LIST_HEAD(person_list);

static int __init list_init(void)
{
    struct person *p, *tmp;
    int i;
    char *names[] = {"Alice", "Bob", "Charlie"};
    
    // Add entries
    for (i = 0; i < 3; i++) {
        p = kmalloc(sizeof(*p), GFP_KERNEL);
        strcpy(p->name, names[i]);
        p->age = 20 + i * 5;
        list_add_tail(&p->list, &person_list);
    }
    
    // Print entries
    list_for_each_entry(p, &person_list, list) {
        printk(KERN_INFO "%s, age %d\n", p->name, p->age);
    }
    
    return 0;
}

static void __exit list_exit(void)
{
    struct person *p, *tmp;
    
    // Clean up
    list_for_each_entry_safe(p, tmp, &person_list, list) {
        list_del(&p->list);
        kfree(p);
    }
    
    printk(KERN_INFO "List cleaned up\n");
}

module_init(list_init);
module_exit(list_exit);
MODULE_LICENSE("GPL");
```

## Hash Lists (hlist)

Optimized for hash tables with single pointer head.

```c
#include <linux/list.h>

struct hlist_head {
    struct hlist_node *first;
};

struct hlist_node {
    struct hlist_node *next, **pprev;
};
```

### Usage

```c
#define HASH_SIZE 256

struct my_hash_entry {
    int key;
    char data[64];
    struct hlist_node node;
};

static struct hlist_head hash_table[HASH_SIZE];

static void hash_init(void)
{
    int i;
    for (i = 0; i < HASH_SIZE; i++)
        INIT_HLIST_HEAD(&hash_table[i]);
}

static unsigned int hash_func(int key)
{
    return key % HASH_SIZE;
}

static void hash_add(int key, const char *data)
{
    struct my_hash_entry *entry;
    unsigned int hash;
    
    entry = kmalloc(sizeof(*entry), GFP_KERNEL);
    entry->key = key;
    strcpy(entry->data, data);
    
    hash = hash_func(key);
    hlist_add_head(&entry->node, &hash_table[hash]);
}

static struct my_hash_entry *hash_find(int key)
{
    struct my_hash_entry *entry;
    unsigned int hash = hash_func(key);
    
    hlist_for_each_entry(entry, &hash_table[hash], node) {
        if (entry->key == key)
            return entry;
    }
    
    return NULL;
}
```

## Queues (kfifo)

Circular buffer implementation.

```c
#include <linux/kfifo.h>

// Define kfifo
DEFINE_KFIFO(my_fifo, char, 128);

// Or dynamic allocation
struct kfifo my_fifo;
kfifo_alloc(&my_fifo, 128, GFP_KERNEL);

// Put data
char data[] = "Hello";
kfifo_in(&my_fifo, data, strlen(data));

// Get data
char buffer[64];
int len = kfifo_out(&my_fifo, buffer, sizeof(buffer));

// Peek without removing
len = kfifo_out_peek(&my_fifo, buffer, sizeof(buffer));

// Check status
if (kfifo_is_empty(&my_fifo))
    printk("FIFO is empty\n");

if (kfifo_is_full(&my_fifo))
    printk("FIFO is full\n");

unsigned int avail = kfifo_avail(&my_fifo);
unsigned int len = kfifo_len(&my_fifo);

// Reset
kfifo_reset(&my_fifo);

// Free
kfifo_free(&my_fifo);
```

## Red-Black Trees

Self-balancing binary search tree.

```c
#include <linux/rbtree.h>

struct my_node {
    int key;
    char data[64];
    struct rb_node node;
};

static struct rb_root my_tree = RB_ROOT;

static int my_insert(struct rb_root *root, struct my_node *data)
{
    struct rb_node **new = &(root->rb_node), *parent = NULL;
    
    while (*new) {
        struct my_node *this = rb_entry(*new, struct my_node, node);
        
        parent = *new;
        if (data->key < this->key)
            new = &((*new)->rb_left);
        else if (data->key > this->key)
            new = &((*new)->rb_right);
        else
            return -1;  // Duplicate
    }
    
    rb_link_node(&data->node, parent, new);
    rb_insert_color(&data->node, root);
    
    return 0;
}

static struct my_node *my_search(struct rb_root *root, int key)
{
    struct rb_node *node = root->rb_node;
    
    while (node) {
        struct my_node *data = rb_entry(node, struct my_node, node);
        
        if (key < data->key)
            node = node->rb_left;
        else if (key > data->key)
            node = node->rb_right;
        else
            return data;
    }
    
    return NULL;
}

static void my_delete(struct rb_root *root, struct my_node *data)
{
    rb_erase(&data->node, root);
}

// Iterate
struct rb_node *node;
for (node = rb_first(&my_tree); node; node = rb_next(node)) {
    struct my_node *data = rb_entry(node, struct my_node, node);
    printk("Key: %d\n", data->key);
}
```

## Bitmaps

Efficient bit manipulation.

```c
#include <linux/bitmap.h>

#define BITS 256

DECLARE_BITMAP(my_bitmap, BITS);

// Set bit
set_bit(5, my_bitmap);

// Clear bit
clear_bit(5, my_bitmap);

// Test bit
if (test_bit(5, my_bitmap))
    printk("Bit 5 is set\n");

// Test and set
if (test_and_set_bit(5, my_bitmap))
    printk("Bit was already set\n");

// Find first zero bit
int bit = find_first_zero_bit(my_bitmap, BITS);

// Find next set bit
bit = find_next_bit(my_bitmap, BITS, 0);

// Clear all bits
bitmap_zero(my_bitmap, BITS);

// Set all bits
bitmap_fill(my_bitmap, BITS);

// Copy bitmap
DECLARE_BITMAP(copy, BITS);
bitmap_copy(copy, my_bitmap, BITS);
```

## IDR (ID Radix Tree)

Maps integers to pointers.

```c
#include <linux/idr.h>

static DEFINE_IDR(my_idr);

// Allocate ID and store pointer
struct my_data *data = kmalloc(sizeof(*data), GFP_KERNEL);
int id = idr_alloc(&my_idr, data, 0, 0, GFP_KERNEL);
if (id < 0) {
    kfree(data);
    return id;
}

// Lookup
data = idr_find(&my_idr, id);

// Remove
idr_remove(&my_idr, id);

// Iterate
int id;
struct my_data *entry;
idr_for_each_entry(&my_idr, entry, id) {
    printk("ID: %d\n", id);
}

// Destroy
idr_destroy(&my_idr);
```

## Work Queues

Deferred work execution.

```c
#include <linux/workqueue.h>

struct my_work {
    struct work_struct work;
    int data;
};

static void my_work_handler(struct work_struct *work)
{
    struct my_work *my = container_of(work, struct my_work, work);
    printk("Work executed with data: %d\n", my->data);
    kfree(my);
}

// Schedule work
struct my_work *work = kmalloc(sizeof(*work), GFP_KERNEL);
work->data = 42;
INIT_WORK(&work->work, my_work_handler);
schedule_work(&work->work);

// Delayed work
DECLARE_DELAYED_WORK(my_delayed_work, my_work_handler);
schedule_delayed_work(&my_delayed_work, msecs_to_jiffies(1000));

// Cancel work
cancel_work_sync(&work->work);
cancel_delayed_work_sync(&my_delayed_work);
```

## Kernel Linked List Example

```c
#include <linux/module.h>
#include <linux/kernel.h>
#include <linux/list.h>
#include <linux/slab.h>

struct student {
    int id;
    char name[32];
    int grade;
    struct list_head list;
};

static LIST_HEAD(student_list);

static void add_student(int id, const char *name, int grade)
{
    struct student *s = kmalloc(sizeof(*s), GFP_KERNEL);
    s->id = id;
    strcpy(s->name, name);
    s->grade = grade;
    list_add_tail(&s->list, &student_list);
}

static void print_students(void)
{
    struct student *s;
    printk(KERN_INFO "Student List:\n");
    list_for_each_entry(s, &student_list, list) {
        printk(KERN_INFO "ID: %d, Name: %s, Grade: %d\n",
               s->id, s->name, s->grade);
    }
}

static void remove_student(int id)
{
    struct student *s, *tmp;
    list_for_each_entry_safe(s, tmp, &student_list, list) {
        if (s->id == id) {
            list_del(&s->list);
            kfree(s);
            printk(KERN_INFO "Removed student ID: %d\n", id);
            return;
        }
    }
}

static void cleanup_students(void)
{
    struct student *s, *tmp;
    list_for_each_entry_safe(s, tmp, &student_list, list) {
        list_del(&s->list);
        kfree(s);
    }
}

static int __init data_struct_init(void)
{
    add_student(1, "Alice", 85);
    add_student(2, "Bob", 90);
    add_student(3, "Charlie", 78);
    
    print_students();
    
    remove_student(2);
    
    print_students();
    
    return 0;
}

static void __exit data_struct_exit(void)
{
    cleanup_students();
    printk(KERN_INFO "Module unloaded\n");
}

module_init(data_struct_init);
module_exit(data_struct_exit);

MODULE_LICENSE("GPL");
MODULE_DESCRIPTION("Kernel data structures example");
```

## Comparison of Data Structures

| Structure | Use Case | Time Complexity |
|-----------|----------|-----------------|
| Linked List | Sequential access, frequent insertion/deletion | O(n) search, O(1) insert/delete |
| Hash List | Fast lookup by key | O(1) average |
| Red-Black Tree | Sorted data, range queries | O(log n) |
| Bitmap | Bit flags, resource allocation | O(1) per bit |
| kfifo | FIFO queue, circular buffer | O(1) |
| IDR | Integer to pointer mapping | O(log n) |

## Best Practices

1. **Use kernel structures** instead of implementing your own
2. **Choose appropriate structure** for your use case
3. **Protect shared structures** with locks
4. **Free memory** properly to avoid leaks
5. **Use safe iterators** when modifying lists
6. **Initialize structures** before use
7. **Consider cache locality** for performance

## Next Steps

Proceed to [Interrupt Handling](08-interrupt-handling.md) to learn about handling hardware interrupts.
