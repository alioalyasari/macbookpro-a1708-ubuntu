# MacBook Pro A1708 (MacBookPro14,1) — Ubuntu 26.04 Complete Setup Guide

> **Hardware:** MacBook Pro 13" 2017 No Touch Bar (A1708 / MacBookPro14,1)
> **OS:** Ubuntu 26.04
> **Kernel:** 7.0.0-15-generic

---

## The Problem

Out of the box on Ubuntu, the MacBook Pro A1708:
- This is technically AI slop but if this could help one person avoid hours of the troubleshooting me and Claude did (LOL), then it's worth it
- Appears to suspend when closing the lid but stays fully awake (running hot, draining battery)
- Hard freezes when actually suspending — requires holding power button to reboot
- WiFi, keyboard and trackpad don't recover after resume
- Audio shows as "Dummy Output" — no sound
- FaceTime HD webcam not working

---

## What Does NOT Apply to This Model

The A1708 **does not have a T2 chip**. Many guides online recommend the t2linux kernel and `apple-bce` module — these do **not** apply to this machine. Skip those entirely.

---

## Part 1 — Fixing Suspend/Resume

### Step 1 — Kernel Parameters

```bash
sudo nano /etc/default/grub
```

Change `GRUB_CMDLINE_LINUX_DEFAULT` to:

```
GRUB_CMDLINE_LINUX_DEFAULT="quiet splash intel_iommu=off acpi_sleep=nonvs button.lid_init_state=open nvme_core.default_ps_max_latency_us=0 nvme.noacpi=1 pci=noaer i915.enable_dc=0 i915.enable_fbc=0 i915.enable_psr=0"
```

Apply changes:
```bash
sudo update-grub
```

**Why these parameters:**
- `intel_iommu=off` — fixes freeze on idle/suspend (common Ubuntu 24.04+ bug)
- `acpi_sleep=nonvs` — prevents ACPI NVS save/restore issues
- `button.lid_init_state=open` — prevents lid jitter causing instant wake
- `nvme_core.default_ps_max_latency_us=0` and `nvme.noacpi=1` — stops NVMe entering a deep power state it can't wake from
- `pci=noaer` — suppresses PCIe error reporting that can cause hangs
- `i915.enable_dc=0 i915.enable_fbc=0 i915.enable_psr=0` — disables Intel display power saving that causes black screen on wake

---

### Step 2 — Disable Thunderbolt

Thunderbolt causes 2-3 minute delays during resume. If you don't use Thunderbolt devices (regular USB-C still works):

```bash
sudo nano /etc/modprobe.d/disable-thunderbolt.conf
```

Add:
```
blacklist thunderbolt
install thunderbolt /bin/true
```

```bash
sudo update-initramfs -u
```

---

### Step 3 — Fix Sleep Configuration

Ubuntu 26.04 defaults to `suspend-then-hibernate` which tries to hibernate after 1 hour. Without a proper swap partition this causes a hard freeze.

```bash
sudo nano /etc/systemd/sleep.conf
```

Set these values (remove `#` if needed):
```ini
[Sleep]
AllowSuspendThenHibernate=no
SuspendState=mem
```

---

### Step 4 — Create the Suspend Fix Script

This script disables NVMe d3cold and manages wake sources before sleep:

```bash
sudo nano /usr/local/bin/tm-suspend-fix.sh
```

```bash
#!/usr/bin/env bash
find /sys/devices/ -name d3cold_allowed -exec sh -c 'echo 0 > "$1" 2>/dev/null' _ {} \;

for device in LID0 XHC1 ARPT RP01 RP09 RP10; do
    if grep -q "$device.*enabled" /proc/acpi/wakeup; then
        echo "$device" > /proc/acpi/wakeup
    fi
done

if grep -q "SPIT.*disabled" /proc/acpi/wakeup; then
    echo "SPIT" > /proc/acpi/wakeup
fi
```

```bash
sudo chmod +x /usr/local/bin/tm-suspend-fix.sh
```

---

### Step 5 — Create the Systemd Sleep Hook

This hook runs before sleep and after wake, handling module unload/reload for WiFi, keyboard and trackpad:

```bash
sudo nano /lib/systemd/system-sleep/tm-suspend-fix
```

```bash
#!/usr/bin/env bash
case $1 in
  pre)
    /usr/local/bin/tm-suspend-fix.sh
    modprobe -r applespi
    modprobe -r brcmfmac
    ;;
  post)
    sleep 5
    modprobe brcmfmac
    modprobe applespi
    sleep 3
    nmcli radio wifi off
    sleep 2
    nmcli radio wifi on
    echo "$(date) Resume complete." >> /tmp/suspend.log
    ;;
esac
```

