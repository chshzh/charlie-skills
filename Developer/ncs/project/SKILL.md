---
name: ncs-project
description: Complete Nordic nRF Connect SDK project workflow - generate projects from templates, review quality with QA reports, and continuously improve through feedback loops
---

# NCS Project Workflow

Unified skill for managing the complete lifecycle of Nordic nRF Connect SDK projects.

## 🎯 PRD-Driven Development

**PRD.md is the single source of truth for every NCS project.**  

Before any action (generate, review, improve), always:
1. 📋 **Start with PRD.md** - Copy PRD_TEMPLATE.md to your project
2. ✅ **Select Features** - Check boxes for required features (100+ options)
3. 📝 **Define Requirements** - Document user stories and acceptance criteria
4. 🏗️ **Guide Development** - Use PRD.md to drive implementation
5. 🔍 **Review Against PRD** - Validate project meets documented requirements
6. 🔄 **Update PRD** - Keep it current as project evolves

## 🎯 Three Core Actions

### 1. **Generate** - Create New Projects
Start new NCS projects with proven templates and best practices built-in.  
**Always create PRD.md first to guide development.**

### 2. **Review** - Validate Quality  
Systematically review projects against standards and generate detailed QA reports.  
**Compare implementation against PRD.md requirements.**

### 3. **Improve** - Evolve Templates
Update templates based on review findings for continuous improvement.  
**Update PRD_TEMPLATE.md based on lessons learned.**

---

## 🧩 Workspace Application Setup

All customer-facing apps must be *workspace applications* so anyone can clone the repo and fetch the exact Nordic SDK revision in one command.

1. **Add `west.yml` at the repo root** (pin `sdk-nrf` revision and import Zephyr):
        ```yaml
        manifest:
               self:
                      path: my_app
               remotes:
                      - name: ncs
                             url-base: https://github.com/nrfconnect
               projects:
                      - name: nrf
                             remote: ncs
                             repo-path: sdk-nrf
                             revision: v3.2.1
                             import: true
        ```
2. **Document the workflow in README**:
        ```bash
        west init -l my_app
        west update -o=--depth=1 -n
        west build -p -b nrf7002dk/nrf5340/cpuapp
        ```
3. **CI/CD requirement** – add `.github/workflows/build.yml` that:
        - parses the NCS version from `west.yml`
        - runs `west init/update` inside the GitHub-hosted container `ghcr.io/nrfconnect/sdk-nrf-toolchain:${NCS_VERSION}`
        - executes format/static checks (`checkpatch`, `clang-format --dry-run`)
        - builds every supported configuration (matrix strategy)
        - uploads merged HEX artifacts for releases

Use `nordic_wifi_opus_audio_demo/.github/workflows/build.yml` as the reference implementation.

---

## 📋 Quick Command Reference

### Generate New Project

**Basic NCS Application**:
```bash
# Create structure
mkdir my_project && cd my_project
mkdir -p src boards scripts tests doc

# Copy core templates
cp ~/.claude/skills/Developer/ncs/project/templates/LICENSE .
cp ~/.claude/skills/Developer/ncs/project/templates/.gitignore .
cp ~/.claude/skills/Developer/ncs/project/templates/README_TEMPLATE.md README.md
```

**Wi-Fi Project** (add overlay):
```bash
# Copy Wi-Fi configuration
cp ~/.claude/skills/Developer/ncs/project/configs/wifi-sta.conf overlay-wifi-sta.conf
# Or: wifi-softap.conf, wifi-p2p.conf, wifi-raw.conf
```

**Build**:
```bash
# Basic build (use your board)
west build -p -b nrf7002dk/nrf5340/cpuapp
# Or: nrf54l15dk/nrf54l15/cpuapp
# Or: nrf54lm20dk/nrf54lm20/cpuapp

# With Wi-Fi overlay
west build -p -b nrf7002dk/nrf5340/cpuapp -- -DEXTRA_CONF_FILE=overlay-wifi-sta.conf
```

