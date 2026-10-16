# Sound Blaster AE-5 Plus — Analog 5.1 Surround on Linux

Working setup notes for **Creative Sound BlasterX AE-5 Plus → Logitech Z906 (analog 6-channel direct input)**.
Verified working on **CachyOS** (kernel 7.2.10-1-cachyos, PipeWire 1.6.9, WirePlumber 0.5.18).

---

## TL;DR — the two things that fix it

1. **The card's DSP is stuck in stereo.** The ALSA mixer control `Surround Channel Config` defaults to `2.0`.
   PipeWire sends 6 channels, but the card only outputs front L/R, so rear, center and sub stay silent.
   **Fix:** set it to `5.1`, then save it with `alsactl store`.
2. **The rear and center/LFE channels are swapped** compared with the order ALSA expects.
   **Fix:** a WirePlumber rule that remaps the channel order to `FL, FR, FC, LFE, RL, RR`.

That's the whole fix. No kernel parameters, no modprobe options, no custom drivers.
This is almost certainly why it failed on **Bazzite** and **OpenMandriva**: by default every distro leaves the DSP in `2.0` mode.

---

## 1. Hardware

| Item | Value |
|---|---|
| Card | Creative Labs CA0132 Sound Core3D — CA0113 [Sound BlasterX AE-5 Plus] |
| PCI ID | `1102:0012`, subsystem `1102:0191` |
| Kernel driver | `snd_hda_intel` + `snd_hda_codec_ca0132` (in-tree, nothing to install) |
| ALSA card name | `HDA Creative` (card index was `1` on CachyOS; it can differ) |
| PCI address (CachyOS) | `0f:00.0` → PipeWire names include `pci-0000_0f_00.0` |
| Speakers | Logitech Z906 on its **6-channel direct** input (not the stereo input) |

### Cable colours (AE-5 Plus → Z906)

| Card jack | Z906 6-channel input | Channels |
|---|---|---|
| Green | Front (green) | FL / FR |
| Black | Rear (black) | RL / RR |
| Orange | Center/Sub (orange) | FC / LFE |

On the Z906 console, select the **6-channel direct** input. The stereo inputs mix everything down to 2.0, so there's no real surround.

---

## 2. Packages needed

### Arch / CachyOS (what was installed and working)

```
pipewire          1:1.6.9
pipewire-pulse    1:1.6.9
pipewire-alsa     1:1.6.9
wireplumber       0.5.18
alsa-utils        1.2.16     # amixer, alsactl, speaker-test (REQUIRED)
alsa-lib          1.2.16.1
alsa-ucm-conf     1.2.16.1
alsa-firmware     1.2.4      # provides ctefx-desktop.bin, ctefx-r3di.bin (CA0132 DSP firmware)
linux-firmware    20260916   # linux-firmware-other provides ctefx.bin, ctspeq.bin
pavucontrol       6.2        # optional GUI for choosing the profile
```

Install everything in one go:

```bash
sudo pacman -S --needed pipewire pipewire-pulse pipewire-alsa wireplumber \
    alsa-utils alsa-ucm-conf alsa-firmware linux-firmware pavucontrol
```

### Firmware files that must exist

The CA0132 DSP needs these firmware files. Without them, the card falls back to a crippled mode with no DSP and no surround switching:

```
/usr/lib/firmware/ctefx.bin(.zst)          # linux-firmware(-other)
/usr/lib/firmware/ctspeq.bin(.zst)         # linux-firmware(-other)
/usr/lib/firmware/ctefx-desktop.bin        # alsa-firmware   <-- desktop cards like AE-5
/usr/lib/firmware/ctefx-r3di.bin           # alsa-firmware
```

Check: `ls /usr/lib/firmware/ | grep -i ct`

### Equivalent packages on other distros

| Distro | Packages |
|---|---|
| Fedora / **Bazzite** (atomic) | `pipewire`, `pipewire-pulseaudio`, `pipewire-alsa`, `wireplumber`, `alsa-utils`, `alsa-firmware`, `linux-firmware`. On Bazzite, `alsa-utils` may be missing; install it with `rpm-ostree install alsa-utils alsa-firmware` and reboot. |
| **OpenMandriva** | `pipewire`, `pipewire-pulse`, `pipewire-alsa`, `wireplumber`, `alsa-utils`, `alsa-firmware`, `linux-firmware` (`dnf install ...`) |
| Debian / Ubuntu | `pipewire`, `pipewire-pulse`, `pipewire-alsa`, `wireplumber`, `alsa-utils`, `firmware-misc-nonfree` (Debian) / `linux-firmware` (Ubuntu) |

