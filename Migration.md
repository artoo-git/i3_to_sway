# Migrating from X11/i3 to Wayland/Sway on Lenovo ThinkPad X270

## Initial Conditions

**Hardware:**
- Lenovo ThinkPad X270
- Intel HD Graphics 520 (Skylake GT2, Gen 9)
- Docking station with 2x Dell P2722H external monitors (1920x1080) via DisplayPort

**Previous Setup:**
- Debian GNU/Linux (forky/sid)
- Kernel: 6.17.13+deb14-amd64
- X11 with i3 window manager
- Remmina for RDP connections to xxx network via gateway
- Microsoft Teams screen sharing capability

**Critical Workflows:**
1. RDP access to `business-managed windows machine` via `business-managed` gateway
2. MS-Teams screen sharing for remote presentations
3. Multi-monitor docking/undocking workflow
4. Proper power management (suspend on lid close, display power-off when idle)

## Migration Objectives

1. Move to modern Wayland compositor (Sway) for better performance and security
2. Maintain all critical workflows (RDP, Teams screen sharing)
3. Preserve multi-monitor docking functionality
4. Ensure proper power management and idle handling
5. Achieve stability comparable to X11 setup

## Issues Encountered and Solutions

### 1. Remmina RDP Client Crashes

**Problem:**
Remmina crashed immediately on connection attempts to RDP gateway in both Wayland and X11 modes after migration. Error:
```
[ERROR][com.freerdp.core.gateway.tsg] - Unexpected offset value: 120, expected: 0
```

**Root Cause:**
FreeRDP gateway protocol bug in versions 1.4.39 and 1.4.40 affecting Kerberos authentication handling. The client attempts Kerberos first, fails with "Cannot find KDC for realm", then should retry with NTLM but crashes instead.

**Solutions Attempted:**
- Downgrading FreeRDP (unsuccessful - bug present in both versions)
- Flatpak Remmina (unsuccessful - same underlying FreeRDP)
- GNOME Connections (insufficient - lacks RDP Gateway support)

**Final Solution:**
Use `wlfreerdp3` command-line client as replacement:

```bash
#!/bin/bash
# ~/bin/xxx-rdp
wlfreerdp3 /v:xxxxxxxxx.xxxxxxxxx.xxx.ac.uk \
           /gateway:g:xxxxxxxxx.xxxxxxxxx.xxx.ac.uk,u:xxxxxxxxx,d:xxxxxxxxx \
           /u:xxxxxxxxx \
           /d:xxxxxxxxx \
           +clipboard \
           /cert:ignore \
           /sec:nla \
           /tls:seclevel:0 \
           /f \
           /smart-sizing \
           +auto-reconnect \
           /bpp:32 \
           /sound \
           +compression \
           /network:auto \
           /gfx:AVC444 \
           -grab-keyboard \
           /auth-pkg-list:!kerberos,ntlm
```

**Key flags:**
- `/auth-pkg-list:!kerberos,ntlm` - Disable Kerberos to avoid double-authentication
- `/f` - Fullscreen mode
- `/smart-sizing` - Scale remote desktop to window
- `+clipboard` - Bidirectional clipboard (works perfectly)
- Exit fullscreen: `Ctrl+Alt+Enter`

**Alternative for GUI access:**
Launch Remmina with XWayland backend:
```bash
GDK_BACKEND=x11 remmina
```

Create desktop launcher override in `~/.local/share/applications/remmina.desktop`:
```ini
[Desktop Entry]
Exec=env GDK_BACKEND=x11 remmina
```

### 2. Teams Screen Sharing

**Problem:**
Teams-for-linux desktop application has poor Wayland compatibility and unreliable screen sharing.

**Solution:**
Use browser-based Teams (teams.microsoft.com) instead of desktop app.

**Configuration required:**
Set Electron Wayland hint in Sway config:
```bash
exec export ELECTRON_OZONE_PLATFORM_HINT=auto
```

Enable PipeWire screen sharing through xdg-desktop-portal-wlr (see Portal Configuration section).

**Result:** Full screen sharing capability via browser with native Wayland performance.

### 3. Portal Configuration for Screen Sharing

**Problem:**
`xdg-desktop-portal-gtk` repeatedly crashed trying to access X11 display `:0` in Wayland environment, causing system instability.

**Root Cause:**
Portal-gtk was not receiving proper Wayland environment variables from systemd user services.

**Solution:**

1. **Configure portal preferences** (`~/.config/xdg-desktop-portal/sway-portals.conf`):
```ini
[preferred]
default=wlr;gtk
org.freedesktop.impl.portal.ScreenCast=wlr
org.freedesktop.impl.portal.Screenshot=wlr
```

2. **Export environment to systemd** (in `~/.config/sway/config`):
```bash
# MUST be early, before other exec lines
exec dbus-update-activation-environment --systemd WAYLAND_DISPLAY XDG_CURRENT_DESKTOP=sway XDG_SESSION_TYPE=wayland
exec systemctl --user import-environment WAYLAND_DISPLAY XDG_CURRENT_DESKTOP XDG_SESSION_TYPE
```

