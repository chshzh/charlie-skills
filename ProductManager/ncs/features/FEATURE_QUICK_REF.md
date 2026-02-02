# NCS Feature Quick Reference

Quick reference for selecting features during project planning and PDR creation.

---

## 📋 Feature Checklist

Select features for your project:

### Wi-Fi Features

- [ ] **Wi-Fi Shell** - Interactive Wi-Fi commands for development/debugging
  - Overlay: `overlay-wifi-shell.conf`
  - Flash: +10KB | RAM: +4KB | Heap: Shared with Wi-Fi
  
- [ ] **Wi-Fi STA** - Connect to existing Wi-Fi access points
  - Config: `wifi-sta.conf`
  - Flash: +80KB | RAM: +40KB | Heap: 80KB minimum
  
- [ ] **Wi-Fi SoftAP** - Create Wi-Fi access point for clients
  - Config: `wifi-softap.conf`
  - Flash: +90KB | RAM: +50KB | Heap: 100KB
  
- [ ] **Wi-Fi P2P** - Wi-Fi Direct peer-to-peer connections
  - Config: `wifi-p2p.conf`
  - Flash: +95KB | RAM: +55KB | Heap: 100KB

### Network Protocols

- [ ] **UDP** - Fast, connectionless datagram protocol
  - Overlay: `overlay-udp.conf`
  - Flash: +5KB | RAM: +8KB | Heap: Minimal
  - Use for: Real-time data, sensor telemetry, streaming
  
- [ ] **TCP** - Reliable, connection-oriented protocol
  - Overlay: `overlay-tcp.conf`
  - Flash: +15KB | RAM: +16KB | Heap: +4KB per connection
  - Use for: File transfer, reliable data, HTTP/MQTT base
  
- [ ] **MQTT** - IoT publish/subscribe messaging
  - Overlay: `overlay-mqtt.conf`
  - Flash: +45KB | RAM: +30KB | Heap: 40KB
  - Use for: Cloud telemetry, device messaging
  - Requires: Wi-Fi STA, TCP, TLS
  
- [ ] **HTTP Client** - RESTful API and web requests
  - Overlay: `overlay-http-client.conf`
  - Flash: +40KB | RAM: +25KB | Heap: 40KB
  - Use for: REST APIs, webhooks, cloud integration
  - Requires: Wi-Fi STA, TCP, DNS, TLS
  
- [ ] **HTTPS Server** - Secure web server on device
  - Overlay: `overlay-https-server.conf`
  - Flash: +80KB | RAM: +60KB | Heap: 128KB
  - Use for: Web UI, configuration portal, REST APIs
  - Requires: Wi-Fi (STA or SoftAP), TCP, TLS certs
  
- [ ] **CoAP** - Lightweight constrained protocol
  - Overlay: `overlay-coap.conf`
  - Flash: +25KB | RAM: +15KB | Heap: 32KB
  - Use for: Lightweight IoT, resource-constrained devices
  - Requires: Wi-Fi, UDP, DTLS

### Advanced Features

- [ ] **Memfault** - Cloud monitoring, debugging, OTA
  - Overlay: `overlay-memfault.conf`
  - Flash: +120KB | RAM: +50KB | Heap: 96KB | Storage: 64KB+
  - Use for: Remote debugging, crash analysis, fleet management
  - Requires: Wi-Fi STA, HTTP Client, TLS, Flash storage
  - Setup: Sign up at memfault.com, add project key
  
- [ ] **BLE Provisioning** - Bluetooth credential provisioning
  - Overlay: `overlay-ble-prov.conf`
  - Flash: +60KB | RAM: +20KB | Heap: Shared (98KB+ total)
  - Use for: User-friendly setup, secure credential provisioning
  - Requires: Wi-Fi, Bluetooth LE, Settings/NVS

### Architecture Patterns

- [ ] **Simple Multi-Threaded Architecture**
  - Overlay: `overlay-multithreaded.conf`
  - Best for: Simple apps, 1-3 threads, quick prototypes
  - Communication: Queues, semaphores, mutexes
  - Complexity: Low
  
- [ ] **SMF + zbus Modular Architecture**
  - Overlay: `overlay-smf-zbus.conf`
  - Best for: Complex apps, 4+ modules, scalable systems
  - Communication: Message bus (zbus)
  - State Management: State Machine Framework (SMF)
  - Complexity: Medium
  - Reference: Nordic's Asset Tracker Template pattern

> **Details**: See `guides/ARCHITECTURE_PATTERNS.md` for comprehensive comparison and implementation

---

## 🔢 Common Combinations

### IoT Sensor with Cloud (MQTT)
```
☑ Wi-Fi STA
☑ Wi-Fi Shell (dev only)
☑ UDP
☑ MQTT
☑ Memfault

Memory: Flash ~260KB | RAM ~120KB | Heap ~100KB
```

### Smart Home Device (HTTP + BLE Setup)
```
☑ Wi-Fi STA
☑ BLE Provisioning
☑ HTTP Client
☑ UDP
☑ Memfault

Memory: Flash ~280KB | RAM ~130KB | Heap ~100KB
```

### Local Web Server (Configuration Portal)
```
☑ Wi-Fi SoftAP
☑ HTTPS Server
☑ Wi-Fi Shell
☑ TCP

Memory: Flash ~200KB | RAM ~110KB | Heap ~128KB
```

