# LCD35B-show-V2 Modernization Plan

**Target**: Make the 3.5" LCD35B-v2 display work on modern Raspberry Pi OS (BookWorm, 32-bit & 64-bit) without breaking user systems.

**Current Status**: Original script is from ~2015, incompatible with:
- BookWorm (changed system architecture, X11 deprecated)
- 64-bit OS (dependency conflicts, missing arm64 packages)
- Pi 5 (kernel/firmware changes)
- Modern systemd (replaced rc.local/inittab)

---

## Architecture Overview

```
lcd35b-install.sh (main installer, ~150 lines)
├── Pre-flight checks (OS version, arch, backups)
├── config.txt modification (safe append, not overwrite)
├── Device tree overlay installation
├── Touchscreen driver setup (if needed)
├── Optional: fbcp framebuffer copy (if needed)
└── Uninstall script generation
```

---

## Phase 1: Pre-Flight & Validation (Week 1)

### 1.1 System Detection & Validation
**Status**: ❌ TODO  
**Owner**: TBD

- [ ] Detect OS version (`lsb_release -cs`)
- [ ] Validate BookWorm support (32/64-bit)
- [ ] Check Pi model (3B+, 4, 5)
- [ ] Verify SPI/I2C already enabled or enable them
- [ ] Create backup of `/boot/config.txt` before ANY modifications
- [ ] Verify sufficient disk space
- [ ] Check for conflicting overlays already loaded

**Files to create**:
- `scripts/detect-system.sh` — Outputs JSON with system info
- `scripts/validate-backups.sh` — Backup critical files to timestamped dir

**Test cases**:
```bash
# Should work
- Pi 4B, BookWorm 32-bit (latest)
- Pi 4B, BookWorm 64-bit (latest)
- Pi 3B+, BookWorm 32-bit

# Should fail gracefully with helpful message
- Pi 2 (too old)
- Bullseye (older, but warn)
- X86 Ubuntu (clear error)
```

---

### 1.2 Minimal Manual Test
**Status**: ❌ TODO  
**Owner**: You (Doesntevenwork)

Before writing the script, verify display works with manual steps:

```bash
# On your Pi, manually test:
sudo cp /boot/config.txt /boot/config.txt.backup

# Append overlay (don't overwrite)
echo "dtoverlay=waveshare35b-v2" | sudo tee -a /boot/config.txt

# Copy device tree
sudo cp waveshare35b-v2-overlay.dtbo /boot/overlays/waveshare35b-v2.dtbo

# Reboot
sudo reboot

# Does display work? Test rotation?
```

**Outcome**: If this works → fbcp not needed. If not → document what's missing.

---

## Phase 2: Safe Config Modifications (Week 1-2)

### 2.1 Config.txt Handler
**Status**: ❌ TODO  
**Owner**: TBD

Replace dangerous `sudo cp` with safe append/sed:

```bash
# OLD (dangerous): cp -rf ./boot/config-35b-v2.txt /boot/config.txt
# NEW (safe):
add_or_update_config_entry() {
  local key=$1
  local value=$2
  local config_file="/boot/config.txt"
  
  # Backup first
  sudo cp "$config_file" "${config_file}.lcd35b.backup"
  
  # Check if key exists
  if grep -q "^${key}=" "$config_file"; then
    # Update existing
    sudo sed -i "s/^${key}=.*/key=${value}/" "$config_file"
  else
    # Append new
    echo "${key}=${value}" | sudo tee -a "$config_file" > /dev/null
  fi
}
```

**Files to create**:
- `scripts/config-manager.sh` — All config manipulation logic
- `config-overlays/lcd35b-v2-bookworm.txt` — Modern-safe config template

**Test cases**:
- [ ] Existing key update works
- [ ] New key append works
- [ ] Backup created
- [ ] Rollback works

---

### 2.2 Device Tree Overlay Installation
**Status**: ⚠️ PARTIAL (overlay exists, just needs safe install)

```bash
# Copy overlay to bootloader
sudo cp ./waveshare35b-v2-overlay.dtb /boot/overlays/waveshare35b-v2.dtbo

# Enable in config.txt
add_or_update_config_entry "dtoverlay" "waveshare35b-v2"
```

