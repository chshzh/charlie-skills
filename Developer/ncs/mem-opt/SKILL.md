````skill
---
name: ncs-mem
description: Optimize and debug memory usage in Nordic nRF Connect SDK (NCS) applications. Use when analyzing RAM/Flash usage, reducing memory footprint, debugging stack overflows, or investigating memory-related crashes in nRF projects.
---

# Nordic nRF Connect SDK (NCS) Memory Optimization and Debugging

## Key Concepts

**Memory Types in Embedded Systems:**
- **Flash (ROM)**: Stores code, constants, and read-only data
- **RAM**: Runtime memory for stack, heap, global/static variables
- **Stack**: Per-thread memory for function calls and local variables
- **Heap**: Dynamic memory allocation pool

**Memory Analysis Tools:**
- **west build size reports**: Detailed memory usage breakdown
- **Puncover**: Interactive memory visualization tool
- **GDB memory inspection**: Runtime memory analysis
- **Thread Analyzer**: Stack usage monitoring
- **Memory mapping**: Linker scripts and partition manager

## Critical Rules

1. **Always ensure NCS environment is set up** before memory analysis (use ncs-env-setup skill)
2. **Build in Release mode** for accurate Flash size measurements (Debug builds are larger)
3. **Enable memory debugging features only during development** (they consume extra memory)
4. **Check partition manager configuration** for multi-image builds

## Workflow for Memory Analysis

### Step 1: Build and Generate Memory Reports

Build the project and examine memory usage:
```sh
west build -b <board>
```

The build output shows memory summary:
```
Memory region         Used Size  Region Size  %age Used
           FLASH:      123456 B       1 MB     11.77%
             RAM:       45678 B     256 KB     17.41%
```

### Step 2: Detailed Memory Analysis

**Generate detailed size report:**
```sh
west build -t rom_report      # Flash usage by symbol
west build -t ram_report      # RAM usage by symbol
west build -t puncover         # Interactive HTML visualization
```

**Examine symbol sizes:**
```sh
arm-none-eabi-nm --size-sort -S build/zephyr/zephyr.elf | tail -20
```

**View memory map:**
```sh
cat build/zephyr/zephyr.map
```

### Step 3: Identify Memory Hotspots

**Flash optimization targets:**
- Large functions or libraries
- Debug strings and logging
- Unused features/drivers
- Crypto/networking stacks

**RAM optimization targets:**
- Thread stacks (often oversized)
- Heap size
- Static buffers
- Global variables

### Step 4: Apply Optimizations

See "Common Memory Optimizations" section below.

## Common Memory Optimizations

### Flash Reduction Strategies

**1. Disable Unused Features**

Review and disable in `prj.conf`:
```
# Disable assertions in production
CONFIG_ASSERT=n

# Minimal logging (or disable completely)
CONFIG_LOG=n
# Or use minimal level
CONFIG_LOG_DEFAULT_LEVEL=1
CONFIG_LOG_MODE_MINIMAL=y

# Disable shell if not needed
CONFIG_SHELL=n

# Disable printk for production
CONFIG_PRINTK=n
CONFIG_EARLY_CONSOLE=n
```

**2. Optimize Compilation**

```
# Size optimization (default is usually speed)
CONFIG_SIZE_OPTIMIZATIONS=y
CONFIG_COMPILER_OPT="-Os"

# Link-time optimization
CONFIG_LTO=y
```

**3. Remove Debug Information**

Build in Release mode:
```sh
west build -b <board> -- -DCMAKE_BUILD_TYPE=Release
```

**4. Reduce Networking Stack Size**

```
# For minimal networking
CONFIG_NET_BUF_DATA_SIZE=128
CONFIG_NET_PKT_RX_COUNT=4
CONFIG_NET_PKT_TX_COUNT=4

# Disable IPv6 if only using IPv4
CONFIG_NET_IPV6=n
```

**5. Minimize Bluetooth Stack**