---

## 3. Step-by-step setup

### Step 1 — Confirm the card and driver

```bash
lspci -nnk | grep -iA3 creative        # driver in use: snd_hda_intel
cat /proc/asound/cards                 # look for "HDA Creative"
lsmod | grep ca0132                    # snd_hda_codec_ca0132 must be loaded
journalctl -k -b | grep -i ca0132      # should say "autoconfig for Sound BlasterX AE-5"
```

### Step 2 — Switch the card's DSP from 2.0 to 5.1 (**the main fix**)

```bash
# List the options (0=2.0 1=2.1 2=4.0 3=4.1 4=5.1)
amixer -c Creative cget name='Surround Channel Config'

# Set to 5.1
amixer -c Creative cset name='Surround Channel Config' 4
# (or by name: amixer -c Creative cset name='Surround Channel Config' '5.1')
```

Also check that these are set (they were like this on the working system):

```bash
amixer -c Creative cget name='Output Select'                          # 0 = Speakers (not Headphone)
amixer -c Creative cget name='HP/Speaker Auto Detect Playback Switch' # off
amixer -c Creative cget name='Full-Range Front Speakers'              # on
amixer -c Creative cget name='Full-Range Rear Speakers'               # on
amixer -c Creative cget name='Bass Redirection'                       # off
```

If `Output Select` is on `Headphone`, set it to Speakers:
`amixer -c Creative cset name='Output Select' 0`

### Step 3 — Make the mixer setting survive reboot

```bash
sudo alsactl store
```

This writes `/var/lib/alsa/asound.state`. The udev rule `/usr/lib/udev/rules.d/90-alsa-restore.rules` (from `alsa-utils`) restores it each time the card is detected at boot.

Check that it was saved:

```bash
grep -A8 "Surround Channel Config" /var/lib/alsa/asound.state | grep value
#   value '5.1'
```

> If it does **not** survive a reboot on some distro, use the fallback systemd service in section 5.3.

### Step 4 — Pick the Analog Surround 5.1 profile in PipeWire

GUI: **pavucontrol → Configuration tab → Sound BlasterX AE-5 → "Analog Surround 5.1 Output + Analog Stereo Input"**.
Or in KDE System Settings → Sound → choose the profile on the card.

CLI:

```bash
pactl list cards short                                     # find the card name
pactl set-card-profile alsa_card.pci-0000_0f_00.0 output:analog-surround-51+input:analog-stereo
pactl set-default-sink alsa_output.pci-0000_0f_00.0.analog-surround-51
```

WirePlumber remembers the profile and default sink on its own after that.

> ⚠ The `pci-0000_0f_00.0` part is the card's PCI address on **this** motherboard and slot. On another PC or slot it will be different. Get yours with `pactl list cards short` / `pactl list sinks short`.

### Step 5 — Test every speaker

```bash
speaker-test -D pipewire -c 6 -t wav -l 1
```

A voice names each channel in turn: *Front Left, Front Center, Front Right, Rear Right, Rear Left, LFE*.
Check each name comes from the matching speaker.

On CachyOS, the result **before** the remap was:

- "Front Center" → came from **Rear Left**
- "Rear Left" → came from **Front Center**
- "LFE" → came from **Rear Right**

That's the rear pair and the center/LFE pair being swapped. Step 6 fixes it.

### Step 6 — Fix the channel order with a WirePlumber rule

Default PipeWire order for this sink: `[ FL, FR, RL, RR, FC, LFE ]`
Correct order for the AE-5 Plus: `[ FL, FR, FC, LFE, RL, RR ]`

Create **`~/.config/wireplumber/wireplumber.conf.d/51-soundblaster-ae5-surround.conf`**:

```
# Sound Blaster AE-5 Plus: rear and center/LFE outputs are swapped
# relative to the default ALSA 5.1 order, so remap the channels.
monitor.alsa.rules = [
  {
    matches = [
      { node.name = "alsa_output.pci-0000_0f_00.0.analog-surround-51" }
    ]
    actions = {
      update-props = {
        audio.position = [ FL, FR, FC, LFE, RL, RR ]
      }
    }
  }
]
```

**Portable version.** It doesn't depend on the PCI address, so use it on a new distro or PC:

```
monitor.alsa.rules = [
  {
    matches = [
      {
        node.name      = "~alsa_output.*analog-surround-51"
        alsa.card_name = "HDA Creative"
      }
    ]
    actions = {
      update-props = {
        audio.position = [ FL, FR, FC, LFE, RL, RR ]
      }
    }
  }
]
```

(All properties inside one `{ }` must match together. `~` means regex.)

Apply and verify:

```bash
mkdir -p ~/.config/wireplumber/wireplumber.conf.d
# (create the file above)
systemctl --user restart wireplumber
sleep 3
pw-cli info alsa_output.pci-0000_0f_00.0.analog-surround-51 | grep audio.position
#   audio.position = "[ FL, FR, FC, LFE, RL, RR ]"
speaker-test -D pipewire -c 6 -t wav -l 1
```

This file is in your home folder, so it loads every login. It only applies to the analog 5.1 sink, so digital (optical/AC3) output isn't touched.

> **Alternative to the remap:** physically swap the **black** and **orange** cables at the card end.
> If you do that, **delete the remap config**, or the channels get swapped back the wrong way.

### Step 7 — Reboot and test again

```bash
amixer -c Creative cget name='Surround Channel Config' | tail -1   # values=4
pactl get-default-sink                                              # ...analog-surround-51
speaker-test -D pipewire -c 6 -t wav -l 1
```

---

## 4. Quick "do it all" script

Run as your normal user. It asks for sudo only for `alsactl store`.

```bash
#!/usr/bin/env bash
set -e
CARD=Creative

# 1. DSP to 5.1, speakers output
amixer -c $CARD cset name='Surround Channel Config' 4
amixer -c $CARD cset name='Output Select' 0 || true
sudo alsactl store

# 2. WirePlumber channel remap (portable match)
mkdir -p ~/.config/wireplumber/wireplumber.conf.d
cat > ~/.config/wireplumber/wireplumber.conf.d/51-soundblaster-ae5-surround.conf <<'EOF'
monitor.alsa.rules = [
  {
    matches = [
      {
        node.name      = "~alsa_output.*analog-surround-51"
        alsa.card_name = "HDA Creative"
      }
    ]
    actions = {
      update-props = {
        audio.position = [ FL, FR, FC, LFE, RL, RR ]
      }
    }
  }
]
EOF
systemctl --user restart wireplumber
sleep 3

# 3. Select the 5.1 profile and make it default
CARDNAME=$(pactl list cards | awk '/^[[:space:]]*Name: alsa_card/ {n=$2} /AE-5/ {print n; exit}')
pactl set-card-profile "$CARDNAME" output:analog-surround-51+input:analog-stereo
SINK=$(pactl list sinks short | awk '/analog-surround-51/ {print $2; exit}')
pactl set-default-sink "$SINK"

# 4. Test
speaker-test -D pipewire -c 6 -t wav -l 1
```

---

## 5. Troubleshooting — if something is different

### 5.1 Only front speakers play (rears/center/sub silent)
- `Surround Channel Config` is still `2.0`. Repeat Step 2 and Step 3.
- PipeWire profile is "Analog Stereo", not "Analog Surround 5.1". Repeat Step 4.
- The Z906 is on a stereo input, not 6-channel direct.
- `Output Select` is on `Headphone`. Set it to `0` (Speakers).

### 5.2 Speakers swapped / wrong positions
- Run `speaker-test -D pipewire -c 6 -t wav -l 1` and write down where each name comes from.
- Edit `audio.position` in the WirePlumber file. The list maps PipeWire channels to the card's
  physical outputs **in hardware order** (out0, out1, ... out5). Hardware order on this card:
  `out0/1 = green jack`, `out2/3 = orange jack (FC, LFE)`, `out4/5 = black jack (RL, RR)`.
- If center and sub are swapped with each other (orange jack L/R), use
  `[ FL, FR, LFE, FC, RL, RR ]`.
- If the left and right rears are swapped: `[ FL, FR, FC, LFE, RR, RL ]`.
- After each edit: `systemctl --user restart wireplumber` and test again.

### 5.3 Setting resets to 2.0 after reboot (alsactl restore not working)

Typical on immutable distros (Bazzite) or distros without the alsa-restore rule. Create a system service that forces it at boot:

`/etc/systemd/system/ae5-surround.service`