3. **Disable problematic portal-gtk:**
```bash
systemctl --user mask xdg-desktop-portal-gtk
systemctl --user restart xdg-desktop-portal
```

**Result:** Stable portal operation with wlr handling all screen capture needs. Teams browser screen sharing works reliably.

### 4. Display Management - Kanshi Configuration

**Problem:**
Display port numbers changed between sessions (DP-7/DP-6 → DP-5/DP-4), causing Kanshi profile matching failures. Initial attempts with wildcard matching failed because identical monitors couldn't be distinguished.

**Solution:**
Match displays by monitor serial numbers instead of port names.

**Kanshi configuration** (`~/.config/kanshi/config`):
```
profile docked {
    output "Dell Inc. DELL P2722H 9S67DJ3" enable mode 1920x1080@60Hz position 0,0 scale 1
    output "Dell Inc. DELL P2722H 9267DJ3" enable mode 1920x1080@60Hz position 1920,0 scale 1
    output "Chimei Innolux Corporation 0x1239 Unknown" enable mode 1920x1080@60Hz position 939,1080 scale 1.2
}

profile laptop {
    output "Chimei Innolux Corporation 0x1239 Unknown" enable mode 1920x1080@60Hz position 0,0 scale 1.2
}
```

**To get monitor identifiers:**
```bash
swaymsg -t get_outputs
```

**Result:** Reliable automatic display configuration on dock/undock regardless of port number changes.

### 5. Sway Crashes on Display Power-Off

**Problem:**
Sway crashed consistently when running `swaymsg "output * power off"` while docked with external monitors. Debug logs showed:
```
[DEBUG] [wlr] [backend/drm/fb.c:147] Failed to get DMA-BUF from buffer
[DEBUG] [wlr] [backend/drm/drm.c:769] connector eDP-1: Failed to import buffer for scan-out
[DEBUG] [wlr] [backend/drm/drm.c:769] connector DP-4: Failed to import buffer for scan-out
```

**Root Cause:**
Intel i915 graphics driver DMA-BUF buffer import failures on Wayland, specifically affecting DisplayPort outputs when docked. The issue occurs continuously even before power-off command is issued, indicating a driver/wlroots interaction bug.

**System specifics:**
- Intel HD Graphics 520 (Skylake, Gen 9)
- Sway 1.11
- wlroots 0.19.1
- Kernel 6.17.13 (very new - potential KMS/DRM compatibility issue)

**Investigation findings:**
- FBC (Frame Buffer Compression) already disabled due to VT-d
- PSR (Panel Self Refresh) at auto (`-1`)
- DC (Display C-states) at auto (`-1`)
- Issue only occurs when external monitors connected
- Laptop screen only: power-off works fine
- External monitors: DMA-BUF import failures cause crash

**Attempted solutions:**
1. Kernel parameters `i915.enable_psr=0 i915.enable_dc=0` (not yet tested)
2. Newer kernel or wlroots version (pending)

**Working workaround:**
Only power off laptop screen (eDP-*) when idle, leave external monitors running.

**Swayidle configuration** (`~/.config/sway/config`):
```bash
exec swayidle -w \
    timeout 300 'swaylock -f -c 000000' \
    timeout 600 'swaymsg "output eDP-* power off"' \
    resume 'swaymsg "output eDP-* power on"' \
    before-sleep 'swaylock -f -c 000000'
```

**Timeline:**
- 5 minutes idle: Lock screen
- 10 minutes idle: Power off laptop screen only
- On resume: Power laptop screen back on
- Before sleep: Lock screen

**Result:** Stable idle management without crashes. External monitors consume minimal power when static anyway.

### 6. Autorandr Conflict

**Problem:**
Autorandr service (X11 display manager) was still active and conflicting with Kanshi on dock/undock events.

**Solution:**
Remove or Disable autorandr completely:
```bash
sudo apt purge autorandr
rm ~/.config/systemd/user/autorandr.service
systemctl --user daemon-reload
```

## Final Configuration Files

### Sway Config (`~/.config/sway/config`)

Key sections:

```bash
### Variables
set $mod Mod4
set $term foot
set $menu wofi --show drun

### Output configuration
output eDP-1 scale 1.20

### Environment setup (critical for portals)
exec dbus-update-activation-environment --systemd WAYLAND_DISPLAY XDG_CURRENT_DESKTOP=sway XDG_SESSION_TYPE=wayland
exec systemctl --user import-environment WAYLAND_DISPLAY XDG_CURRENT_DESKTOP XDG_SESSION_TYPE

### Idle configuration
exec swayidle -w \
    timeout 300 'swaylock -f -c 000000' \
    timeout 600 'swaymsg "output eDP-* power off"' \
    resume 'swaymsg "output eDP-* power on"' \
    before-sleep 'swaylock -f -c 000000'

for_window [class=".*"] inhibit_idle fullscreen
for_window [app_id=".*"] inhibit_idle fullscreen

### Kanshi for display management
exec kanshi
```

### Waybar Config (`~/.config/waybar/config`)

