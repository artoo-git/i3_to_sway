# Sway Configuration for Lenovo ThinkPad X270

Wayland/Sway configuration for professional use on Lenovo ThinkPad X270 with docking station, external monitors, RDP connectivity, and Teams screen sharing.

## Hardware

- **Laptop:** Lenovo ThinkPad X270
- **Graphics:** Intel HD Graphics 520 (Skylake GT2, Gen 9)
- **Docking:** 2x Dell P2722H monitors (1920x1080) via DisplayPort
- **OS:** Debian GNU/Linux (forky/sid)
- **Kernel:** 6.17.13+deb14-amd64

## Features

* **Multi-monitor docking** - Automatic display configuration via Kanshi  
* **RDP connectivity** - Gateway-based RDP via wlfreerdp3  
* **Screen sharing** - Browser-based Teams with PipeWire  
* **Clipboard** - Bidirectional clipboard in RDP sessions  
* **Professional workflow** - Status bar (Waybar), launcher (wofi), terminal (foot)

## Installation

Other optional packages are available, This is what I used:

```bash
# Core Sway packages
sudo apt install sway swayidle swaylock kanshi waybar wofi foot

# Portal for screen sharing
sudo apt install xdg-desktop-portal-wlr

# RDP client
sudo apt install freerdp3-wayland

# Screenshot tools
sudo apt install grim slurp

# Display configuration tool (optional)
sudo apt install wdisplays

# Notification daemon
sudo apt install mako-notifier
```

### Configuration Setup

```bash
# Clone this repository
git clone https://github.com/YOUR_USERNAME/sway-config.git
cd sway-config

# Install configs
mkdir -p ~/.config/{sway,kanshi,waybar,wofi,foot,xdg-desktop-portal}

cp config/sway/config ~/.config/sway/
cp config/kanshi/config ~/.config/kanshi/
cp config/waybar/* ~/.config/waybar/
cp config/xdg-desktop-portal/sway-portals.conf ~/.config/xdg-desktop-portal/

# Install RDP script
mkdir -p ~/bin
cp scripts/ucl-rdp ~/bin/
chmod +x ~/bin/ucl-rdp
```

### First Launch

```bash
# From TTY (Ctrl+Alt+F2)
sway

# Or add to ~/.bash_profile for auto-start:
if [ -z "$WAYLAND_DISPLAY" ] && [ "$XDG_VTNR" -eq 1 ]; then
  exec sway
fi
```

### RDP Connection

```bash
# Via script 
~/bin/ucl-rdp

# Or via Remmina (XWayland mode)
GDK_BACKEND=x11 remmina
```

**In fullscreen mode:** Press `Ctrl+Alt+Enter` to exit fullscreen.

### Teams Screen Sharing

1. Open browser (Firefox or Chrome)
2. Navigate to teams.microsoft.com
3. Join meeting
4. Share screen - select window or entire screen
5. PipeWire portal will show picker

### Display Management

**Automatic (recommended):**
- Kanshi detects dock/undock and applies profiles automatically
- Edit `~/.config/kanshi/config` to customize

**Manual:**
```bash
wdisplays
```

**Get monitor info for Kanshi config:**
```bash
swaymsg -t get_outputs
```

## Configuration

### Main Config

`~/.config/sway/config` - Main Sway configuration

Key customizations:
- Mod key: `Super` (Windows key)
- Terminal: `foot`
- Launcher: `wofi`
- Font: `monospace 10`
- Laptop scaling: `1.2`

### Kanshi Display Profiles

`~/.config/kanshi/config` - Automatic display configuration

**Important:** Uses monitor serial numbers for reliable matching. Update with your monitor serials from `swaymsg -t get_outputs`.

Example:
```
profile docked {
    output "Dell Inc. DELL P2722H 9S67DJ3" enable mode 1920x1080@60Hz position 0,0 scale 1
    output "Dell Inc. DELL P2722H 9267DJ3" enable mode 1920x1080@60Hz position 1920,0 scale 1
    output "Chimei Innolux Corporation 0x1239 Unknown" enable mode 1920x1080@60Hz position 939,1080 scale 1.2
}
```