**Files involved**:
- `waveshare35b-v2-overlay.dtb` — Existing, verify it works on BookWorm
- Document: When does this need recompilation?

---

## Phase 3: Touchscreen Driver (Week 2)

### 3.1 ADS7846 Touchscreen Setup
**Status**: ⚠️ PARTIAL (old X11 config exists, likely broken)

**Current approach** (broken):
- Install `xinput-calibrator` (32-bit only, not on 64-bit)
- Copy calibration file to `/etc/X11/xorg.conf.d/` (X11 might not exist on modern systems)

**Modern approach**:
1. **Check if X11 exists** → if not, skip X11 calibration
2. **Use device tree ADS7846 overlay** instead:
   ```
   # In config.txt:
   dtoverlay=ads7846,cs=1,penirq=17,penirq_pull=2,speed=1000000,keep_vref_on=1,swapxy=1,pmax=255,xohms=60
   ```
3. **Optional manual calibration** (only if user requests it)

**Files to create**:
- `scripts/touchscreen-setup.sh` — Detect & setup touch driver
- `config-overlays/ads7846-35b-v2.txt` — ADS7846 parameters for LCD35B
- `docs/TOUCHSCREEN-CALIBRATION.md` — How to calibrate if needed

**Test cases**:
- [ ] Detect if X11 desktop installed
- [ ] Load ADS7846 overlay
- [ ] Test touch input with `evtest /dev/input/event*`

---

## Phase 4: Framebuffer Copy (fbcp) - OPTIONAL (Week 2-3)

### 4.1 Evaluate if fbcp Needed
**Status**: ❌ TODO (must test manually first)

Questions:
- Does display work WITHOUT fbcp on BookWorm?
- Is fbcp even needed for SPI displays on modern Pi?

**If NOT needed**: Skip this entirely. Remove `fbcp` from old script.

**If needed**: Build it safely:

```bash
# Current fbcp location: ./rpi-fbcp/
# Problem: CMake cache conflicts, hardcoded paths

# Fix:
if [ ! -d "/tmp/fbcp-build" ]; then
  mkdir -p /tmp/fbcp-build
  cd /tmp/fbcp-build
  cmake ../../LCD-show/rpi-fbcp || exit 1
  make || exit 1
  sudo install fbcp /usr/local/bin/
  # Don't leave build artifacts in repo
  cd /
  rm -rf /tmp/fbcp-build
fi
```

**Files to create**:
- `scripts/fbcp-builder.sh` — Safe fbcp build/install
- Document: When fbcp is needed vs when it's optional

---

## Phase 5: Safety & Rollback (Week 2)

### 5.1 Automatic Uninstall Script
**Status**: ❌ TODO

Generate an `uninstall-lcd35b.sh` script during installation:

```bash
#!/bin/bash
# Generated: 2024-XX-XX by lcd35b-install.sh
# Restores system to pre-installation state

# Restore config.txt
sudo cp /boot/config.txt.lcd35b.backup /boot/config.txt

# Remove overlay
sudo rm -f /boot/overlays/waveshare35b-v2.dtbo

# Remove fbcp if installed
sudo rm -f /usr/local/bin/fbcp

# Remove X11 configs if installed
sudo rm -f /etc/X11/xorg.conf.d/99-calibration.conf-35b

echo "Uninstall complete. Reboot to apply changes: sudo reboot"
```

**Files to create**:
- `scripts/generate-uninstall.sh` — Template generator

---

## Phase 6: Testing & Documentation (Week 3)

### 6.1 Test Matrix
**Status**: ❌ TODO

| Hardware | OS | Version | 32/64 | Display | Touch | Status |
|----------|----|----|----|----|----|----|
| Pi 3B+ | BookWorm | Latest | 32-bit | ? | ? | TODO |
| Pi 4B | BookWorm | Latest | 32-bit | ? | ? | TODO |
| Pi 4B | BookWorm | Latest | 64-bit | ? | ? | TODO |
| Pi 5 | BookWorm | Latest | 64-bit | ? | ? | TODO |