```bash
sudo chmod +x /lib/systemd/system-sleep/tm-suspend-fix
```

**Why unload modules before sleep:**
- `brcmfmac` — WiFi driver hangs during suspend causing hard freeze
- `applespi` — SPI keyboard/trackpad driver needs clean reload on resume

---

### Step 6 — Reboot and Test

```bash
sudo reboot
```

After reboot:
- Suspend via system menu or close the lid
- Press power button once to wake
- WiFi, keyboard and trackpad should all recover automatically

---

## Part 2 — FaceTime HD Webcam

The webcam uses a Broadcom PCIe chip with no upstream Linux driver. The community-maintained `facetimehd` driver works on this model.

### Step 1 — Install Prerequisites

```bash
sudo apt install -y gcc-12 xz-utils curl cpio make dwarves linux-headers-generic git kmod libssl-dev checkinstall debhelper dkms mplayer unrar
```

### Step 2 — Clone the Driver

```bash
cd ~
git clone https://github.com/patjak/facetimehd.git
cd ~/facetimehd/firmware
git clone https://github.com/patjak/facetimehd-firmware.git facetimehd-firmware
```

### Step 3 — Install Firmware from macOS Drivers

Download the BootCamp package from https://support.apple.com/kb/DL1837 (must use a browser, curl won't work).

Extract and get firmware:

```bash
cd ~/Downloads
unzip bootcamp5.1.5769.zip -d ~/Downloads/bootcamp5.1.5769/
cd ~/Downloads/bootcamp5.1.5769/BootCamp/Drivers/Apple/
unrar x AppleCamera64.exe

dd bs=1 skip=1663920 count=33060 if=AppleCamera.sys of=9112_01XX.dat
dd bs=1 skip=1644880 count=19040 if=AppleCamera.sys of=1771_01XX.dat
dd bs=1 skip=1606800 count=19040 if=AppleCamera.sys of=1871_01XX.dat
dd bs=1 skip=1625840 count=19040 if=AppleCamera.sys of=1874_01XX.dat

sudo mkdir -p /lib/firmware/facetimehd
sudo cp *.dat /lib/firmware/facetimehd/
```

### Step 4 — Install Firmware Package

```bash
cd ~/facetimehd/firmware/facetimehd-firmware
sudo make
sudo make install
```

### Step 5 — Build and Install Driver

```bash
cd ~/facetimehd
sudo make
sudo make install
sudo cp /sys/kernel/btf/vmlinux /usr/lib/modules/$(uname -r)/build/ 2>/dev/null || true
sudo depmod
sudo modprobe facetimehd
```

### Step 6 — Test

```bash
mplayer -vo gl tv://
```

---

## Part 3 — Touchpad Scroll Speed

GNOME has no built-in scroll speed setting. Use libinput config:

```bash
sudo nano /etc/X11/xorg.conf.d/30-touchpad.conf
```

```
Section "InputClass"
    Identifier "touchpad"
    Driver "libinput"
    MatchIsTouchpad "on"
    Option "ScrollPixelDistance" "30"
EndSection
```

Higher number = slower scroll. Default is ~15. Reboot to apply.

---

## Part 4 — Audio Fix

By default audio may show as "Dummy Output". The `snd_hda_macbookpro` driver by davidjo fixes this.

### Step 1 — Install Prerequisites

```bash
sudo apt install gcc linux-headers-generic make patch wget
```

### Step 2 — Clone and Install

```bash
cd ~
git clone https://github.com/davidjo/snd_hda_macbookpro.git
cd snd_hda_macbookpro/
sudo ./install.cirrus.driver.sh
```

### Step 3 — Reboot

```bash
sudo reboot
```

After rebooting go to **Settings → Sound** and set the output to **Analogue Stereo Output** for speakers, or **Analogue Stereo Duplex** if you want to use the internal microphone as well.

**Notes:**
- Internal microphone works but recorded volume is very low — use something like PulseEffects to amplify
- Headphone detection may not work perfectly
- Audio format is limited to 44.1 kHz

---

## What Doesn't Work

- **FaceTime HD Camera** — works via community driver but image quality is limited
- **Hibernate** — not configured (requires swap partition setup)
- **macOS-style suspend-then-poweroff** — Linux stays in suspend indefinitely

---

## Credits

- [t2linux wiki](https://wiki.t2linux.org) — general Linux on Mac reference
- [Dunedan/mbp-2016-linux](https://github.com/Dunedan/mbp-2016-linux/issues/207) — suspend fix approach
- [patjak/facetimehd](https://github.com/patjak/facetimehd) — webcam driver
