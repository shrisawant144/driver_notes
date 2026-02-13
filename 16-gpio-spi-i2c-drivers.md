# GPIO, SPI & I2C Device Drivers

## 🎯 Layman's Explanation

These are the three most common ways embedded devices talk to sensors, displays, and other chips. Let's understand each:

---

## GPIO (General Purpose Input/Output)

**What is GPIO?**
GPIO pins are like **light switches** - you can turn them ON (high/1) or OFF (low/0), or read their state.

**Real-World Analogy:**
- GPIO = Light switch on your wall
- Output mode = You control the switch (turn LED on/off)
- Input mode = You read the switch (is button pressed?)

**Common Uses:**
- 💡 Control LEDs
- 🔘 Read buttons
- 🔊 Control buzzers
- 🔌 Enable/disable chips
- 📡 Simple signaling

**How It Works:**
```
GPIO Pin
  ├─ Output Mode: You set HIGH (3.3V) or LOW (0V)
  │    Example: Turn LED on/off
  └─ Input Mode: You read HIGH or LOW
       Example: Is button pressed?
```

**Example - Blinking LED:**
```
1. Set GPIO as output
2. Set GPIO HIGH → LED on
3. Wait 500ms
4. Set GPIO LOW → LED off
5. Repeat
```

**Analogy:**
Like a **digital on/off switch** - no in-between, just 0 or 1.

**GPIO Interrupt:**
Instead of constantly checking "is button pressed?", you can say:
"Tell me WHEN button is pressed"

```
Setup interrupt on GPIO
    ↓
Button pressed → Interrupt fires → Your function called
```

**Analogy:**
- Polling = Constantly asking "Are we there yet?"
- Interrupt = "I'll tell you when we arrive"

---

## I2C (Inter-Integrated Circuit)

**What is I2C?**
A **two-wire communication protocol** for talking to sensors and chips. Think of it as a **party line phone** - multiple devices share the same wires.

**The Two Wires:**
- **SDA** (Serial Data) - Data travels here
- **SCL** (Serial Clock) - Timing signal

**Real-World Analogy:**
- I2C = Conference call
- Master = Meeting host (usually your CPU)
- Slaves = Participants (sensors, displays)
- Address = Phone number (each device has unique address)

**How It Works:**
```
Master: "Hey device at address 0x50, are you there?"
Slave: "Yes!"
Master: "Give me temperature reading"
Slave: "It's 25°C"
```

**Common I2C Devices:**
- 🌡️ Temperature sensors
- 📏 Accelerometers
- 🖥️ OLED displays
- ⏰ Real-time clocks
- 💾 EEPROMs

**I2C Transaction:**
```
START → Address → Read/Write → Data → ACK → STOP
```

**Analogy:**
1. START = "Attention everyone!"
2. Address = "I'm talking to device #5"
3. Data = "Here's the message"
4. ACK = "Got it!"
5. STOP = "Conversation over"

**Advantages:**
- Only 2 wires (simple!)
- Multiple devices on same bus
- Standardized protocol

**Disadvantages:**
- Slower than SPI
- Limited distance (~1 meter)

**Speed:**
- Standard: 100 kHz
- Fast: 400 kHz
- High-speed: 3.4 MHz

---

## SPI (Serial Peripheral Interface)

**What is SPI?**
A **four-wire communication protocol** for high-speed data transfer. Think of it as a **dedicated phone line** - faster but needs more wires.

**The Four Wires:**
- **MOSI** (Master Out Slave In) - Master → Slave data
- **MISO** (Master In Slave Out) - Slave → Master data
- **SCLK** (Serial Clock) - Timing signal
- **CS** (Chip Select) - Choose which device to talk to

**Real-World Analogy:**
- SPI = Private phone lines
- Master = Boss
- Slaves = Employees
- CS = Calling specific employee
- MOSI/MISO = Two-way conversation (full duplex!)

**How It Works:**
```
Master pulls CS low → "I'm talking to YOU"
Master sends data on MOSI
Slave sends data on MISO (simultaneously!)
Master pulls CS high → "Done talking"
```

**Common SPI Devices:**
- 📟 LCD displays
- 💾 SD cards
- 📡 RF modules
- 🔊 Audio codecs
- 🎮 Sensors (high-speed)

