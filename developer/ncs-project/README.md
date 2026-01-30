# NCS Project Skill

**Complete Nordic nRF Connect SDK Project Management**

This skill provides an integrated workflow for:
- ✅ **Generate**: Create projects with proven templates
- ✅ **Review**: Validate quality with systematic checks
- ✅ **Improve**: Evolve templates through feedback

## 🚀 Quick Start

### Generate a New Wi-Fi Project

```bash
# 1. Create structure
mkdir wifi_app && cd wifi_app
mkdir -p src boards

# 2. Copy templates
cp ~/.claude/skills/developer/ncs-project/templates/{LICENSE,.gitignore,README_TEMPLATE.md} .

# 3. Copy Wi-Fi config (choose one)
cp ~/.claude/skills/developer/ncs-project/configs/wifi-sta.conf overlay-wifi.conf

# 4. Create core files (see guides/PROJECT_STRUCTURE.md)

# 5. Build
west build -p -b nrf7002dk/nrf5340/cpuapp -- -DEXTRA_CONF_FILE=overlay-wifi.conf
```

### Review a Project

```bash
# Quick automated check
~/.claude/skills/developer/ncs-project/review/check_project.sh /path/to/project

# Full review: Use review/CHECKLIST.md
```

## 📁 Structure

```
ncs-project/
├── SKILL.md                # This is the main skill reference (~2000 tokens)
│
├── templates/              # Copy to new projects
│   ├── LICENSE
│   ├── .gitignore
│   ├── README_TEMPLATE.md
│   └── PDR_TEMPLATE.md
│
├── configs/                # Wi-Fi configurations
│   ├── wifi-sta.conf
│   ├── wifi-softap.conf
│   ├── wifi-p2p.conf
│   └── wifi-raw.conf
│
├── guides/                 # Detailed documentation
│   ├── WIFI_GUIDE.md
│   ├── CONFIG_GUIDE.md
│   └── PROJECT_STRUCTURE.md
│
├── review/                 # QA tools
│   ├── check_project.sh
│   ├── QA_REPORT_TEMPLATE.md
│   ├── CHECKLIST.md
│   └── IMPROVEMENT_GUIDE.md
│
└── examples/               # Reference implementations
```

## 📖 Documentation

- **SKILL.md**: Quick reference guide (load this for overview)
- **guides/**: Comprehensive guides (reference when needed)
- **templates/**: Ready-to-use project templates
- **configs/**: Wi-Fi overlay configurations

## 🔄 Workflow

```
Generate → Develop → Review → QA Report → Fix → Improve Templates → Generate
```

## Token Efficiency

- **SKILL.md**: ~2,000 tokens (auto-loaded)
- **Guides**: 5,000+ tokens each (loaded on demand)
- **Templates**: Accessed as needed

**Total auto-load**: ~2,000 tokens vs previous ~35,000 tokens = **94% reduction!**

## 🆘 Getting Help

**For generation**: See `SKILL.md` and `guides/PROJECT_STRUCTURE.md`  
**For Wi-Fi**: See `guides/WIFI_GUIDE.md`  
**For review**: Use `review/CHECKLIST.md`  
**For configs**: Check `guides/CONFIG_GUIDE.md`

Start with `SKILL.md` for the complete quick reference!