### Waybar

`~/.config/waybar/config` - Status bar configuration  
`~/.config/waybar/style.css` - Status bar styling

Shows: workspaces, window title, network, battery, memory, CPU, date/time

### Portal Configuration

`~/.config/xdg-desktop-portal/sway-portals.conf`

Configures screen sharing backend:
```ini
[preferred]
default=wlr;gtk
org.freedesktop.impl.portal.ScreenCast=wlr
org.freedesktop.impl.portal.Screenshot=wlr
```

**Required setup:**
```bash
systemctl --user mask xdg-desktop-portal-gtk
systemctl --user restart xdg-desktop-portal
```

### Idle Management

Configured in `~/.config/sway/config`:
- **5 minutes idle** → Screen lock
- **10 minutes idle** → Power off laptop screen
- **Lid close** → Suspend system
- **Before suspend** → Lock screen

**Note:** Only laptop screen powers off to avoid i915 driver crash with external monitors.

### RDP Issues

If wlfreerdp3 fails:
- Check network connectivity to gateway
- Verify credentials
- Try XWayland Remmina: `GDK_BACKEND=x11 remmina`

## Other Known Issues

### External Monitor Power-Off Crash

**Issue:** Sway crashes when trying to power off external monitors via `swaymsg "output * power off"` while docked.

**Cause:** Intel i915 DMA-BUF buffer import failures on DisplayPort outputs (driver bug).

**Workaround:** Swayidle only powers off laptop screen (`eDP-*`), leaving external monitors active.

**Potential fix (untested):**
```bash
# Add to /etc/default/grub
GRUB_CMDLINE_LINUX_DEFAULT="quiet i915.enable_psr=0 i915.enable_dc=0"

sudo update-grub
sudo reboot
```

### Remmina Native Wayland

**Issue:** Remmina crashes on RDP gateway connections in native Wayland mode.

**Cause:** FreeRDP bug in versions 1.4.39-1.4.40 with Kerberos/NTLM gateway authentication.

**Workaround:** Use wlfreerdp3 script or launch Remmina with XWayland (`GDK_BACKEND=x11 remmina`).

## File Structure

```
~/.config/
├── sway/
│   └── config                    # Main Sway configuration
├── kanshi/
│   └── config                    # Display profiles
├── waybar/
│   ├── config                    # Status bar config
│   └── style.css                 # Status bar styling
├── wofi/
│   ├── config                    # Launcher config
│   └── style.css                 # Launcher styling
├── foot/
│   └── foot.ini                  # Terminal config
├── xdg-desktop-portal/
│   └── sway-portals.conf         # Portal preferences
└── mako/
    └── config                    # Notification daemon

~/bin/
└── ucl-rdp                       # RDP connection script

~/.local/share/applications/
└── remmina.desktop               # Remmina XWayland launcher
```

## Performance Notes

- **Memory usage:** ~2GB idle (Sway + Waybar + standard apps)
- **CPU usage:** <5% idle
- **Battery life:** ~7-8 hours typical usage (comparable to X11)
- **External monitor support:** Tested with 2x 1080p displays, stable at 60Hz

## Migration from X11/i3

See [X11-to-Sway-Migration.md](X11-to-Sway-Migration.md) for detailed migration guide including:
- All issues encountered during migration
- Step-by-step solutions
- Configuration differences from i3
- Testing procedures

## Credits

- **Sway:** https://swaywm.org/
- **wlroots:** https://gitlab.freedesktop.org/wlroots/wlroots
- **Waybar:** https://github.com/Alexays/Waybar
- **Kanshi:** https://sr.ht/~emersion/kanshi/

## Contact

For general Sway support:
- Official docs: https://github.com/swaywm/sway/wiki
- Reddit: r/swaywm
- IRC: #sway on Libera.Chat

---

**Status:** Production-ready, daily driver since [20260202]

**Last updated:** [20260209]