**SPI vs I2C:**
```
SPI                          I2C
───                          ───
4+ wires                     2 wires
Faster (MHz)                 Slower (kHz)
Full duplex                  Half duplex
No addressing                Addressing
Point-to-point               Multi-drop bus
```

**Analogy:**
- **I2C** = Shared taxi (slower, but efficient for multiple stops)
- **SPI** = Private car (faster, but need separate car for each destination)

**SPI Modes:**
Different devices expect different clock behavior:
- Mode 0: Clock idle low, sample on rising edge
- Mode 1: Clock idle low, sample on falling edge
- Mode 2: Clock idle high, sample on falling edge
- Mode 3: Clock idle high, sample on rising edge

**Analogy:**
Like different handshake styles - both parties must agree!

---

## Comparison Table

```
Feature          GPIO        I2C           SPI
───────          ────        ───           ───
Wires            1 per pin   2 (shared)    4+ (per device)
Speed            N/A         100-400 kHz   MHz
Complexity       Simple      Medium        Medium
Use Case         On/Off      Sensors       High-speed data
Distance         Short       ~1m           ~10cm
Multi-device     Many pins   Yes (bus)     Yes (CS pins)
```

---

## When to Use What?

**Use GPIO when:**
- Simple on/off control
- Reading button states
- Controlling LEDs
- Chip enable signals

**Use I2C when:**
- Multiple sensors
- Limited pins available
- Moderate speed OK
- Standard sensors (most use I2C)

**Use SPI when:**
- High-speed needed
- Display updates
- SD card access
- Audio streaming
- Pins available

---

## Real-World Example - Smart Thermostat

```
GPIO:
  - LED indicators (on/off)
  - Button inputs (user presses)
  - Relay control (heater on/off)

I2C:
  - Temperature sensor (read temp)
  - Humidity sensor (read humidity)
  - OLED display (show info)

SPI:
  - SD card (log data)
  - WiFi module (fast communication)
```

---

## The Big Picture

```
CPU/SoC
  ├─ GPIO pins → LEDs, Buttons, Simple control
  ├─ I2C bus → Sensors, Small displays, RTCs
  └─ SPI bus → High-speed devices, SD cards, Displays
```

**Analogy:**
Your home communication:
- **GPIO** = Light switches (simple on/off)
- **I2C** = Intercom system (talk to multiple rooms, shared line)
- **SPI** = Direct phone lines (fast, dedicated lines)

---

## Key Takeaways

1. **GPIO** = Digital on/off, simplest
2. **I2C** = 2-wire bus, multiple devices, slower
3. **SPI** = 4-wire, faster, point-to-point
4. Choose based on speed needs and pin availability
5. Most embedded projects use all three!

## GPIO (General Purpose Input/Output)

### GPIO Descriptor Interface

```c
#include <linux/gpio/consumer.h>

struct gpio_desc *gpio;

// Get GPIO
gpio = gpiod_get(&pdev->dev, "reset", GPIOD_OUT_LOW);
if (IS_ERR(gpio))
    return PTR_ERR(gpio);

// Set value
gpiod_set_value(gpio, 1);  // High
gpiod_set_value(gpio, 0);  // Low

// Get value
int val = gpiod_get_value(gpio);

// Direction
gpiod_direction_input(gpio);
gpiod_direction_output(gpio, 0);

// Release
gpiod_put(gpio);
```

### Legacy GPIO Interface

```c
#include <linux/gpio.h>

int gpio_num = 17;

// Request GPIO
if (gpio_request(gpio_num, "my-gpio") < 0)
    return -EBUSY;

// Direction
gpio_direction_output(gpio_num, 0);
gpio_direction_input(gpio_num);

// Set/Get value
gpio_set_value(gpio_num, 1);
int val = gpio_get_value(gpio_num);

// Free GPIO
gpio_free(gpio_num);
```

### GPIO Interrupts

```c
int irq = gpiod_to_irq(gpio);

irqreturn_t gpio_irq_handler(int irq, void *dev_id)
{
    pr_info("GPIO interrupt triggered\n");
    return IRQ_HANDLED;
}

request_irq(irq, gpio_irq_handler,
            IRQF_TRIGGER_RISING | IRQF_TRIGGER_FALLING,
            "gpio-irq", dev);

// Cleanup
free_irq(irq, dev);
```

## I2C Drivers

### I2C Driver Structure

