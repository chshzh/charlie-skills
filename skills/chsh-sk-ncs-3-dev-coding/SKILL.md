---
name: chsh-sk-ncs-3-dev-coding
description: >-
  Load when implementing NCS project code from engineering specs. Reads specs
  in docs/dev-specs/. Checks zego bricks (/opt/nordic/ncs/v3.4.0/zego/bricks/)
  for reusable modules before writing new ones; uses nordic-wifi-webdash and
  nordic-wifi-memfault as reference implementations for patterns not yet in a
  brick. Use when specs are ready and code needs to be written or updated.
---

# chsh-sk-ncs-3-dev-coding — NCS Code Implementation

Implements NCS firmware from the engineering specs in `docs/dev-specs/`.
Reuses **zego bricks** for common modules; falls back to two real project
repositories as reference implementations for anything not yet extracted into a brick.

> **Prerequisite**: `docs/dev-specs/1-architecture.md` and at least one module
> spec must exist. If specs are missing, run **chsh-sk-ncs-2-dev-spec** first.

> **Knowledge sources**: Call `mcp__claude_ai_Nordic_MCP__nordicsemi_workflow_ncs` to load `embedded-code-guidance-ncs-zephyr` — covers NCS code style, Kconfig patterns, and driver idioms. Use `mcp__claude_ai_Nordic_MCP__nordicsemi_search_sources` for API lookups, board targets, and driver symbol names.

---

## Reusable Modules — check zego bricks first

Before creating any `src/modules/<name>/`, check whether it already exists as a
**zego brick** at `/opt/nordic/ncs/v3.4.0/zego/bricks/` — a standalone, portable
Zephyr module with its own Kconfig, hardware-abstraction backends, and spec doc.
If it matches, **wire it in via Kconfig — do not copy its source or re-derive its
state machine** from an app that used to implement the same thing inline.

| Brick | Directory | Covers | Spec |
|-------|-----------|--------|------|
| button | `zego/bricks/button/` | GPIO button, debounce, gesture classification (click/double-click/long-press), `BUTTON_CHAN` | `zego/bricks/button/docs/button-spec.md` |
| led | `zego/bricks/led/` | Per-LED SMF, ROTATE/BLINK/BREATHE effects, `LED_CMD_CHAN`/`LED_STATE_CHAN` | `zego/bricks/led/docs/led-spec.md` |
| wifi (`app_main`) | `zego/bricks/wifi/` | Startup banner, Wi-Fi mode selector + NVS persistence, `zego_wifi_mode` shell command | `zego/bricks/wifi/docs/wifi-spec.md` |
| network | `zego/bricks/network/` | Wi-Fi event dispatch (STA/SoftAP/P2P_GO/P2P_GC), DHCP, weak-hook callback API | `zego/bricks/network/docs/network-spec.md` |
| wifi_ble_prov | `zego/bricks/wifi_ble_prov/` | BLE GATT Wi-Fi credential provisioning | `zego/bricks/wifi_ble_prov/docs/wifi-ble-prov-spec.md` |
| ux | `zego/bricks/ux/` | Button-gesture-to-action wiring, LED Wi-Fi-state feedback, weak-hook gesture overrides | `zego/bricks/ux/docs/ux-spec.md` |
| ntp | `zego/bricks/ntp/` | SNTP time sync, real-world `CLOCK_REALTIME` | `zego/bricks/ntp/docs/ntp-spec.md` |
| memonitor | `zego/bricks/memonitor/` | Periodic heap/thread watermark sampler, `MEMONITOR_CHAN` | `zego/bricks/memonitor/docs/memonitor-spec.md` |