```
# Reduce Bluetooth buffers
CONFIG_BT_BUF_ACL_TX_COUNT=3
CONFIG_BT_BUF_ACL_TX_SIZE=27
CONFIG_BT_CTLR_DATA_LENGTH_MAX=27

# Disable unused Bluetooth features
CONFIG_BT_OBSERVER=n
CONFIG_BT_BROADCASTER=n
```

### RAM Reduction Strategies

**1. Optimize Thread Stack Sizes**

Analyze actual stack usage:
```
# Enable thread analyzer
CONFIG_THREAD_ANALYZER=y
CONFIG_THREAD_NAME=y
```

Then at runtime, check stack usage and adjust:
```c
// In code, enable periodic analysis
CONFIG_THREAD_ANALYZER_AUTO=y
CONFIG_THREAD_ANALYZER_AUTO_INTERVAL=60
```

Reduce oversized stacks in `prj.conf`:
```
CONFIG_MAIN_STACK_SIZE=2048      # Default often 4096+
CONFIG_IDLE_STACK_SIZE=512       # Default often 1024+
CONFIG_SYSTEM_WORKQUEUE_STACK_SIZE=1024
CONFIG_BT_RX_STACK_SIZE=1024
```

**2. Reduce Heap Size**

```
# If not using malloc/new, disable heap
CONFIG_HEAP_MEM_POOL_SIZE=0

# Or minimize it
CONFIG_HEAP_MEM_POOL_SIZE=2048
```

**3. Optimize Buffer Sizes**

```
# Network buffers
CONFIG_NET_BUF_DATA_SIZE=128

# UART buffers
CONFIG_UART_ASYNC_TX_BUF_SIZE=256
CONFIG_UART_ASYNC_RX_BUF_SIZE=256
```

**4. Reduce Shell/Log Memory**

```
# Reduce logging buffer
CONFIG_LOG_BUFFER_SIZE=512

# Reduce shell backend buffer
CONFIG_SHELL_BACKEND_SERIAL_TX_RING_BUFFER_SIZE=64
CONFIG_SHELL_BACKEND_SERIAL_RX_RING_BUFFER_SIZE=64
```

## Memory Debugging

### Debug Stack Overflows

**Enable stack protection:**
```
CONFIG_STACK_SENTINEL=y        # Detects overflow
CONFIG_USERSPACE=y             # Memory protection (if HW supports)
CONFIG_THREAD_STACK_INFO=y     # Stack usage info
CONFIG_MPU_STACK_GUARD=y       # MPU protection (ARM Cortex-M)
```

**Monitor stack usage:**
```c
#include <zephyr/kernel.h>

void check_stack_usage(void) {
    size_t unused;
    k_thread_stack_space_get(k_current_get(), &unused);
    printk("Thread %s: %zu bytes unused\n", 
           k_thread_name_get(k_current_get()), unused);
}
```

**Use Thread Analyzer:**
```
CONFIG_THREAD_ANALYZER=y
CONFIG_THREAD_ANALYZER_USE_PRINTK=y
CONFIG_THREAD_ANALYZER_AUTO=y
```

View output via UART/RTT to see per-thread stack usage.

### Debug Heap Issues

**Enable heap debugging:**
```
CONFIG_HEAP_MEM_POOL_SIZE=8192
CONFIG_SYS_HEAP_VALIDATE=y     # Validate heap on alloc/free
```

**Track allocations:**
```c
#include <zephyr/sys/heap_listener.h>

// Monitor heap usage
size_t free_bytes, used_bytes;
sys_heap_runtime_stats_get(&_system_heap, &free_bytes, &used_bytes);
printk("Heap: %zu used, %zu free\n", used_bytes, free_bytes);
```

### Debug Memory Corruption

**Enable memory protection:**
```
CONFIG_USERSPACE=y
CONFIG_MPU_STACK_GUARD=y
CONFIG_HW_STACK_PROTECTION=y
CONFIG_THREAD_STACK_INFO=y
```

**Use address sanitizers (if available):**
```
# Some platforms support AddressSanitizer
CONFIG_ASAN=y
```