```c
#include <linux/i2c.h>

static const struct i2c_device_id device_id[] = {
    { "my_device", 0 },
    { }
};
MODULE_DEVICE_TABLE(i2c, device_id);

static const struct of_device_id device_of_match[] = {
    { .compatible = "vendor,my-device" },
    { }
};
MODULE_DEVICE_TABLE(of, device_of_match);

static int device_probe(struct i2c_client *client,
                        const struct i2c_device_id *id)
{
    pr_info("I2C device probed: addr=0x%02x\n", client->addr);
    return 0;
}

static void device_remove(struct i2c_client *client)
{
    pr_info("I2C device removed\n");
}

static struct i2c_driver device_driver = {
    .driver = {
        .name = "my_i2c_driver",
        .of_match_table = device_of_match,
    },
    .probe = device_probe,
    .remove = device_remove,
    .id_table = device_id,
};

module_i2c_driver(device_driver);
```

### I2C Read/Write Operations

```c
// Write byte
int i2c_smbus_write_byte_data(struct i2c_client *client,
                               u8 command, u8 value);

// Read byte
s32 i2c_smbus_read_byte_data(struct i2c_client *client, u8 command);

// Write word
int i2c_smbus_write_word_data(struct i2c_client *client,
                               u8 command, u16 value);

// Read word
s32 i2c_smbus_read_word_data(struct i2c_client *client, u8 command);

// Block write
int i2c_smbus_write_i2c_block_data(struct i2c_client *client,
                                    u8 command, u8 length,
                                    const u8 *values);

// Block read
int i2c_smbus_read_i2c_block_data(struct i2c_client *client,
                                   u8 command, u8 length, u8 *values);
```

### I2C Transfer

```c
struct i2c_msg msgs[2];
u8 reg = 0x10;
u8 data[4];

// Write then read
msgs[0].addr = client->addr;
msgs[0].flags = 0;  // Write
msgs[0].len = 1;
msgs[0].buf = &reg;

msgs[1].addr = client->addr;
msgs[1].flags = I2C_M_RD;  // Read
msgs[1].len = sizeof(data);
msgs[1].buf = data;

int ret = i2c_transfer(client->adapter, msgs, 2);
```

### Example: I2C EEPROM Driver

```c
#include <linux/module.h>
#include <linux/i2c.h>

struct eeprom_dev {
    struct i2c_client *client;
};

static ssize_t eeprom_read(struct file *file, char __user *buf,
                           size_t count, loff_t *offset)
{
    struct eeprom_dev *dev = file->private_data;
    u8 data[256];
    int ret;
    
    if (*offset >= 256)
        return 0;
    if (*offset + count > 256)
        count = 256 - *offset;
    
    ret = i2c_smbus_read_i2c_block_data(dev->client, *offset,
                                        count, data);
    if (ret < 0)
        return ret;
    
    if (copy_to_user(buf, data, count))
        return -EFAULT;
    
    *offset += count;
    return count;
}

static int eeprom_probe(struct i2c_client *client,
                        const struct i2c_device_id *id)
{
    struct eeprom_dev *dev;
    
    dev = devm_kzalloc(&client->dev, sizeof(*dev), GFP_KERNEL);
    if (!dev)
        return -ENOMEM;
    
    dev->client = client;
    i2c_set_clientdata(client, dev);
    
    pr_info("EEPROM device probed\n");
    return 0;
}

static void eeprom_remove(struct i2c_client *client)
{
    pr_info("EEPROM device removed\n");
}

static const struct i2c_device_id eeprom_id[] = {
    { "eeprom_24c02", 0 },
    { }
};
MODULE_DEVICE_TABLE(i2c, eeprom_id);

static struct i2c_driver eeprom_driver = {
    .driver = {
        .name = "eeprom_driver",
    },
    .probe = eeprom_probe,
    .remove = eeprom_remove,
    .id_table = eeprom_id,
};

module_i2c_driver(eeprom_driver);
MODULE_LICENSE("GPL");
```

## SPI Drivers

### SPI Driver Structure