### 6.2 Documentation
**Files to create**:
- `README-35B-V2.md` — Installation guide, supported systems, troubleshooting
- `docs/ARCHITECTURE.md` — How the script works, what each part does
- `docs/TROUBLESHOOTING.md` — Common issues & solutions
- `docs/TESTING.md` — How to test the driver

---

## Phase 7: CI/CD & Quality Assurance (Week 3)

### 7.1 Shell Script Linting
**Status**: ❌ TODO

```bash
# Use shellcheck to validate
shellcheck -x scripts/*.sh lcd35b-install.sh

# Set error handling
set -euo pipefail  # Exit on error, undefined vars, pipe failures
```

### 7.2 Version Management
**Status**: ❌ TODO

Tag releases:
```bash
git tag -a v1.0.0-bookworm -m "First modernized release"
```

---

## File Structure (Target Layout)

```
LCD-show/
├── lcd35b-install.sh                          # Main installer (rewritten)
├── lcd35b-uninstall.sh                        # Generated during install
├── LCD35B-show-V2                             # Keep for backward compat (deprecated)
│
├── scripts/
│   ├── detect-system.sh                       # OS/hardware detection
│   ├── config-manager.sh                      # Safe config.txt editing
│   ├── touchscreen-setup.sh                   # ADS7846 setup
│   ├── fbcp-builder.sh                        # Optional fbcp build
│   ├── validate-backups.sh                    # Backup verification
│   └── generate-uninstall.sh                  # Generate uninstall script
│
├── config-overlays/
│   ├── lcd35b-v2-bookworm.txt                 # config.txt template
│   ├── ads7846-35b-v2.txt                     # Touchscreen params
│   └── pre-install-checks.sh                  # Pre-flight validation
│
├── docs/
│   ├── README-35B-V2.md                       # Installation guide
│   ├── ARCHITECTURE.md                        # Design docs
│   ├── TROUBLESHOOTING.md                     # Common issues
│   ├── TESTING.md                             # How to test
│   └── CHANGELOG.md                           # Version history
│
├── waveshare35b-v2-overlay.dtbo               # Keep existing
└── tests/                                     # (Optional) Test scripts
    ├── test-display.sh                        # Test display works
    ├── test-touch.sh                          # Test touchscreen
    └── test-rotation.sh                       # Test rotations
```

---

## Success Criteria

✅ **Installation Works**:
- [ ] Script runs without errors on BookWorm 32/64-bit
- [ ] Display appears after reboot
- [ ] Touchscreen responds to input
- [ ] HDMI still works (if user has external monitor)
- [ ] SSH still works (no bricking)

✅ **Rollback Works**:
- [ ] Uninstall script restores system
- [ ] No orphaned files left behind
- [ ] System boots normally after uninstall

✅ **Documentation**:
- [ ] README has clear installation steps
- [ ] Troubleshooting covers known issues
- [ ] No user has to guess what's happening

---

## Next Steps

1. **Immediate** (this session):
   - [ ] You manually test Phase 1.2 (manual config)
   - [ ] Document if display works without fbcp
   - [ ] Fork repo to your account

2. **This week**:
   - [ ] Phase 1: System detection & validation
   - [ ] Phase 2: Config.txt safe handler
   - [ ] Phase 3: Touchscreen setup

3. **Next week**:
   - [ ] Phase 4: fbcp (if needed)
   - [ ] Phase 5: Rollback & uninstall
   - [ ] Phase 6: Testing

4. **Final week**:
   - [ ] CI/CD setup
   - [ ] Documentation finalization
   - [ ] First release (v1.0.0-bookworm)

---

## Questions for Clarity

Before we start coding:

1. **Have you tested the manual 3-step setup yet?** (config.txt append, overlay copy, reboot)
   - Does display work?
   - Do you need fbcp?
   
2. **What Pi model & OS version are you actually using?**
   - (This becomes your primary test target)

3. **Do you want to support older OS versions** (Bullseye) or focus only on BookWorm?

4. **Should we aim to support OTHER LCD35B variants** (A, C, not just V2)?

Answers will help prioritize which phases to tackle first.
