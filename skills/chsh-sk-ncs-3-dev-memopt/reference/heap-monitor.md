# Heap Monitoring — Reference

## Use the zego memonitor brick

Heap/thread watermark sampling is now a standalone **zego brick**, not a
per-project module — reuse it instead of writing a new one:

```
/opt/nordic/ncs/v3.4.0/zego/bricks/memonitor/
├── src/memonitor.c, memonitor.h
├── Kconfig
└── docs/memonitor-spec.md   ← full API, Kconfig reference, log format, wiring
```

Wire it in (see `memonitor-spec.md`'s "Integration Example" for the exact
`CMakeLists.txt`/`prj.conf` snippets):

```conf
CONFIG_ZEGO_MEMONITOR=y
CONFIG_ZEGO_MEMONITOR_LOG_PERIODIC=y   # UART log every sample; leave 'n' on flash-tight boards
```

Consumers read `memonitor_get_heaps()` / `memonitor_get_threads()` for a
thread-safe point-in-time snapshot from any context (HTTP handlers included).
The primary validated consumer is `nordic-wifi-webdash`, serving snapshots as
`/api/heaps` and `/api/threads`.

**Note the differences from an older, unrelated `heap_monitor` module this file
used to describe** (`CONFIG_HEAPS_MONITOR`, from a now-superseded reference
project) — don't mix their Kconfig symbols or log format:
- No `CONFIG_HEAPS_MONITOR_WARN_PCT` equivalent — memonitor has no
  threshold-triggered warning log. Read the `hwm=` percentage yourself each
  interval instead of waiting for a `LOG_WRN`.
- Log format is `heap <name> hwm=<used>/<total> (<pct>%)`, one line per heap
  (and one per thread) — not the old `used=… peak=…` phrasing.

## Memfault integration (not built into the brick)

Unlike the older module, `memonitor` does **not** call into Memfault on its own —
it only samples and publishes `MEMONITOR_CHAN`. Wire it yourself in a
`MEMONITOR_CHAN` subscriber inside your app's Memfault wrapper module: call
`memonitor_get_heaps()` on each notification and forward the values via
`MEMFAULT_METRIC_SET_UNSIGNED()`. Register the metric keys in
`src/modules/app_memfault/config/memfault_metrics_heartbeat_config.def`:

```c
#if CONFIG_ZEGO_MEMONITOR
MEMFAULT_METRICS_KEY_DEFINE(ncs_system_heap_total, kMemfaultMetricType_Unsigned)
MEMFAULT_METRICS_KEY_DEFINE(ncs_system_heap_used,  kMemfaultMetricType_Unsigned)
MEMFAULT_METRICS_KEY_DEFINE(ncs_system_heap_peak,  kMemfaultMetricType_Unsigned)
#if CONFIG_MBEDTLS_ENABLE_HEAP
MEMFAULT_METRICS_KEY_DEFINE(ncs_mbedtls_heap_total, kMemfaultMetricType_Unsigned)
MEMFAULT_METRICS_KEY_DEFINE(ncs_mbedtls_heap_used,  kMemfaultMetricType_Unsigned)
MEMFAULT_METRICS_KEY_DEFINE(ncs_mbedtls_heap_peak,  kMemfaultMetricType_Unsigned)
#endif
#endif
```

## Heap Architecture

NCS/Zephyr uses several distinct heaps:

### 1. System heap (`_system_heap`) — monitored by memonitor

- General `malloc`/`calloc`/`free`, POSIX thread allocations
- WPA supplicant (when `CONFIG_NRF_WIFI_GLOBAL_HEAP=y`)
- Sized by `CONFIG_HEAP_MEM_POOL_SIZE` + `HEAP_MEM_POOL_ADD_SIZE_*` (auto-calculated by Kconfig)
- **Wi-Fi apps: ≥64 KB base required; 80 KB recommended** (total ~125 KB with add-ons)

```sh
# Check what subsystems add to heap:
grep "HEAP_MEM_POOL_ADD_SIZE" build/zephyr/.config
# Common output:
# CONFIG_HEAP_MEM_POOL_ADD_SIZE_HOSTAP=41808       # WPA supplicant
# CONFIG_HEAP_MEM_POOL_ADD_SIZE_POSIX_THREADS=256
# CONFIG_HEAP_MEM_POOL_ADD_SIZE_ZBUS=3072
```

### 2. Net buffer slabs — NOT monitored, NOT in system heap

- Dedicated fixed-size slab pools for network packets
- Configured via `CONFIG_NET_BUF_RX_COUNT`, `CONFIG_NET_BUF_TX_COUNT`
- Separate for ISR safety, determinism, and fragmentation avoidance
- Do not attempt to move them to system heap

### 3. Wi-Fi driver heap (when `CONFIG_NRF_WIFI_GLOBAL_HEAP=n`)

- Dedicated region for firmware and DMA descriptors
- Provides isolation and predictable performance
- Use `CONFIG_NRF_WIFI_GLOBAL_HEAP=y` only if severely memory-constrained

### 4. mbedTLS heap (`CONFIG_MBEDTLS_ENABLE_HEAP=y`) — monitored by memonitor

- Fixed-size buffer carved out of RAM at boot, separate from `_system_heap`
- Sized exclusively by `CONFIG_MBEDTLS_HEAP_SIZE` (no auto add-ons)
- Peaks during TLS handshakes; can spike to ~70 KB on Wi-Fi/HTTPS workloads
- `CONFIG_MBEDTLS_MEMORY_DEBUG=y` is auto-selected by `CONFIG_ZEGO_MEMONITOR`
- Recommended margin: 1.5× of measured peak

```
CONFIG_MBEDTLS_ENABLE_HEAP=y
CONFIG_MBEDTLS_HEAP_SIZE=110592    # 150% of ~72 KB measured peak
```
