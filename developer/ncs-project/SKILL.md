---
name: ncs-project
description: Complete Nordic nRF Connect SDK project workflow - generate projects from templates, review quality with QA reports, and continuously improve through feedback loops
---

# NCS Project Workflow

Unified skill for managing the complete lifecycle of Nordic nRF Connect SDK projects.

## 🎯 Three Core Actions

### 1. **Generate** - Create New Projects
Start new NCS projects with proven templates and best practices built-in.

### 2. **Review** - Validate Quality  
Systematically review projects against standards and generate detailed QA reports.

### 3. **Improve** - Evolve Templates
Update templates based on review findings for continuous improvement.

---

## 📋 Quick Command Reference

### Generate New Project

**Basic NCS Application**:
```bash
# Create structure
mkdir my_project && cd my_project
mkdir -p src boards scripts tests doc

# Copy core templates
cp ~/.claude/skills/developer/ncs-project/templates/LICENSE .
cp ~/.claude/skills/developer/ncs-project/templates/.gitignore .
cp ~/.claude/skills/developer/ncs-project/templates/README_TEMPLATE.md README.md
```

**Wi-Fi Project** (add overlay):
```bash
# Copy Wi-Fi configuration
cp ~/.claude/skills/developer/ncs-project/configs/wifi-sta.conf overlay-wifi-sta.conf
# Or: wifi-softap.conf, wifi-p2p.conf, wifi-raw.conf
```

**Build**:
```bash
# Basic build
west build -p -b nrf7002dk/nrf5340/cpuapp

# With Wi-Fi overlay
west build -p -b nrf7002dk/nrf5340/cpuapp -- -DEXTRA_CONF_FILE=overlay-wifi-sta.conf
```

### Review Project

**Quick automated check** (~1 min):
```bash
~/.claude/skills/developer/ncs-project/review/check_project.sh /path/to/project
```

**Standard review** (2-3 hours):
1. Use `review/CHECKLIST.md` as guide
2. Fill out `review/QA_REPORT_TEMPLATE.md`
3. Generate recommendations

### Improve Templates

After reviews, update templates based on findings:
1. Document issues in `review/IMPROVEMENT_GUIDE.md`
2. Update relevant templates/configs
3. Test improvements
4. Deploy for next project

---

## 📦 What's Included

### Templates (Ready to Copy)
- `templates/LICENSE` - Nordic 5-Clause license
- `templates/.gitignore` - NCS-specific ignore patterns  
- `templates/README_TEMPLATE.md` - Comprehensive README structure
- `templates/PDR_TEMPLATE.md` - Project documentation repository

### Wi-Fi Configurations
- `configs/wifi-sta.conf` - Station mode (connect to AP)
- `configs/wifi-softap.conf` - SoftAP mode (create AP)
- `configs/wifi-p2p.conf` - P2P/Wi-Fi Direct mode
- `configs/wifi-raw.conf` - Monitor/raw packet mode

### Comprehensive Guides
- `guides/WIFI_GUIDE.md` - Wi-Fi development patterns & best practices
- `guides/CONFIG_GUIDE.md` - Configuration management
- `guides/PROJECT_STRUCTURE.md` - File organization

### Review Tools
- `review/check_project.sh` - Automated validation script
- `review/QA_REPORT_TEMPLATE.md` - Detailed QA report  
- `review/CHECKLIST.md` - Quick reference checklist
- `review/IMPROVEMENT_GUIDE.md` - Template feedback process

### Examples
- `examples/` - Reference implementations

---

## 🚀 Common Workflows

### Starting a New Wi-Fi P2P Project

```bash
# 1. Create project
mkdir wifi_p2p_app && cd wifi_p2p_app
mkdir -p src boards

# 2. Copy templates
cp ~/.claude/skills/developer/ncs-project/templates/{LICENSE,.gitignore} .
cp ~/.claude/skills/developer/ncs-project/templates/README_TEMPLATE.md README.md

# 3. Copy P2P configuration
cp ~/.claude/skills/developer/ncs-project/configs/wifi-p2p.conf overlay-p2p.conf

# 4. Create core files (see guides/PROJECT_STRUCTURE.md)
# - CMakeLists.txt
# - Kconfig  
# - prj.conf
# - src/main.c

# 5. Build
west build -p -b nrf7002dk/nrf5340/cpuapp -- -DEXTRA_CONF_FILE=overlay-p2p.conf

# 6. Initialize git
git init && git add . && git commit -m "Initial commit"
```

