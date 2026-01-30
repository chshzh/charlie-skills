---
name: ncs-env-setup
description: Set up Nordic nRF Connect SDK (NCS) environment and run west commands. Use when building, flashing, or running any west command for nRF projects, or when working with Zephyr-based Nordic applications.
---

# Nordic nRF Connect SDK (NCS) Environment

## Key Concepts

**SDK version** and **Toolchain version** are separate:
- **SDK (nRF Connect SDK)**: The source code (e.g., v3.2.1, main branch, custom branch)
- **Toolchain**: Build tools like west, cmake, compilers (tied to bundle IDs)

These can differ! Example: NCS main branch + v3.2.1 toolchain.

## Critical Rule

**ALWAYS check if the NCS environment is set up before running any `west` command.**

## Workflow for West Commands

When the user requests any west command (`west build`, `west flash`, `west update`, etc.):

### Step 1: Check Terminal Status

Check if there's an active terminal with NCS configured. Look for version in prompt (e.g., `(v3.2.1)`).

### Step 2: Determine SDK and Toolchain Versions

**If NO active terminal OR terminal doesn't show NCS version:**

Ask the user TWO things:
1. **Which toolchain version?** (e.g., "v3.2.1" → determines bundle ID for build tools)
2. **Which SDK path/version?** (e.g., `/opt/nordic/ncs/v3.2.1` or `/opt/nordic/ncs/main`)

Example question: "Which toolchain version would you like to use? (e.g., v3.2.1) And which SDK path? (e.g., /opt/nordic/ncs/v3.2.1 or a custom path)"

**Common scenarios:**
- Same version: Toolchain v3.2.1 + SDK at `/opt/nordic/ncs/v3.2.1`
- Different: Toolchain v3.2.1 + SDK at `/opt/nordic/ncs/main`

**If terminal already shows NCS version in prompt:**
- Proceed directly with the west command

### Step 3: Set Up Environment (if needed)

1. Look up the toolchain bundle ID using the **toolchain version**:
   ```sh
   jq -r '.toolchains[] | select(.ncs_versions[]=="<TOOLCHAIN_VERSION>").identifier.bundle_id' /opt/nordic/ncs/toolchains/toolchains.json
   ```

2. Export PATH, GIT_EXEC_PATH, and source Zephyr environment from the **SDK path**:
   ```sh
   BUNDLE_ID="$(jq -r '.toolchains[] | select(.ncs_versions[]=="<TOOLCHAIN_VERSION>").identifier.bundle_id' /opt/nordic/ncs/toolchains/toolchains.json)"
   export PATH="/opt/nordic/ncs/toolchains/${BUNDLE_ID}/bin:$PATH"
   export GIT_EXEC_PATH=$(ls -d /opt/nordic/ncs/toolchains/${BUNDLE_ID}/Cellar/git/*/libexec/git-core)
   source <SDK_PATH>/zephyr/zephyr-env.sh
   ```

### Step 3.5: Update NCS Main Branch (if using main)

**When using the NCS main branch**, always update to the latest before building:

1. **Pull latest nrf repository changes**:
   ```sh
   cd /opt/nordic/ncs/main/nrf && git pull
   ```

2. **Update all west modules** (fetches dependencies like OpenThread, Matter, Memfault SDK, etc.):
   ```sh
   cd /opt/nordic/ncs/main && west update --narrow -o=--depth=1
   ```

The `--narrow -o=--depth=1` flags speed up the update by doing shallow clones.

**Why this is needed:**
- The main branch manifest changes frequently, adding/removing module dependencies
- Build errors like "Unmet or cyclic dependencies" indicate missing modules
- Running `west update` ensures all required modules (OpenThread, Matter, etc.) are present

### Step 4: Execute West Command

Combine setup and command in a single invocation:
```sh
export PATH="/opt/nordic/ncs/toolchains/<BUNDLE_ID>/bin:$PATH" && \
export GIT_EXEC_PATH=$(ls -d /opt/nordic/ncs/toolchains/<BUNDLE_ID>/Cellar/git/*/libexec/git-core) && \
source <SDK_PATH>/zephyr/zephyr-env.sh && \
cd /path/to/project && \
west <command>
```

Replace:
- `<BUNDLE_ID>`: From toolchain version lookup
- `<SDK_PATH>`: User-specified SDK location (e.g., `/opt/nordic/ncs/v3.2.1` or `/opt/nordic/ncs/main`)

## Common Toolchain Bundle IDs

| Toolchain Version | Bundle ID   |
|-------------------|-------------|
| v2.6.0            | 580e4ef81c  |
| v2.8.0            | 15b490767d  |
| v3.1.1            | 561dce9adf  |
| v3.2.0, v3.2.1    | 322ac893fe  |

For unlisted versions, always look up in `/opt/nordic/ncs/toolchains/toolchains.json`.

## Example Dialogs

### Same SDK and Toolchain Version
**User:** "Build my project at /opt/nordic/ncs/myApps/my_app"  
**Assistant:** "Which toolchain version would you like to use? (e.g., v3.2.1, v3.1.1)"  
**User:** "v3.2.1"  
**Assistant:** *Uses toolchain v3.2.1 + SDK at /opt/nordic/ncs/v3.2.1, then builds*

### Different SDK and Toolchain Versions
**User:** "Build my project using NCS main branch"  
**Assistant:** "Which toolchain version would you like to use with the main branch? (e.g., v3.2.1)"  
**User:** "v3.2.1 toolchain with SDK at /opt/nordic/ncs/main"  
**Assistant:** *Sets up toolchain v3.2.1, pulls latest nrf repo, runs west update, then builds*

### User Specifies Both Upfront
**User:** "Build with v3.2.1 toolchain and NCS main branch at /opt/nordic/ncs/main"  
**Assistant:** *Sets up environment, pulls latest, runs west update, then builds*

### Full Main Branch Build Sequence
```sh
# 1. Set up toolchain
export PATH="/opt/nordic/ncs/toolchains/<BUNDLE_ID>/bin:$PATH"
export GIT_EXEC_PATH=$(ls -d /opt/nordic/ncs/toolchains/<BUNDLE_ID>/Cellar/git/*/libexec/git-core)

# 2. Pull latest NCS main
cd /opt/nordic/ncs/main/nrf && git pull

# 3. Update west modules
cd /opt/nordic/ncs/main && west update --narrow -o=--depth=1

# 4. Source environment and build
source /opt/nordic/ncs/main/zephyr/zephyr-env.sh
cd /path/to/project && west build -b <board> -p
```

## Troubleshooting

### Git remote helper issues

After exporting PATH, verify git-remote-https resolves:
```sh
command -v git-remote-https || echo "Missing git-remote-https. Current GIT_EXEC_PATH: $GIT_EXEC_PATH"
```

### Avoid nrfutil sdk-manager --shell

Don't use `nrfutil sdk-manager … --shell` in automated terminals—it keeps the VS Code task busy after builds complete.

## Notes

- **Toolchain** provides build tools (west, cmake, compilers) via bundle ID
- **SDK path** provides source code and zephyr-env.sh
- Multiple terminals can have different PATH exports simultaneously
- Prompt may not change after PATH export; rely on the explicit bundle/version you set
- To switch versions, repeat setup with new bundle ID and/or SDK path (no terminal restart needed)
- When user says "NCS v3.2.1", clarify if they mean toolchain, SDK, or both