```c
#include <linux/spi/spi.h>

static const struct of_device_id spi_of_match[] = {
    { .compatible = "vendor,my-spi-device" },
    { }
};
MODULE_DEVICE_TABLE(of, spi_of_match);

static int spi_probe(struct spi_device *spi)
{
    pr_info("SPI device probed: max_speed=%d Hz\n", spi->max_speed_hz);
    
    spi->mode = SPI_MODE_0;
    spi->bits_per_word = 8;
    spi_setup(spi);
    
    return 0;
}

static void spi_remove(struct spi_device *spi)
{
    pr_info("SPI device removed\n");
}

static struct spi_driver spi_driver = {
    .driver = {
        .name = "my_spi_driver",
        .of_match_table = spi_of_match,
    },
    .probe = spi_probe,
    .remove = spi_remove,
};

module_spi_driver(spi_driver);
```

### SPI Read/Write Operations

```c
// Simple write
int spi_write(struct spi_device *spi, const void *buf, size_t len);

// Simple read
int spi_read(struct spi_device *spi, void *buf, size_t len);

// Write then read
int spi_write_then_read(struct spi_device *spi,
                        const void *txbuf, unsigned n_tx,
                        void *rxbuf, unsigned n_rx);

// Synchronous transfer
struct spi_transfer xfer = {
    .tx_buf = tx_data,
    .rx_buf = rx_data,
    .len = len,
};
struct spi_message msg;

spi_message_init(&msg);
spi_message_add_tail(&xfer, &msg);
int ret = spi_sync(spi, &msg);
```

### Example: SPI Flash Driver

```c
#include <linux/module.h>
#include <linux/spi/spi.h>

#define CMD_READ  0x03
#define CMD_WRITE 0x02

struct spi_flash {
    struct spi_device *spi;
};

static int flash_read(struct spi_flash *flash, u32 addr,
                      void *buf, size_t len)
{
    u8 cmd[4];
    
    cmd[0] = CMD_READ;
    cmd[1] = (addr >> 16) & 0xFF;
    cmd[2] = (addr >> 8) & 0xFF;
    cmd[3] = addr & 0xFF;
    
    return spi_write_then_read(flash->spi, cmd, 4, buf, len);
}

static int flash_probe(struct spi_device *spi)
{
    struct spi_flash *flash;
    
    flash = devm_kzalloc(&spi->dev, sizeof(*flash), GFP_KERNEL);
    if (!flash)
        return -ENOMEM;
    
    flash->spi = spi;
    spi_set_drvdata(spi, flash);
    
    spi->mode = SPI_MODE_0;
    spi->bits_per_word = 8;
    spi_setup(spi);
    
    pr_info("SPI Flash probed\n");
    return 0;
}

static void flash_remove(struct spi_device *spi)
{
    pr_info("SPI Flash removed\n");
}

static const struct of_device_id flash_of_match[] = {
    { .compatible = "jedec,spi-nor" },
    { }
};
MODULE_DEVICE_TABLE(of, flash_of_match);

static struct spi_driver flash_driver = {
    .driver = {
        .name = "spi_flash",
        .of_match_table = flash_of_match,
    },
    .probe = flash_probe,
    .remove = flash_remove,
};

module_spi_driver(flash_driver);
MODULE_LICENSE("GPL");
```

## Device Tree Bindings

### GPIO in Device Tree

```dts
my_device {
    compatible = "vendor,my-device";
    reset-gpios = <&gpio 17 GPIO_ACTIVE_HIGH>;
    enable-gpios = <&gpio 27 GPIO_ACTIVE_LOW>;
};
```

### I2C in Device Tree

```dts
&i2c1 {
    status = "okay";
    
    my_device@50 {
        compatible = "vendor,my-device";
        reg = <0x50>;
    };
};
```

### SPI in Device Tree

```dts
&spi0 {
    status = "okay";
    
    my_device@0 {
        compatible = "vendor,my-spi-device";
        reg = <0>;
        spi-max-frequency = <1000000>;
    };
};
```

## Best Practices

- Use descriptor-based GPIO API for new code
- Always check return values
- Use devm_* functions for automatic cleanup
- Implement proper error handling
- Use device tree for hardware description
- Handle device removal gracefully
- Use appropriate transfer methods for performance
- Implement power management when needed

## Debugging

```bash
# GPIO
cat /sys/kernel/debug/gpio

# I2C
i2cdetect -y 1
i2cdump -y 1 0x50
i2cget -y 1 0x50 0x00

# SPI
cat /sys/class/spi_master/spi0/spi0.0/modalias
```