### Before Release Review

```bash
# 1. Quick check
~/.claude/skills/developer/ncs-project/review/check_project.sh .

# 2. Fix critical issues immediately

# 3. Request formal review
# Reviewer uses review/CHECKLIST.md and generates QA report

# 4. Address findings

# 5. Re-review if needed
```

### Post-Review Improvement

```bash
# 1. Document common issues found
# Add to review/IMPROVEMENT_GUIDE.md

# 2. Update templates
# Fix gaps in templates/ or configs/

# 3. Update guides
# Enhance guides/ with new patterns

# 4. Next project benefits!
```

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
- Project Structure: 15 points
- Core Files Quality: 20 points
- Configuration: 15 points
- Code Quality: 20 points
- Documentation: 15 points
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
1. **Start with templates** - Don't reinvent the wheel
2. **Follow Wi-Fi memory requirements** - Heap ≥80KB critical
3. **Handle all network events** - Connect/disconnect/IP assigned
4. **Document as you build** - Update README alongside code
5. **Test early and often** - Build + hardware testing

### For Reviewers
1. **Run automated check first** - check_project.sh
2. **Be systematic** - Follow checklist completely
3. **Provide examples** - Show specific code issues
4. **Suggest solutions** - Not just problems
5. **Update templates** - Feed learnings back

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
│   └── PDR_TEMPLATE.md        # Documentation template
│
├── configs/                    # Configuration overlays
│   ├── wifi-sta.conf          # Wi-Fi Station mode
│   ├── wifi-softap.conf       # Wi-Fi SoftAP mode
│   ├── wifi-p2p.conf          # Wi-Fi P2P mode
│   └── wifi-raw.conf          # Monitor/raw packet mode
│
├── guides/                     # Detailed documentation (5000+ lines each)
│   ├── WIFI_GUIDE.md          # Complete Wi-Fi development guide
│   ├── CONFIG_GUIDE.md        # Configuration management
│   └── PROJECT_STRUCTURE.md   # File organization guide
│
├── review/                     # Quality assurance resources
│   ├── check_project.sh       # Automated checks (executable)
│   ├── QA_REPORT_TEMPLATE.md  # Detailed QA report format
│   ├── CHECKLIST.md           # Review checklist
│   └── IMPROVEMENT_GUIDE.md   # Template improvement process
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
│  GENERATE   │ ← Templates + Best Practices
└──────┬──────┘
       │
       ▼
┌─────────────┐
│   DEVELOP   │ (Build features, test, document)
└──────┬──────┘
       │
       ▼
┌─────────────┐
│   REVIEW    │ ← Automated + Manual Checks
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
│  FIX ISSUES │  │   IMPROVE   │ ← Update Templates
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
- **Missing files**: Copy from templates/
- **Hardcoded credentials**: Move to .gitignore overlay
- **Poor documentation**: Use templates/README_TEMPLATE.md

---

## 📞 Getting Help

**For project generation**:
- See `guides/PROJECT_STRUCTURE.md`
- Check `guides/WIFI_GUIDE.md` for Wi-Fi
- Review `examples/` for reference

**For reviews**:
- Use `review/CHECKLIST.md`
- Follow `review/QA_REPORT_TEMPLATE.md`
- Run `review/check_project.sh`

**For template improvements**:
- Document in `review/IMPROVEMENT_GUIDE.md`
- Update affected templates
- Test with new project

**External resources**:
- [NCS Documentation](https://developer.nordicsemi.com/nRF_Connect_SDK/doc/latest/nrf/index.html)
- [Nordic DevZone](https://devzone.nordicsemi.com/)
- [Zephyr Docs](https://docs.zephyrproject.org/latest/)

---

## 🎯 Summary

This skill provides everything needed for professional NCS project development:

✅ **Generate**: Proven templates and configurations  
✅ **Review**: Systematic validation and QA reporting  
✅ **Improve**: Continuous template enhancement  
✅ **Wi-Fi**: Complete guide for all modes  
✅ **Security**: Best practices and checks  
✅ **Quality**: Automated + manual validation  

**Start building better NCS projects today!** 🚀