Wiring pattern (from any brick's spec, or `zego/README.md`):

```cmake
# CMakeLists.txt — before find_package(Zephyr ...)
get_filename_component(ZEGO_BUTTON_DIR ${CMAKE_CURRENT_SOURCE_DIR}/../zego/bricks/button REALPATH)
list(APPEND EXTRA_ZEPHYR_MODULES ${ZEGO_BUTTON_DIR})
```
```conf
# prj.conf
CONFIG_ZEGO_BUTTON=y
```

See `zego/nordic-wifi-app-template/` for a complete app wiring all eight bricks
together in one `CMakeLists.txt`/`prj.conf` — the fastest way to see the pattern
end to end, and itself built with `chsh-sk-ncs-0-workflow`.

---

## Reference Implementations — patterns not (yet) in a brick

For everything else, browse these two real projects:

| Repo | Patterns to reference |
|------|-----------------------|
| [`nordic-wifi-webdash`](https://github.com/chshzh/nordic-wifi-webdash) | HTTP webserver with gzip static assets from flash, REST API design, event-triggered service start on zbus (`CLIENT_CONNECTED_CHAN`) |
| [`nordic-wifi-memfault`](https://github.com/chshzh/nordic-wifi-memfault) | Memfault metrics/coredump/OTA, disconnect-time log persist to external flash, optional MQTT/HTTPS/CDR modules via Kconfig flags |

Use the `gh` CLI to read specific files from these public repos when
implementing a similar module (e.g.
`gh api repos/chshzh/nordic-wifi-webdash/contents/<path> --jq '.content' | base64 -d`).

---

## Step 0 — Read Inputs

```bash
cat docs/dev-specs/1-architecture.md   # module map, Zbus channels, boot order
ls docs/dev-specs/                   # all module specs
git log --oneline -5                 # recent commits
ls src/modules/ 2>/dev/null          # existing modules
```

Identify what needs to be created vs updated:
- **New project**: all modules are new → follow Steps 1–2 for each
- **Specs changed**: compare spec Revision History against `git log -- docs/dev-specs/` → implement only changed modules
- **Bug fix**: no spec change → go directly to the affected module

> **Guard — check before writing any code.** If the conversation contains
> discussed-but-undocumented changes: requirements changed → route to
> **chsh-sk-ncs-1-pm-prd** first; only implementation/spec details changed → route to
> **chsh-sk-ncs-2-dev-spec** first. Never touch `src/` until PRD and specs are current.
> "Start implementation" does not bypass this — it's the same rule
> `chsh-sk-ncs-0-workflow`'s Phase 3 section documents, restated here because this is
> normally where you actually enter, not there.

---

## Step 1 — Browse Reference Implementations

For each module in the spec, first check the zego bricks table above. Only fall
back to these reference repos for what's not covered there:

```
# SMF module (state machine + Zbus publish) — a brick already covers this shape:
# → zego/bricks/button/  (5-state gesture FSM, BUTTON_CHAN publish) — wire it in, don't reimplement
# → zego/bricks/led/     (per-LED SMF, LED_CMD_CHAN/LED_STATE_CHAN) — wire it in, don't reimplement
# Write a new SMF module only for logic that isn't hardware-button/LED shaped.

# SYS_INIT + Zbus listener/subscriber module (no SMF) — also often already a brick:
# → zego/bricks/network/    (Wi-Fi event dispatch, weak-hook callback API)
# → zego/bricks/memonitor/  (periodic sampler, MEMONITOR_CHAN publish)
# → reference: nordic-wifi-memfault/src/modules/app_memfault/core/ (zbus subscriber, upload on connect — Memfault-specific, not a brick)

# Library wrapper module (wraps external SDK or Zephyr subsystem) — not brick-covered:
# → reference: nordic-wifi-memfault/src/modules/app_memfault/     (Memfault SDK wrapper with core/metrics/ota/cdr)
# → reference: nordic-wifi-webdash/src/modules/webserver/         (Zephyr HTTP server + REST API wrapper)

# Main.c + Kconfig + CMakeLists.txt patterns
# → zego/nordic-wifi-app-template/  (wires all 8 bricks via EXTRA_ZEPHYR_MODULES — the current canonical pattern)
# → reference: nordic-wifi-memfault/CMakeLists.txt, Kconfig, prj.conf  (shell disabled, ZMS settings — Memfault-specific)
# → reference: nordic-wifi-webdash/CMakeLists.txt, Kconfig, prj.conf   (shell enabled, NVS settings — webserver-specific)
```

Look for: how Zbus channels are declared, how `SYS_INIT` is used, how Kconfig
guards modules, how callbacks are wired to the rest of the app.

---

## Step 2 — Implement

For each module from the spec (skip to Step 2 item 2 if it's a zego brick — no
`src/modules/<name>/` to create, just wire the Kconfig):

1. Create `src/modules/<name>/` with:
   - `<name>.c` — state machine/thread loop, Zbus integration, callback implementations
   - `<name>.h` — public API and type declarations (if needed)
   - `Kconfig.<name>` — module Kconfig with `CONFIG_APP_<MODULE>_MODULE`
   - `CMakeLists.txt` — `zephyr_library_sources` guarded by `CONFIG_APP_<MODULE>_MODULE`

2. Wire into top-level:
   - `Kconfig`: `rsource "src/modules/<name>/Kconfig.<name>"`
   - `prj.conf`: `CONFIG_APP_<MODULE>_MODULE=y` + required Kconfig from spec
   - `CMakeLists.txt`: `add_subdirectory(src/modules/<name>)`

3. Set PRD and specs versions in `prj.conf`:
   ```kconfig
   # Track which PRD and specs revision this code was written against.
   # Copy the newest Changelog timestamp from each document.
   CONFIG_APP_PRD_VERSION="YYYY-MM-DD-HH-MM"    # from docs/pm-prd/PRD.md Changelog
   CONFIG_APP_SPECS_VERSION="YYYY-MM-DD-HH-MM"  # from docs/dev-specs/0-overview.md Changelog
   ```
   Update these values every time the code is re-synced to a new PRD or specs revision.
   They appear in the boot banner as `PRD: ...` and `Specs: ...`.

4. **Host test — pure-logic modules only.** If this module has a pure-logic entry
   point (parsing, credential validation, a state-transition decision — a function
   that takes plain data in and returns plain data out, with no `zephyr/drivers/*`
   call in its path), write a `ztest` suite and run it on `native_sim` **before**
   building for the target board. This is the fast inner loop: no board, no UART, no
   wait — iterate here until it passes, then move to Step 5.

   Don't force it. If the module's logic isn't already separable from its hardware
   I/O, restructuring it just to make it testable is not worth the detour — skip
   host testing for that module and go straight to Step 5.

   ```
   tests/modules/<name>/
   ├── CMakeLists.txt   # pulls in src/modules/<name>/<name>.c directly — the real
   │                    # module, never a copy — so a passing test means the shipped
   │                    # code works, not a reimplementation of it
   ├── prj.conf         # CONFIG_ZTEST=y + whatever Kconfig the module itself needs
   ├── testcase.yaml    # integration_platforms: [native_sim]
   └── src/main.c       # ZTEST_SUITE + ZTEST() cases
   ```

   `tests/modules/<name>/CMakeLists.txt`:
   ```cmake
   cmake_minimum_required(VERSION 3.20.0)
   find_package(Zephyr REQUIRED HINTS $ENV{ZEPHYR_BASE})
   project(test_<name>)
   target_sources(app PRIVATE
     src/main.c
     ${CMAKE_CURRENT_SOURCE_DIR}/../../../src/modules/<name>/<name>.c
   )
   target_include_directories(app PRIVATE ${CMAKE_CURRENT_SOURCE_DIR}/../../../src/modules/<name>)
   ```

   `tests/modules/<name>/prj.conf`:
   ```kconfig
   CONFIG_ZTEST=y
   ```

   `tests/modules/<name>/testcase.yaml`:
   ```yaml
   common:
     integration_platforms:
       - native_sim
     tags:
       - unit
   tests:
     modules.<name>.unit: {}
   ```

   `tests/modules/<name>/src/main.c`:
   ```c
   #include <zephyr/ztest.h>
   #include "<name>.h"

   ZTEST_SUITE(<name>_suite, NULL, NULL, NULL, NULL, NULL);

   ZTEST(<name>_suite, test_valid_input)
   {
       /* call the module's pure-logic function directly; assert on the result */
   }

   ZTEST(<name>_suite, test_rejects_malformed_input)
   {
       /* same, with bad input — assert the error path */
   }
   ```

   Build and run:
   ```bash
   west build -b native_sim tests/modules/<name> -d build/tests/<name>
   ./build/tests/<name>/zephyr/zephyr.exe
   ```
   Fix, rebuild, rerun — until every case passes.

5. Build and verify on target hardware:
   ```bash
   west build -p -b <board>
   ```
   Fix all warnings. Confirm UART output matches spec test points.

6. Set up CI (new projects only):
   - Read the template at `.claude/skills/chsh-sk-ncs-3-dev-coding/build_template.yml`
   - Replace all `<PLACEHOLDER>` tokens with the project-specific values:
     - `<APP_NAME>` → repo/directory name
     - `<APP_DISPLAY_NAME>` → human-readable title
     - `<GITHUB_REPO>` → `owner/repo` slug
     - `<CMAKE_ARGS_NRF54LM20DK>` → extra cmake args for nRF54LM20DK (e.g. `"-DSHIELD=nrf7002eb2"`)
     - `<CMAKE_ARGS_NRF7002DK>` → extra cmake args for nRF7002DK (empty `""` if none)
     - `<CMAKE_ARGS_NRF5340_AUDIO_DK>` → extra cmake args for nRF5340 Audio DK (e.g. `"-DSHIELD=nrf7002ek"`)
   - Write the result to `.github/workflows/build.yml`
   - If `APP_VERSION_STRING` is not used in the app's `CMakeLists.txt`, remove the version-injection block from the Build step.

7. **Check for doc drift before finishing.** Don't rely on session memory for
   this — anchor on the pins directly, since coding often spans more than one
   session:

   ```bash
   PRD_PIN_BUMP_SHA=$(git log -1 --format='%H' -G'CONFIG_APP_PRD_VERSION' -- prj.conf 2>/dev/null)
   SPEC_PIN_BUMP_SHA=$(git log -1 --format='%H' -G'CONFIG_APP_SPECS_VERSION' -- prj.conf 2>/dev/null)
   git log --oneline "${SPEC_PIN_BUMP_SHA}..HEAD" -- src/ 2>/dev/null   # already committed, undeclared
   git status --short -- src/ 2>/dev/null                              # not committed yet at all
   ```

   List commits since each pin's last bump plus anything still uncommitted, then
   classify against the same pattern list `chsh-ag-git` checks at commit time (its
   Step 2.5 "Step A — Classify the change severity") rather than re-deriving one. If
   nothing matches, skip this step silently.

   If something matches, call `AskQuestion` before Handoff:
   ```
   AskQuestion:
     prompt: "This session added <what> — docs don't cover it yet. Update them first?"
     options:
       - "Update specs now (chsh-sk-ncs-2-dev-spec, Mode B)"
       - "Update specs and PRD now (also chsh-sk-ncs-1-pm-prd, Mode D) — it's user-visible"
       - "Skip — I'll update docs separately"
   ```
   If the user updates docs, re-read the new Changelog timestamps and bump the pins
   (Step 2, item 3) before moving on.

8. Handoff:
   > "Implementation complete. Run **chsh-sk-ncs-4-test-verification** for
   > Verification (code review, clean build, host tests, doc audit — no hardware); then run
   > **chsh-sk-ncs-4-test-validation** for hardware validation."

---

## SMF Implementation Pattern

When a module spec calls for an SMF state machine, use this canonical structure.

**User object** — `smf_ctx` must be the first member:

```c
struct my_module_obj {
    struct smf_ctx ctx;   /* MUST be first */
    struct my_event event;
    /* other state-local data */
} s_obj;
```

**State table** — one entry per state, parent and initial transitions explicit:

```c
static const struct smf_state states[] = {
    /* Parent state: entry, run, exit, parent=NULL, initial→child */
    [STATE_RUNNING] = SMF_CREATE_STATE(running_entry, running_run, NULL,
                                       NULL, &states[STATE_IDLE]),
    /* Child states */
    [STATE_IDLE]    = SMF_CREATE_STATE(idle_entry, idle_run, idle_exit,
                                       &states[STATE_RUNNING], NULL),
    [STATE_ACTIVE]  = SMF_CREATE_STATE(active_entry, active_run, active_exit,
                                       &states[STATE_RUNNING], NULL),
};
```

**Thread loop** — run-to-completion: one msgq pop = one `smf_run_state()` call:

```c
static void my_module_thread(void *a, void *b, void *c)
{
    smf_set_initial(SMF_CTX(&s_obj), &states[STATE_RUNNING]);
    while (1) {
        k_msgq_get(&my_msgq, &s_obj.event, K_FOREVER);
        int rc = smf_run_state(SMF_CTX(&s_obj));
        if (rc) {
            LOG_INF("state machine terminated: %d", rc);
            break;
        }
    }
}
```

**Run function return values:**

| Return | Meaning |
|--------|---------|
| `SMF_EVENT_PROPAGATE` | Event not consumed here — pass up to parent's run. **Use as default.** |
| `SMF_EVENT_HANDLED` | Event consumed — parent run will NOT be called. |
| *(calling `smf_set_state()`)* | Implicitly stops propagation to parent, regardless of return value. |

**Required Kconfig for hierarchical state machines:**

```kconfig
CONFIG_SMF_ANCESTOR_SUPPORT=y      # enables parent state pointer
CONFIG_SMF_INITIAL_TRANSITION=y    # enables parent→child default transition
```

Both must be set when `SMF_CREATE_STATE` uses non-NULL `parent` or `initial` parameters — without them those parameters are silently ignored.

---

## Logging Standards

Every module **must** have structured logging at all four levels. This enables post-mortem debugging without rebuilding.

| Level | When to use | Example |
|-------|-------------|---------|
| `LOG_ERR` | Any non-zero return from a hardware or OS API; anything that stops the module working | `LOG_ERR("dk_set_led_on(%d) failed: %d", n, ret)` |
| `LOG_WRN` | Invalid input, out-of-range index, unexpected-but-recoverable state | `LOG_WRN("LED index %d out of range (max %d)", n, NUM_LEDS - 1)` |
| `LOG_INF` | Module init with key config values; significant mode/state transitions | `LOG_INF("LED %d BLINK period=%u ms", n, period)` |
| `LOG_DBG` | Per-event / per-cycle data (called at high frequency; gated by log level at compile/runtime) | `LOG_DBG("LED %d -> %s", n, on ? "ON" : "OFF")` |

**Rules:**

1. **Never silently discard a non-zero return value.** Always `LOG_ERR` it. Silent failures cause invisible hardware bugs (e.g. `dk_set_led_on()` returns `-ENODEV` when GPIO isn't configured — if not logged, the LED just doesn't light up with no indication why).
2. **Log init params at `LOG_INF`.** Every `SYS_INIT` function must log the key Kconfig values it was compiled with so the boot log alone is sufficient to reproduce a field issue.
3. **Log direction changes and milestones at `LOG_DBG`**, not every tick. For a state machine: log entry to each state. For a timer loop: log on direction/phase change, not every fire.
4. **Log with context**: include the resource index (LED number, button index, etc.) in every message so logs from multi-instance modules are unambiguous.

---

## Gotchas

| Gotcha | Detail |
|--------|--------|
| Zbus publish in ISR context | `zbus_chan_pub()` is not ISR-safe — use a work queue or `k_msgq` to defer from ISR |
| `SYS_INIT` order for library wrappers | `APPLICATION` level runs after `POST_KERNEL`; use explicit priority numbers when one lib depends on another |
| Kconfig `depends on` vs `select` | Use `depends on` for optional features; use `select` only when the module unconditionally requires a lib (mirrors reference repos) |
| `prj.conf` overlay order | `EXTRA_CONF_FILE` overlays win; put test-only settings in overlays, not `prj.conf` |
| `west build -p` vs incremental | Always use `-p` when switching board or overlay set; incremental builds can silently keep stale objects |
| `smf_set_state()` in exit functions | Generates a log warning and **no transition occurs**. Only call `smf_set_state()` from entry or run functions. |
| `SMF_EVENT_PROPAGATE` vs `SMF_EVENT_HANDLED` | Return `SMF_EVENT_PROPAGATE` by default so unhandled events bubble to parents. Only return `SMF_EVENT_HANDLED` when you explicitly consumed the event and do not want the parent to see it. Note: calling `smf_set_state()` already stops propagation regardless of the return value. |
| Missing HSM Kconfig | `CONFIG_SMF_ANCESTOR_SUPPORT=y` and `CONFIG_SMF_INITIAL_TRANSITION=y` are required for parent/initial transitions. Without them, the `parent` and `initial` parameters in `SMF_CREATE_STATE` are silently ignored — the hierarchy simply does not exist at runtime. |
| Module won't build for `native_sim` | Usually means the module reaches across the hardware boundary you meant to isolate (a driver call inside the "pure-logic" function). Treat it as a signal to fix the boundary, not an obstacle to route around — don't add driver stubs just to force the build green. |

---

## Related Skills

| Task | Skill |
|------|-------|
| Generate engineering specs | `chsh-sk-ncs-2-dev-spec` |
| Phase 4 Verification & Test | `chsh-sk-ncs-4-test-verification` |
| Debug build failures, UART analysis | `chsh-sk-ncs-3-dev-debug` |
| Commit after implementation | `chsh-sk-ncs-git-commit` |
| Full lifecycle orchestration | `chsh-sk-ncs-0-workflow` |

## Self-Update Policy

At the **end of each conversation**, review what was discovered and check
whether any facts in this skill are new, corrected, or outdated (e.g. new
reference repo patterns, Zbus gotchas, Kconfig conventions found during
implementation).

If updates are warranted:
1. Collect all proposed changes with a brief rationale for each.
2. Present a summary to the user and ask for approval using `AskQuestion`.
3. Apply approved updates to this file immediately.

Do **not** modify this skill mid-conversation unless the user explicitly asks.
