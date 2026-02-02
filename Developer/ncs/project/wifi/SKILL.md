````skill
---
name: ncs-wifi
description: Wi-Fi development for Nordic nRF Connect SDK - station, SoftAP, P2P, and raw packet modes
parent: ncs-project
---

# Wi-Fi Development Subskill

Complete Wi-Fi support for Nordic nRF70-series devices.

## 🎯 Quick Access

### Wi-Fi Modes (Choose ONE per project)

**Station Mode** (Connect to Access Point):
```bash
cp ~/.claude/skills/Developer/ncs/project/wifi/configs/wifi-sta.conf .
```

**SoftAP Mode** (Create Access Point):
```bash
cp ~/.claude/skills/Developer/ncs/project/wifi/configs/wifi-softap.conf .
```

**P2P Mode** (Wi-Fi Direct):
```bash
cp ~/.claude/skills/Developer/ncs/project/wifi/configs/wifi-p2p.conf .
```

**Raw Packet Mode** (Monitor/Promiscuous):
```bash
cp ~/.claude/skills/Developer/ncs/project/wifi/configs/wifi-raw.conf .
```

## 📖 Documentation

- **[WIFI_GUIDE.md](wifi/guides/WIFI_GUIDE.md)**: Comprehensive Wi-Fi development guide (~8,000 tokens)
  - All Wi-Fi modes explained
  - Memory requirements
  - Configuration details
  - Best practices
  - Troubleshooting

## ⚙️ Configuration Files

Located in `wifi/configs/`:
- `wifi-sta.conf` - Station mode (connect to AP)
- `wifi-softap.conf` - SoftAP mode (create AP)
- `wifi-p2p.conf` - P2P/Wi-Fi Direct mode
- `wifi-raw.conf` - Monitor/raw packet mode

## 🔧 Common Requirements

All Wi-Fi projects need:
```properties
CONFIG_WIFI=y
CONFIG_WIFI_NRF70=y
CONFIG_WIFI_READY_LIB=y
CONFIG_HEAP_MEM_POOL_SIZE=80000  # Minimum!
```

## 📊 Memory Requirements

| Mode | Flash | RAM | Heap |
|------|-------|-----|------|
| STA | 60KB | 50KB | 80KB min |
| SoftAP | 75KB | 60KB | 100KB min |
| P2P | 80KB | 65KB | 100KB min |
| Raw | 50KB | 45KB | 60KB min |

## 🚀 Usage

1. Copy desired Wi-Fi mode config
2. Build with overlay:
```bash
west build -p -b nrf7002dk/nrf5340/cpuapp -- \
  -DEXTRA_CONF_FILE="wifi-sta.conf"
```

For complete details, see [WIFI_GUIDE.md](wifi/guides/WIFI_GUIDE.md)

````