## Partition Manager (Multi-Image Builds)

For builds with bootloader (MCUboot) + app + network core:

**View partition layout:**
```sh
cat build/partitions.yml
```

**Adjust partition sizes:**

Create `pm_static.yml` or `pm_static_<board>.yml`:
```yaml
mcuboot:
  address: 0x0
  size: 0x10000
app:
  address: 0x10000
  size: 0x70000
mcuboot_pad:
  address: 0x80000
  size: 0x200
```

**Check partition usage:**
```sh
# After build, check fit
west build -t partition_manager_report
```

## Memory Profiling Tools

### Puncover - Interactive Memory Visualization

Generate HTML visualization:
```sh
west build -t puncover
# Opens browser with interactive memory breakdown
```

Features:
- Symbol-level Flash/RAM usage
- Clickable tree view by subsystem
- Size trends across builds

### Static Analysis

```sh
# Detailed ELF analysis
arm-none-eabi-size -A build/zephyr/zephyr.elf

# Section sizes
arm-none-eabi-objdump -h build/zephyr/zephyr.elf

# Symbol table
arm-none-eabi-nm -S --size-sort build/zephyr/zephyr.elf
```

### Runtime Analysis with GDB

```sh
west debug
# In GDB:
(gdb) info mem          # Memory regions
(gdb) x/100x 0x20000000 # Examine RAM
(gdb) print &_heap_start
(gdb) print &_heap_end
```

## Common Memory Issues

### Issue: Flash Overflow

**Symptoms:** Build fails with "region `FLASH' overflowed"

**Solutions:**
1. Enable size optimizations: `CONFIG_SIZE_OPTIMIZATIONS=y`
2. Disable debug features: `CONFIG_ASSERT=n`, `CONFIG_LOG=n`
3. Build in Release mode: `-DCMAKE_BUILD_TYPE=Release`
4. Disable unused subsystems (BT features, networking, etc.)
5. Consider device with more Flash

### Issue: RAM Overflow

**Symptoms:** Build fails with "region `RAM' overflowed"

**Solutions:**
1. Reduce thread stack sizes (analyze with Thread Analyzer first)
2. Reduce heap: `CONFIG_HEAP_MEM_POOL_SIZE`
3. Reduce buffers (network, UART, shell, logging)
4. Use static allocation instead of heap
5. Consider device with more RAM

### Issue: Stack Overflow at Runtime

**Symptoms:** Hard faults, crashes, corruption, unexpected behavior

**Solutions:**
1. Enable `CONFIG_STACK_SENTINEL=y` to detect
2. Use Thread Analyzer to measure actual usage
3. Increase problematic thread's stack size
4. Review recursive functions or large local arrays
5. Enable `CONFIG_MPU_STACK_GUARD=y` for protection

### Issue: Heap Exhaustion

**Symptoms:** malloc/k_malloc returns NULL, out-of-memory errors

**Solutions:**
1. Increase `CONFIG_HEAP_MEM_POOL_SIZE`
2. Check for memory leaks (missing free calls)
3. Use static allocation when possible
4. Enable heap validation: `CONFIG_SYS_HEAP_VALIDATE=y`
5. Profile heap usage with sys_heap_runtime_stats_get()

### Issue: High Memory Usage from Logging

**Symptoms:** Large Flash/RAM usage, logging subsystem in rom_report/ram_report

**Solutions:**
1. Reduce log level: `CONFIG_LOG_DEFAULT_LEVEL=1` (errors only)
2. Use deferred mode with smaller buffer
3. Disable logs in production: `CONFIG_LOG=n`
4. Use minimal logging: `CONFIG_LOG_MODE_MINIMAL=y`
5. Remove LOG_MODULE_REGISTER from unused modules

## Best Practices

### Development Phase
```
# Enable all debugging features
CONFIG_ASSERT=y
CONFIG_LOG=y
CONFIG_LOG_DEFAULT_LEVEL=4
CONFIG_THREAD_ANALYZER=y
CONFIG_STACK_SENTINEL=y
CONFIG_SYS_HEAP_VALIDATE=y
```