### Review Project

**Quick automated check** (~1 min):
```bash
~/.claude/skills/ProductManager/ncs/review/check_project.sh /path/to/project
```

**Standard review** (2-3 hours):
1. Use `ProductManager/ncs/review/CHECKLIST.md` as guide
2. Fill out `ProductManager/ncs/review/QA_REPORT_TEMPLATE.md`
3. Generate recommendations

### Improve Templates

After reviews, update templates based on findings:
1. Document issues in `ProductManager/ncs/review/IMPROVEMENT_GUIDE.md`
2. Update relevant templates/configs
3. Test improvements
4. Deploy for next project

---

## 📦 What's Included

### Templates (Ready to Copy)
- `templates/LICENSE` - Nordic 5-Clause license
- `templates/.gitignore` - NCS-specific ignore patterns  
- `templates/README_TEMPLATE.md` - Comprehensive README structure
- `ProductManager/ncs/prd/PRD_TEMPLATE.md` - Product Requirements Document with feature selection

### Wi-Fi Mode Configurations
- `configs/wifi-sta.conf` - Station mode (connect to AP)
- `configs/wifi-softap.conf` - SoftAP mode (create AP)
- `configs/wifi-p2p.conf` - P2P/Wi-Fi Direct mode
- `configs/wifi-raw.conf` - Monitor/raw packet mode

### Feature Overlays (NEW! Modular feature selection)
- `ProductManager/ncs/features/overlays/overlay-wifi-shell.conf` - Interactive Wi-Fi shell
- `ProductManager/ncs/features/overlays/overlay-udp.conf` - UDP protocol
- `ProductManager/ncs/features/overlays/overlay-tcp.conf` - TCP protocol
- `ProductManager/ncs/features/overlays/overlay-mqtt.conf` - MQTT messaging
- `ProductManager/ncs/features/overlays/overlay-http-client.conf` - HTTP/HTTPS client
- `ProductManager/ncs/features/overlays/overlay-https-server.conf` - HTTPS server
- `ProductManager/ncs/features/overlays/overlay-coap.conf` - CoAP protocol
- `ProductManager/ncs/features/overlays/overlay-memfault.conf` - Memfault monitoring
- `ProductManager/ncs/features/overlays/overlay-ble-prov.conf` - BLE credential provisioning

### Architecture Pattern Overlays (NEW!)
- `ProductManager/ncs/features/overlays/overlay-smf-zbus.conf` - SMF+zbus modular architecture
- `ProductManager/ncs/features/overlays/overlay-multithreaded.conf` - Simple multi-threaded architecture

### Comprehensive Guides
- `guides/ARCHITECTURE_PATTERNS.md` - **NEW!** Multi-threaded vs SMF+zbus patterns
- `guides/WIFI_GUIDE.md` - Wi-Fi development patterns & best practices
- `guides/CONFIG_GUIDE.md` - Configuration management
- `guides/PROJECT_STRUCTURE.md` - File organization
- `ProductManager/ncs/features/FEATURE_SELECTION.md` - Complete feature selection guide

**STEP 0: Always start with PRD.md!**

```bash
# 1. Create project
mkdir my_iot_device && cd my_iot_device
mkdir -p src boards config docQuick reference checklist
- `ProductManager/ncs/review/IMPROVEMENT_GUIDE.md` - Template feedback process

### Examples
- `examples/` - Reference implementations

---

## 🚀 Common Workflows

### Starting a New Project with Feature Selection

```bash
# 1. Create project
mkdir my_iot_device && cd my_iot_device
mkdir -p src boards config

# 2. Copy templates
cp ~/.claude/skills/Developer/ncs/project/templates/{LICENSE,.gitignore} .
cp ~/.claude/skills/Developer/ncs/project/templates/README_TEMPLATE.md README.md
cp ~/.claude/skills/ProductManager/ncs/prd/PRD_TEMPLATE.md PRD.md