```ini
[Unit]
Description=Set Sound Blaster AE-5 DSP to 5.1
After=sound.target
Wants=sound.target

[Service]
Type=oneshot
ExecStartPre=/bin/sleep 3
ExecStart=/usr/bin/amixer -c Creative cset name='Surround Channel Config' 4
ExecStart=/usr/bin/amixer -c Creative cset name='Output Select' 0

[Install]
WantedBy=multi-user.target
```

```bash
sudo systemctl daemon-reload
sudo systemctl enable --now ae5-surround.service
```

On Bazzite, `/etc` is writable, so this works fine. If `/usr/bin/amixer` doesn't exist, `alsa-utils` isn't installed; install it first.

Also check whether the restore services exist: `systemctl status alsa-restore.service alsa-state.service`.

### 5.4 `Surround Channel Config` control doesn't exist / very few mixer controls
- The DSP firmware didn't load. Check:
  ```bash
  journalctl -k -b | grep -iE 'ca0132|ctefx|firmware'
  ls /usr/lib/firmware/ | grep -i ct
  ```
- Install `alsa-firmware` + `linux-firmware`, then **reboot** (a cold power-off is best, so the DSP resets).
- Make sure `snd_hda_codec_ca0132` is loaded (`lsmod | grep ca0132`). Very old kernels (< 5.x) have poor AE-5 support; any modern kernel is fine.

### 5.5 No "Analog Surround 5.1" profile in the list
- The distro may be using the **"Pro Audio"** profile or a UCM profile. Check with `pactl list cards | grep -A40 Creative`.
- Make sure `alsa-ucm-conf` is installed and up to date.
- On older setups using **PulseAudio** (not PipeWire): pick the profile in pavucontrol. For the channel remap, add this to `/etc/pulse/default.pa` (or `~/.config/pulse/default.pa`):
  ```
  load-module module-remap-sink sink_name=ae5_51 master=alsa_output.pci-XXXX.analog-surround-51 channels=6 master_channel_map=front-left,front-right,front-center,lfe,rear-left,rear-right channel_map=front-left,front-right,rear-left,rear-right,front-center,lfe remix=no
  set-default-sink ae5_51
  ```
  (Replace `XXXX` with your PCI address from `pactl list sinks short`.)

### 5.6 WirePlumber rule doesn't apply
- Check the WirePlumber version: `wireplumber --version`. The `.conf` syntax here is for **0.5.x**.
  On **0.4.x** (older distros), use a Lua file instead:
  `~/.config/wireplumber/main.lua.d/51-ae5-surround.lua`
  ```lua
  table.insert(alsa_monitor.rules, {
    matches = { { { "node.name", "matches", "alsa_output.*analog-surround-51" } } },
    apply_properties = { ["audio.position"] = "FL,FR,FC,LFE,RL,RR" },
  })
  ```
- The node name doesn't match. Check `pactl list sinks short` and fix the `node.name`, or use the portable regex version.
- Check the journal: `journalctl --user -u wireplumber -b`.

### 5.7 Digital (optical) 5.1 works but analog doesn't
That was exactly the starting point on CachyOS. Digital AC3 goes out of the card's S/PDIF and the Z906 decodes it itself, so the DSP's analog channel config doesn't matter there. Analog needs Steps 2–6.

---

## 6. Optional — card DSP effects

On the working system: `Enable OutFX Playback Switch = off`, so effects aren't applied, even though `FX: Crystalizer`, `FX: Smart Volume`, `FX: Surround` and `FX: X-Bass` are individually on.
For clean, unprocessed 5.1, keep OutFX off. To try the effects:

```bash
amixer -c Creative cset name='Enable OutFX Playback Switch' on
amixer -c Creative cset name='FX: Crystalizer Playback Switch' off   # etc.
sudo alsactl store
```

`alsamixer -c Creative` (F6 to pick the card) shows all the controls interactively.

---

## 7. Files touched (summary)

| File | Purpose | Created by |
|---|---|---|
| `/var/lib/alsa/asound.state` | Saved mixer state, incl. `Surround Channel Config = 5.1` | `sudo alsactl store` |
| `~/.config/wireplumber/wireplumber.conf.d/51-soundblaster-ae5-surround.conf` | Channel remap FL,FR,FC,LFE,RL,RR | manually |
| `/usr/lib/udev/rules.d/90-alsa-restore.rules` | Restores asound.state at boot | package `alsa-utils` (not edited) |
| `/etc/systemd/system/ae5-surround.service` | *Fallback only* (section 5.3) | not used on CachyOS |

Nothing was changed in `/etc/modprobe.d`, the kernel command line, or PipeWire's own config.
