````skill
---
name: ncs-features
description: Modular feature selection for Nordic NCS projects - Wi-Fi Shell, Memfault, BLE Provisioning, and more
parent: ncs-project
---

# Features Subskill

Modular feature overlays for Nordic NCS projects - choose any combination of 12 features.

## 🎯 Feature Categories

### Wi-Fi Features
- **Wi-Fi Shell**: Interactive commands for development/debugging
- Wi-Fi STA, SoftAP, P2P: (See [Developer Wi-Fi skill](../../../Developer/ncs/project/wifi/SKILL.md))

### Network Protocols
- MQTT, HTTP Client, HTTPS Server, CoAP, UDP, TCP  
  (See [Developer protocols skill](../../../Developer/ncs/project/protocols/SKILL.md))

### Advanced Features

**Memfault** - Cloud monitoring, debugging, and OTA:
```bash
cp ~/.claude/skills/ProductManager/ncs/features/overlays/overlay-memfault.conf .
```
- Flash: +120KB, RAM: +50KB, Heap: +96KB
- Requires: Wi-Fi STA, HTTP Client, TLS, Flash storage
- Setup: Sign up at memfault.com

**BLE Provisioning** - Bluetooth credential provisioning:
```bash
cp ~/.claude/skills/ProductManager/ncs/features/overlays/overlay-ble-prov.conf .
```
- Flash: +60KB, RAM: +20KB, Heap: Shared
- Requires: Wi-Fi, Bluetooth LE, Settings/NVS
- Use for: User-friendly device setup

**Wi-Fi Shell** - Interactive Wi-Fi commands:
```bash
cp ~/.claude/skills/ProductManager/ncs/features/overlays/overlay-wifi-shell.conf .
```
- Flash: +15KB, RAM: +8KB, Heap: +8KB
- Essential for development and testing
- Commands: wifi scan, connect, disconnect, stats

## 📖 Complete Documentation

**[FEATURE_SELECTION.md](features/FEATURE_SELECTION.md)** (~15,000 tokens)
- Detailed docs for all 12 features
- Complete Kconfig requirements
- Full code examples
- Memory requirements
- Dependencies
- Best practices

**[FEATURE_QUICK_REF.md](features/FEATURE_QUICK_REF.md)** (~3,000 tokens)
- Quick lookup guide
- Common combinations
- Memory budgets
- Build commands

## 🗂️ Feature Overlays

All overlays in `features/overlays/`:
- `overlay-wifi-shell.conf`
- `overlay-udp.conf`
- `overlay-tcp.conf`
- `overlay-mqtt.conf`
- `overlay-http-client.conf`
- `overlay-https-server.conf`
- `overlay-coap.conf`
- `overlay-memfault.conf`
- `overlay-ble-prov.conf`
- `overlay-multithreaded.conf` (architecture)
- `overlay-smf-zbus.conf` (architecture)

## 🚀 Usage

```bash
# Combine any features
west build -p -b nrf7002dk/nrf5340/cpuapp -- \
  -DEXTRA_CONF_FILE="wifi-sta.conf;overlay-wifi-shell.conf;overlay-mqtt.conf;overlay-memfault.conf"
```

## 📊 Common Combinations

**IoT Sensor with Cloud**:
- Wi-Fi STA + MQTT + Memfault
- Memory: Flash ~260KB, RAM ~120KB, Heap ~100KB

**Smart Home Device**:
- Wi-Fi STA + HTTP Client + BLE Prov + Memfault
- Memory: Flash ~280KB, RAM ~130KB, Heap ~100KB

**Configuration Portal**:
- Wi-Fi SoftAP + HTTPS Server + TCP
- Memory: Flash ~200KB, RAM ~110KB, Heap ~128KB

For complete details, see [FEATURE_SELECTION.md](features/FEATURE_SELECTION.md)

````