# 3. Select features (example: IoT sensor with MQTT + Memfault)
cp ~/.claude/skills/Developer/ncs/project/configs/wifi-sta.conf .
cp ~/.claude/skills/ProductManager/ncs/features/overlays/overlay-wifi-shell.conf .
cp ~/.claude/skills/ProductManager/ncs/features/overlays/overlay-mqtt.conf .
cp ~/.claude/skills/ProductManager/ncs/features/overlays/overlay-memfault.conf .

# 4. Edit PRD.md and check feature boxes for selected features
# Use PRD.md as your guide for all development, review, and improvement
# Update functional requirements based on features

# 5. Create core files (see guides/PROJECT_STRUCTURE.md)
# - CMakeLists.txt
# - Kconfig  
# - prj.conf
# - src/main.c

# 6. Build with all selected features
# Use your board: nrf7002dk/nrf5340/cpuapp, nrf54l15dk/nrf54l15/cpuapp, or nrf54lm20dk/nrf54lm20/cpuapp
west build -p -b nrf7002dk/nrf5340/cpuapp -- \
  -DEXTRA_CONF_FILE="wifi-sta.conf;overlay-wifi-shell.conf;overlay-mqtt.conf;overlay-memfault.conf"

# 7. Initialize git
git init && git add . && git commit -m "Initial commit"
```

### Starting a New Wi-Fi P2P Project

```bash
# 1. Create project
mkdir wifi_p2p_app && cd wifi_p2p_app
mkdir -p src boards

# 2. Copy templates
cp ~/.claude/skills/Developer/ncs/project/templates/{LICENSE,.gitignore} .
cp ~/.claude/skills/Developer/ncs/project/templates/README_TEMPLATE.md README.md

# 3. Copy P2P configuration + desired features
cp ~/.claude/skills/Developer/ncs/project/configs/wifi-p2p.conf .
cp ~/.claude/skills/ProductManager/ncs/features/overlays/overlay-wifi-shell.conf .
cp ~/.claude/skills/ProductManager/ncs/features/overlays/overlay-tcp.conf .

# 4. Create core files (see guides/PROJECT_STRUCTURE.md)
# - CMakeLists.txt
# - Kconfig  
# - prj.conf
# - src/main.c

# 5. Build
# Use your board: nrf7002dk/nrf5340/cpuapp, nrf54l15dk/nrf54l15/cpuapp, or nrf54lm20dk/nrf54lm20/cpuapp
west build -p -b nrf7002dk/nrf5340/cpuapp -- \
  -DEXTRA_CONF_FILE="wifi-p2p.conf;overlay-wifi-shell.conf;overlay-tcp.conf"

# 6. Initialize git
git init && git add . && git commit -m "Initial commit"
```

### Before Release Review
0. Verify PRD.md exists and is up-to-date
cat PRD.md  # Should reflect actual implementation

# 1. Quick check
~/.claude/skills/ProductManager/ncs/review/check_project.sh .

# 2. Fix critical issues immediately

# 3. Request formal review
# Reviewer uses ProductManager/ncs/review/CHECKLIST.md and PRD.md to validate requirements
# Generate QA report comparing implementation vs PRD

# 4. Address findings

# 5. Update PRD.md if requirements changed

# 6. Address findings

# 5. Re-review if needed
```

### Post-Review Improvement

```bash
# 1. Document common issues found
# Add to ProductManager/ncs/review/IMPROVEMENT_GUIDE.md

# 2. Update templates
# Fix gaps in templates/ or configs/

# 3. Update guides
# Enhance guides/ with new patterns

# 4. Next project benefits!
```
**PRD.md** (Product Requirements Document - PRIMARY GUIDE)
- ✅ 
---

## 📊 Critical Requirements by Project Type

### All NCS Projects (MUST HAVE)
- ✅ CMakeLists.txt (min version 3.20.0)
- ✅ Kconfig (source "Kconfig.zephyr")
- ✅ prj.conf (base configuration)
- ✅ src/main.c (application entry)
- ✅ LICENSE (Nordic 5-Clause)
- ✅ README.md (complete documentation)
- ✅ .gitignore (NCS patterns)

