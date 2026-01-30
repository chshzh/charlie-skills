# Claude Skills Repository

Personal collection of specialized Claude skills for development workflows.

## 📦 Skills Overview

### Developer Skills

#### **ncs-project** - Nordic nRF Connect SDK Project Management
Complete lifecycle management for NCS projects with integrated generate→review→improve workflow.

**Location**: `developer/ncs-project/`

**Features**:
- ✅ Project generation with proven templates
- ✅ Wi-Fi configurations (Station, SoftAP, P2P, Raw/Monitor modes)
- ✅ Systematic quality review and validation
- ✅ Continuous improvement through feedback
- ✅ Token-optimized (~2,000 tokens vs 35,000)

**Quick Start**:
```bash
# Generate new project
cp developer/ncs-project/templates/* my_project/
cp developer/ncs-project/configs/wifi-sta.conf my_project/overlay-wifi.conf

# Review project
developer/ncs-project/review/check_project.sh /path/to/project
```

**Documentation**:
- [SKILL.md](developer/ncs-project/SKILL.md) - Main skill reference
- [PROJECT_STRUCTURE.md](developer/ncs-project/guides/PROJECT_STRUCTURE.md) - Project structure guide
- [WIFI_GUIDE.md](developer/ncs-project/guides/WIFI_GUIDE.md) - Wi-Fi development guide

#### **ncs-project-generate** ⚠️ DEPRECATED
Original project generation templates. Consolidated into `ncs-project`.

#### **ncs-project-review** ⚠️ DEPRECATED
Original project review framework. Consolidated into `ncs-project`.

## 🚀 Usage

Skills are designed to be referenced by Claude during conversations. Each skill provides:
- Templates and configurations
- Documentation and guides
- Automation scripts
- Best practices

## 📁 Structure

```
skills/
├── developer/
│   ├── ncs-project/              # Active unified NCS skill
│   ├── ncs-project-generate/     # Deprecated (archived)
│   └── ncs-project-review/       # Deprecated (archived)
├── README.md                      # This file
└── .gitignore
```

## 🔄 Workflow Integration

Skills support iterative development workflows:

```
Generate → Develop → Review → QA Report → Fix → Improve → Generate
```

## 📊 Token Efficiency

Skills are optimized for token consumption:
- Core SKILL.md files: ~2,000 tokens (auto-loaded)
- Detailed guides: 5,000+ tokens (loaded on-demand)
- Templates and configs: Accessed as needed

## 📝 License

Skills and templates include appropriate license headers for target projects.

Individual skills may reference or include:
- Nordic 5-Clause License (LicenseRef-Nordic-5-Clause)
- Apache License 2.0
- MIT License

## 👤 Author

**Charlie Shao** ([@chshzh](https://github.com/chshzh))  
Nordic Semiconductor - Olso, Norway

## 📅 Last Updated

January 30, 2026