Memory module adjusted to match i3status format:
```json
"memory": {
    "interval": 5,
    "format": "MEM:{avail:0.1f}G"
}
```

### Portal Config (`~/.config/xdg-desktop-portal/sway-portals.conf`)

```ini
[preferred]
default=wlr;gtk
org.freedesktop.impl.portal.ScreenCast=wlr
org.freedesktop.impl.portal.Screenshot=wlr
```

## Testing and Verification

### Verify Portal Setup
```bash
# Check environment variables
echo $WAYLAND_DISPLAY  # Should show "wayland-1" or similar
echo $XDG_CURRENT_DESKTOP  # Should show "sway"

# Check portal status
systemctl --user status xdg-desktop-portal-wlr
systemctl --user status xdg-desktop-portal

# Verify gtk portal is masked
systemctl --user status xdg-desktop-portal-gtk  # Should show "masked"
```

### Verify Display Configuration
```bash
# Check current outputs
swaymsg -t get_outputs

# Test Kanshi is running
ps aux | grep kanshi

# Manually trigger profile reload
killall -SIGUSR1 kanshi
```

### Verify Idle Management
```bash
# Check swayidle is running
ps aux | grep swayidle

# Test manual screen lock
swaylock -f -c 000000

# Test screen power off (undocked only!)
swaymsg "output eDP-* power off"
sleep 2
swaymsg "output eDP-* power on"
```

## Current Status

### ✅ Working
- Sway window manager fully functional
- RDP access via wlfreerdp3 (clipboard, fullscreen, reconnect)
- Teams screen sharing via browser
- Multi-monitor docking with automatic configuration (Kanshi)
- Display scaling (1.2x on laptop screen)
- Waybar status bar
- Screenshot functionality (grim + slurp)
- Idle inhibit for fullscreen applications
- Suspend on lid close (including emergency wake on low battery)
- Screen lock after 5 minutes idle
- Laptop screen power-off after 10 minutes idle

### ⚠️ Known Limitations
- **Cannot power off external monitors when idle** - Causes i915 DMA-BUF crash
  - Workaround: Only power off laptop screen
  - Potential fix: Test kernel parameters `i915.enable_psr=0 i915.enable_dc=0`

### If ❌ Not Working
- Teams-for-linux desktop app (poor Wayland support)
  - Solution: Use browser Teams instead
- Native Remmina in Wayland mode (FreeRDP bug)
  - Solution: Use wlfreerdp3 or XWayland mode

### For Future Testing
1. Test kernel parameters to fix external monitor power-off:
   ```bash
   # /etc/default/grub
   GRUB_CMDLINE_LINUX_DEFAULT="quiet i915.enable_psr=0 i915.enable_dc=0"
   
   sudo update-grub
   sudo reboot
   ```

2. Monitor for FreeRDP updates fixing gateway authentication bug

3. Consider creating systemd user service for swayidle:
   ```ini
   # ~/.config/systemd/user/swayidle.service
   [Unit]
   Description=Idle manager for Wayland
   PartOf=graphical-session.target
   
   [Service]
   ExecStart=/usr/bin/swayidle -w timeout 300 'swaylock -f -c 000000' timeout 600 'swaymsg "output eDP-* power off"' resume 'swaymsg "output eDP-* power on"' before-sleep 'swaylock -f -c 000000'
   Restart=on-failure
   
   [Install]
   WantedBy=sway-session.target
   ```

4. Test newer kernels or newer Sway/wlroots versions if DRM/KMS compatibility improves

## Useful Commands Reference

### RDP Connection
```bash
~/bin/xxx-rdp
# Or manually:
wlfreerdp3 /v:xxxxxxxxx.xxxxxxxxx.xxx.ac.uk /gateway:g:xxxxxxxxx.xxxxxxxxx.xxx.ac.uk,u:USERNAME,d:DOMAIN ...
```

### Display Management
```bash
# List outputs
swaymsg -t get_outputs

# Manual display configuration tool
wdisplays

# Reload Kanshi profiles
killall -SIGUSR1 kanshi
```

### Debugging
```bash
# Run Sway with debug output
sway -d 2>&1 | tee ~/sway-debug.log

# Check Sway config validity
sway -C ~/.config/sway/config

# Monitor portal activity
journalctl --user -f | grep portal

# Check for Sway crashes
journalctl -b | grep -E "segfault|crash|killed"
```

## Conclusion

The migration from X11/i3 to Wayland/Sway was successful despite several significant challenges. All critical workflows are now functional:

- RDP access through wlfreerdp3 provides reliable gateway connections with full clipboard support
- Teams screen sharing works seamlessly via browser with native Wayland portals
- Multi-monitor docking is fully automated through Kanshi with stable profile matching
- Power management works correctly with appropriate workarounds for driver limitations

The main compromise is the inability to power off external monitors when idle due to Intel i915 DMA-BUF import failures. 
This is a known driver/compositor interaction issue that may be resolved with kernel parameters, driver updates, or newer wlroots versions. 
The current workaround (powering off only the laptop screen) is acceptable for daily use.