### Wi-Fi Projects (ADDITIONAL REQUIREMENTS)
- ✅ `CONFIG_WIFI=y` and `CONFIG_WIFI_NRF70=y`
- ✅ `CONFIG_WIFI_READY_LIB=y` (safe initialization)
- ✅ `CONFIG_HEAP_MEM_POOL_SIZE≥80000` (critical!)
- ✅ Network stack configured
- ✅ Proper Wi-Fi mode overlay (sta/softap/p2p/raw)
- ✅ Event handling implemented
- ⚠️ NO hardcoded credentials

### Wi-Fi P2P Specific
- ✅ `CONFIG_NRF70_P2P_MODE=y`
- ✅ `CONFIG_NRF_WIFI_LOW_POWER=n` (disable for P2P)
- ✅ Both DHCP server and client enabled
- ✅ Timing delays configured (4-way handshake, DHCP start)
- ✅ GO intent properly set (0-15)

---

## 🔍 Review Scoring Guide

**100-Point Scale**:
- **PRD.md Quality**: 15 points (completeness, accuracy, up-to-date)
- Project Structure: 10 points
- Core Files Quality: 15 points
- Configuration: 15 points
- Code Quality: 15 points
- Documentation: 10 points
- Wi-Fi/Features: 10 points
- Security: 10 points
- Build & Testing: 10 points

**Grades**:
- 90-100: ✅ Excellent - Ready for release
- 80-89: ✅ Good - Minor improvements needed
- 70-79: ⚠️ Satisfactory - Notable issues to fix
- 60-69: 🔧 Needs work - Significant problems
- <60: ❌ Fail - Major rework required

---

## 🔒 Security Checklist

**Before Every Commit**:
- [ ] No hardcoded credentials in code
- [ ] No passwords in configuration files
- [ ] Credential overlays in .gitignore
- [ ] Track only `overlay-*-credentials.conf.template` files – real credential overlays stay local
- [ ] No private keys committed
- [ ] No API tokens in code

**Wi-Fi Security**:
- [ ] WPA2-PSK minimum (prefer WPA3)
- [ ] TLS enabled for network communications
- [ ] Certificate validation implemented
- [ ] Input validation on network data

---

## 🎓 Best Practices

### For Developers
1. **Start with PRD.md** - Create and complete PRD.md before coding
2. **Check feature boxes** - Select features in PRD.md feature matrix
3. **Follow PRD requirements** - Implement according to PRD specifications
4. **Start with templates** - Don't reinvent the wheel
5. **Follow Wi-Fi memory requirements** - Heap ≥80KB critical
6. **Handle all network events** - Connect/disconnect/IP assigned
7. **Update PRD as you build** - Keep PRD.md current with changes
8. **Test early and often** - Build + hardware testing

## 🧪 Lessons from SoftAP Webserver QA

- **Document the actual SoftAP network** – Standardize on the `192.168.7.0/24` subnet with the gateway at `192.168.7.1`, and mirror those values across README Quick Start tables, PRD test cases, and troubleshooting guides so users never see stale `192.168.1.x` examples again.
- **Ship credential overlays, not secrets** – Check in `overlay-wifi-credentials.conf.template` (or equivalent) and `.gitignore` the developer-specific overlay. Every README needs a note that developers must copy the template locally and keep passwords out of logs, configs, and commits.
- **Describe per-board capabilities** – When a design supports multiple DKs, PRD.md and README.md must include button/LED tables that match each board’s silkscreen so QA can validate acceptance criteria without guessing.
- **Keep automation green** – Treat `ProductManager/ncs/review/check_project.sh` as a release gate. Fix the script as soon as it drifts (missing tools, path errors, etc.) before starting manual QA work.
- **Plan for Wi-Fi recovery** – SoftAP enable can fail after power events. Build a retry/back-off path into SMF/zbus states (and document it in requirements) so the system self-recovers instead of leaving QA to cycle power manually.