### P2P File Transfer
```
☑ Wi-Fi P2P
☑ TCP
☑ UDP
☑ Wi-Fi Shell

Memory: Flash ~120KB | RAM ~80KB | Heap ~100KB
```

---

## 🏗️ Build Command Template

**Supported WiFi Boards**:
- `nrf7002dk/nrf5340/cpuapp` - nRF7002 DK (built-in WiFi)
- `nrf54l15dk/nrf54l15/cpuapp` - nRF54L15 DK + nRF7002EB
- `nrf54lm20dk/nrf54lm20/cpuapp` - nRF54LM20 DK + nRF7002EB

Based on selections, use this build pattern:

```bash
west build -p -b <board> -- \
  -DEXTRA_CONF_FILE="<wifi-mode>.conf;<feature1>.conf;<feature2>.conf"
```

**Example** (Wi-Fi STA + MQTT + Memfault on nRF7002DK):
```bash
west build -p -b nrf7002dk/nrf5340/cpuapp -- \
  -DEXTRA_CONF_FILE="wifi-sta.conf;overlay-mqtt.conf;overlay-memfault.conf"
```

**Example** (Same features on nRF54L15DK):
```bash
west build -p -b nrf54l15dk/nrf54l15/cpuapp -- \
  -DEXTRA_CONF_FILE="wifi-sta.conf;overlay-mqtt.conf;overlay-memfault.conf"
```

---

## 📊 Memory Budget Guide

### Minimum by Wi-Fi Mode
- **STA**: Flash 80KB, RAM 40KB, Heap 80KB
- **SoftAP**: Flash 90KB, RAM 50KB, Heap 100KB
- **P2P**: Flash 95KB, RAM 55KB, Heap 100KB

### Protocol Additions
- **UDP**: +5KB Flash, +8KB RAM
- **TCP**: +15KB Flash, +16KB RAM
- **MQTT**: +45KB Flash, +30KB RAM, +40KB Heap
- **HTTP Client**: +40KB Flash, +25KB RAM, +40KB Heap
- **HTTPS Server**: +80KB Flash, +60KB RAM, +128KB Heap
- **CoAP**: +25KB Flash, +15KB RAM, +32KB Heap

### Advanced Feature Additions
- **Memfault**: +120KB Flash, +50KB RAM, +96KB Heap
- **BLE Provisioning**: +60KB Flash, +20KB RAM (shared heap)
- **Wi-Fi Shell**: +10KB Flash, +4KB RAM
├── features/                   # Feature overlays (choose ANY)
│   ├── overlay-wifi-shell.conf
│   ├── overlay-udp.conf
│   ├── overlay-tcp.conf
│   ├── overlay-mqtt.conf
│   ├── overlay-http-client.conf
│   ├── overlay-https-server.conf
│   ├── overlay-coap.conf
│   ├── overlay-memfault.conf
│   ├── overlay-ble-prov.conf
│   ├── overlay-smf-zbus.conf        # SMF+zbus architecture
│   └── overlay-multithreaded.conf   # Simple multi-threaded
│
└── templates/
    └── modules/                # Module templates for SMF+zbus
        ├── module_template_smf.c
        ├── module_template_smf.h
        ├── messages.h
        └── Kconfig.module_template
```
~/.claude/skills/ProductManager/ncs/features/
├── configs/                    # Wi-Fi modes (choose ONE)
│   ├── wifi-sta.conf
│   ├── wifi-softap.conf
│   ├── wifi-p2p.conf
│   └── wifi-raw.conf
│
└── features/                   # Feature overlays (choose ANY)
    ├── overlay-wifi-shell.conf
    ├── overlay-udp.conf
    ├── overlay-tcp.conf
    ├── overlay-mqtt.conf
    ├── overlay-http-client.conf
    ├── overlay-https-server.conf
    ├── overlay-coap.conf
    ├── overlay-memfault.conf
    └── overlay-ble-prov.conf
```

---

## 🚨 Important Notes

### Dependencies
- **MQTT** requires: TCP + TLS
- **HTTP Client** requires: TCP + DNS + TLS
- **HTTPS Server** requires: TCP + TLS + certificates
- **CoAP** requires: UDP + DTLS
- **Memfault** requires: HTTP Client (all its dependencies)

### Security
- **Never hardcode credentials** - use BLE Provisioning or Settings
- **Use TLS** for all internet-facing protocols
- **Generate certificates** for HTTPS Server before deployment
- **Configure Memfault project key** via overlay

### Power Management
- **Disable** `CONFIG_NRF_WIFI_LOW_POWER=n` for:
  - Development
  - SoftAP mode
  - P2P mode
- **Enable** `CONFIG_NRF_WIFI_LOW_POWER=y` for:
  - Production STA mode
  - Battery-powered devices

### Heap Sizing
- Start with recommended heap size
- Monitor runtime with `kernel uptime` and `kernel stacks`
- Increase if seeing allocation failures
- Decrease for optimization after testing

---

## 📝 PDR Integration

When creating a PDR (Project Design Review):

1. **Open** `templates/PDR_TEMPLATE.md`
2. **Check boxes** in Section 1.1 Feature Selection
3. **Copy overlay files** to project directory
4. **Update build command** in PDR with selected overlays
5. **Document functional requirements** based on selected features
6. **Plan testing** for each selected feature

---

## 📚 Full Documentation

For complete details on each feature:
- Configuration requirements
- Full code examples
- API usage
- Best practices
- Troubleshooting

See: `guides/FEATURE_SELECTION.md`

---

**Quick Reference Card | NCS Project Skill**