### Production Phase
```
# Minimize memory usage
CONFIG_SIZE_OPTIMIZATIONS=y
CONFIG_LTO=y
CONFIG_ASSERT=n
CONFIG_LOG=n              # Or LOG_DEFAULT_LEVEL=1
CONFIG_PRINTK=n
CONFIG_SHELL=n
# Build: west build -b <board> -- -DCMAKE_BUILD_TYPE=MinSizeRel
```

### Continuous Monitoring

1. Track memory usage across builds
2. Set CI/CD thresholds (e.g., alert if Flash > 80%)
3. Review rom_report/ram_report regularly
4. Profile on target hardware (Debug vs Release builds differ)

## Example Workflows

### Basic Memory Analysis
```sh
# 1. Build the project
west build -b nrf52840dk_nrf52840

# 2. Check memory summary (already shown in build output)

# 3. Generate detailed reports
west build -t rom_report      # View Flash usage
west build -t ram_report      # View RAM usage
west build -t puncover         # Interactive HTML report

# 4. Identify largest symbols
arm-none-eabi-nm --size-sort -S build/zephyr/zephyr.elf | tail -30
```

### Reduce Flash by 50%
```sh
# 1. Start with current usage
west build -b nrf52840dk_nrf52840
# Note Flash usage (e.g., 250 KB)

# 2. Apply optimizations to prj.conf
# - CONFIG_SIZE_OPTIMIZATIONS=y
# - CONFIG_LTO=y
# - CONFIG_LOG=n
# - CONFIG_ASSERT=n
# - CONFIG_SHELL=n

# 3. Rebuild in Release mode
west build -b nrf52840dk_nrf52840 -p -- -DCMAKE_BUILD_TYPE=MinSizeRel

# 4. Verify new usage (should be ~125 KB)
```

### Debug Stack Overflow
```sh
# 1. Enable stack debugging in prj.conf:
# CONFIG_STACK_SENTINEL=y
# CONFIG_THREAD_ANALYZER=y
# CONFIG_THREAD_ANALYZER_AUTO=y
# CONFIG_THREAD_NAME=y

# 2. Rebuild and flash
west build -b nrf52840dk_nrf52840
west flash

# 3. Monitor UART/RTT output for:
# - "Stack overflow detected" from sentinel
# - Thread analyzer periodic reports

# 4. Increase stack for problematic thread
# CONFIG_<THREAD>_STACK_SIZE=<larger_value>
```

### Optimize Thread Stacks
```sh
# 1. Enable thread analyzer
# CONFIG_THREAD_ANALYZER=y
# CONFIG_THREAD_ANALYZER_AUTO=y
# CONFIG_THREAD_ANALYZER_AUTO_INTERVAL=30
# CONFIG_THREAD_NAME=y

# 2. Build, flash, and run application through typical use cases
west build -b nrf52840dk_nrf52840
west flash

# 3. View RTT/UART output showing stack usage per thread
# Example output:
#   sysworkq  : unused 512 usage 512 / 1024 (50 %)
#   bt_rx     : unused 256 usage 768 / 1024 (75 %)

# 4. Adjust stack sizes in prj.conf based on actual usage + margin
# CONFIG_SYSTEM_WORKQUEUE_STACK_SIZE=1536  # Was 1024, needs more
# CONFIG_BT_RX_STACK_SIZE=896              # Was 1024, can reduce
```

## Notes

- Memory usage varies significantly between Debug and Release builds
- Always test on actual hardware (QEMU/emulation differs)
- Stack overflow is the most common memory issue in embedded systems
- When in doubt, use Thread Analyzer before adjusting stack sizes
- Partition Manager is used for multi-core and bootloader scenarios
- Some optimizations (LTO, size opts) increase build time
- Memory-mapped peripherals don't count against Flash/RAM limits
- Use `west build -t menuconfig` to explore memory-related Kconfig options
- Device Tree overlays can affect memory layout (reserved regions)

````