### For Reviewers
1. **Verify PRD.md exists** - Must be present and complete
2. **Compare against PRD** - Validate implementation matches requirements
3. **Run automated check first** - check_project.sh
4. **Be systematic** - Follow checklist completely
5. **Provide examples** - Show specific code issues
6. **Suggest solutions** - Not just problems
7. **Update templates** - Feed learnings back to PRD_TEMPLATE.md

---

## 📁 File Organization Reference

```
ncs-project/
├── SKILL.md                    # This file (quick reference)
│
├── templates/                  # Project templates (copy to new projects)
│   ├── LICENSE                 # Nordic 5-Clause license
│   ├── .gitignore             # NCS ignore patterns
│   ├── README_TEMPLATE.md     # Complete README structure
│   └── PRD_TEMPLATE.md        # Product Requirements Document with feature selection
│
├── configs/                    # Wi-Fi mode configurations
│   ├── wifi-sta.conf          # Wi-Fi Station mode
│   ├── wifi-softap.conf       # Wi-Fi SoftAP mode
│   ├── wifi-p2p.conf          # Wi-Fi P2P mode
│   └── wifi-raw.conf          # Monitor/raw packet mode
│
├── features/                   # Modular feature overlays (NEW!)
│   ├── overlay-wifi-shell.conf    # Wi-Fi shell commands
│   ├── overlay-udp.conf           # UDP protocol
│   ├── overlay-tcp.conf           # TCP protocol
│   ├── overlay-mqtt.conf          # MQTT messaging
│   ├── overlay-http-client.conf   # HTTP client
│   ├── overlay-https-server.conf  # HTTPS server
│   ├── overlay-coap.conf          # CoAP protocol
│   ├── overlay-ble-prov.conf      # BLE provisioning
│   ├── overlay-smf-zbus.conf      # SMF+zbus modular architecture (NEW!)
│   └── overlay-multithreaded.conf # Simple multi-threaded architecture (NEW!)ing
│   └── overlay-ble-prov.conf      # BLE provisioning
│
├── guides/                     # Detailed documentation
│   ├── WIFI_GUIDE.md          # Complete Wi-Fi development guide
│   ├── CONFIG_GUIDE.md        # Configuration management
│   ├── PROJECT_STRUCTURE.md   # File organization guide
│   ├── FEATURE_SELECTION.md   # Feature selection guide (NEW!)
│   └── ARCHITECTURE_PATTERNS.md # Multi-threaded vs SMF+zbus patterns (NEW!)
│
├── (See ProductManager/ncs/review/) # Quality assurance resources now centralized under Product Manager
│
└── examples/                   # Reference implementations
    ├── basic_app/             # Minimal NCS application
    ├── wifi_sta/              # Wi-Fi Station example
    └── wifi_p2p/              # Wi-Fi P2P example
```

---

## 🔄 Continuous Improvement Cycle

```
┌─────────────┐
│ PRD.md ★    │ ← Define requirements & select features
└──────┬──────┘
       │
       ▼
┌─────────────┐
│  GENERATE   │ ← Templates + Best Practices
└──────┬──────┘
       │
       ▼
┌─────────────┐
│   DEVELOP   │ (Build features according to PRD, test, document)
└──────┬──────┘
       │
       ▼
┌─────────────┐
│   REVIEW    │ ← Automated + Manual Checks vs PRD.md
└──────┬──────┘
       │
       ▼
┌─────────────┐
│  QA REPORT  │ (Findings + Recommendations)
└──────┬──────┘
       │
       ├──────────────┐
       ▼              ▼
┌─────────────┐  ┌─────────────┐
│  FIX ISSUES │  │   IMPROVE   │ ← Update PRD_TEMPLATE.md
└─────────────┘  └──────┬──────┘
                        │
                        ▼
              (Better next project!)
```

---

## 🆘 Common Issues & Quick Fixes

### Build Fails
- **Missing Zephyr**: Run `west update`
- **Kconfig error**: Check prj.conf syntax
- **Linker error**: Increase stack/heap sizes

### Wi-Fi Connection Fails
- **Wrong credentials**: Verify SSID/password via shell
- **Out of memory**: Increase `CONFIG_HEAP_MEM_POOL_SIZE≥80000`
- **DHCP timeout**: Check network, increase timeout values

### P2P Connection Fails
- **GO negotiation timeout**: Check both devices have P2P enabled
- **Role assignment wrong**: Adjust GO intent (0-15)
- **DHCP fails**: Add timing delays (4-way handshake, DHCP start)

### Review Fails
- **Missing PRD.md**: Copy from templates/PRD_TEMPLATE.md and complete it
- **PRD out of date**: Update to reflect actual implementation
- **Missing files**: Copy from templates/
- **Hardcoded credentials**: Move to .gitignore overlay
- **Poor documentation**: Use templates/README_TEMPLATE.md

---

## 📞 Getting Help

**Start here**:
- Copy `templates/PRD_TEMPLATE.md` to your project as PRD.md
- Complete the feature selection checklist
- Use PRD.md to guide all project decisions

**For project generation**:
- See `guides/PROJECT_STRUCTURE.md`
- Check `guides/WIFI_GUIDE.md` for Wi-Fi
- Review `examples/` for reference
- Follow `guides/FEATURE_SELECTION.md` for features

**For reviews**:
- Verify PRD.md exists and is accurate
- Use `ProductManager/ncs/review/CHECKLIST.md`
- Follow `ProductManager/ncs/review/QA_REPORT_TEMPLATE.md`
- Run `ProductManager/ncs/review/check_project.sh`
- Compare implementation against PRD requirements

**For template improvements**:
- Document in `ProductManager/ncs/review/IMPROVEMENT_GUIDE.md`
- Update affected templates (especially PRD_TEMPLATE.md)
- Test with new project

**External resources**:
- [NCS Documentation](https://developer.nordicsemi.com/nRF_Connect_SDK/doc/latest/nrf/index.html)
- [Nordic DevZone](https://devzone.nordicsemi.com/)
- [Zephyr Docs](https://docs.zephyrproject.org/latest/)

---

## 🎯 Summary

This skill provides everything needed for professional NCS project development:

✅ **PRD-Driven**: Product Requirements Document guides all actions  
✅ **Generate**: Proven templates and configurations  
✅ **Review**: Systematic validation against PRD and QA reporting  
✅ **Improve**: Continuous template enhancement  
✅ **Wi-Fi**: Complete guide for all modes  
✅ **Features**: Modular overlay system for 12+ features
✅ **Security**: Best practices and checks  
✅ **Quality**: Automated + manual validation  

### 🆕 Feature Selection System

**12 Core Features Available**:

**Wi-Fi Features:**
- Wi-Fi Shell - Interactive commands
- Wi-Fi STA - Station mode
- Wi-Fi SoftAP - Access point mode
- Wi-Fi P2P - Wi-Fi Direct

**Network Protocols:**
- UDP - Fast datagram protocol
- TCP - Reliable stream protocol
- MQTT - IoT messaging
- HTTP Client - REST API calls
- HTTPS Server - Web interface
- CoAP - Constrained protocol

**Advanced Features:**
- Memfault - Cloud monitoring & OTA
- BLE Provisioning - Credential setup
Architecture Patterns (NEW!):**
- Simple Multi-Threaded - Traditional approach (queues, semaphores)
- SMF + zbus Modular - Nordic's recommended pattern for complex systems

**
**Build with any combination**:
```bash
west build -p -b nrf7002dk/nrf5340/cpuapp -- \
  -DEXTRA_CONF_FILE="wifi-sta.conf;overlay-mqtt.conf;overlay-memfault.conf"
```

**Complete documentation** in `guides/FEATURE_SELECTION.md`:
- Detailed config requirements
- Memory requirements
- Code examples
- Dependencies
- Common combinations

**Start building better NCS projects today!** 🚀
