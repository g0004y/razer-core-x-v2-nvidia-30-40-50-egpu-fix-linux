# 🖥️ eGPU on Linux — ThinkPad T14 Gen 3 + Razer Core X V2 + RTX 4070 SUPER

> A real-world guide to getting an NVIDIA RTX 40-series eGPU working on Linux via USB4/Thunderbolt.
> Documented from actual troubleshooting — every detail here was verified against a working system.

---

## Background

I am not a Linux expert. I already owned the ThinkPad T14 Gen 3 and spent nearly **6 months**
unable to use the eGPU on Linux after buying the RTX 4070 SUPER and Razer Core X V2. I tried
countless kernel versions, driver versions, kernel parameters, and firmware tweaks. Nothing worked
reliably. This guide is everything I learned — the hard way.

After six months of trial and error, the combination documented below finally works:

- Cold boot with eGPU connected
- Reboot with eGPU connected
- Hotplugging after full boot
- Suspend / resume
- PRIME render offload
- External monitor via eGPU (GNOME Wayland)

---

## Hardware

| Component | Details |
|---|---|
| Laptop | Lenovo ThinkPad T14 Gen 3 (Intel) |
| iGPU | Intel Iris Xe Graphics (ADL GT2) |
| eGPU Enclosure | Razer Core X V2 |
| eGPU Card | NVIDIA GeForce RTX 4070 SUPER (ZOTAC) |
| Connection | USB4 @ 40 Gb/s (2 lanes × 20 Gb/s) |
| Kernel | `6.17.0-19-generic` |
| OS | Linux Mint 22.3 Zena (Ubuntu Noble base) |
| Desktop session | GNOME on Wayland |
| NVIDIA Driver | `580.126.09` (proprietary) |
| CUDA | 13.0 |

> **On the connection:** `boltctl list` reports the Razer Core X V2 as USB4 at 40 Gb/s — not TB4 at 32 Gbps as often stated. The enclosure negotiates to the best speed the laptop supports.

---

## The Problem

After connecting the eGPU and booting Linux, the GPU would randomly fail to initialize — sometimes working, sometimes completely invisible. When it failed, `nvidia-smi` returned `No devices were found` and the kernel logged:

```
NVRM: Xid (PCI:0000:52:00): 79, GPU has fallen off the bus.
NVRM: GPU 0000:52:00.0: GPU has fallen off the bus.
NVRM: _kgspLogRpcSanityCheckFailure: GPU0 sanity check failed 0xf
         waiting for RPC response from GSP.
```

The same GPU worked **perfectly on Windows** — confirming this is a Linux driver/firmware issue, not hardware or the enclosure.

---

## Why It Happens

### 1. The Thunderbolt/USB4 Link Drops During Boot

The USB4 controller on the T14 Gen 3 does its own internal link reset sequence during early boot. The PCIe slot goes **down at ~2.5 seconds** and comes back up at **~22 seconds**. The NVIDIA driver loads at ~2.3 seconds — right into that unstable window.

The exact sequence from the kernel log:

```
[2.3s]  nvidia driver loads, begins GSP firmware init
[2.5s]  pcieport: Slot(5): Link Down  ← USB4 drops mid-init
[2.5s]  pcieport: Slot(5): Card not present
[2.6s]  NVRM: Xid 79, GPU has fallen off the bus  ← driver fails
[22.5s] pcieport: Slot(5): Card present, Link Up  ← USB4 recovers
[22.6s] nvidia: enabling device  ← driver retries but never recovers
```

### 2. GSP Firmware Architecture (RTX 40-series specific)

RTX 40-series (Ada Lovelace) introduced **GSP — GPU System Processor** firmware. When the driver loads it must push a firmware blob onto the GPU and wait for it to boot before anything else can happen. This RPC handshake requires uninterrupted, stable PCIe communication for several seconds. Over a USB4 tunnel that drops during boot, GSP never completes.

Older NVIDIA cards (GTX 10xx, RTX 20xx) and **all AMD cards** do not require GSP and initialize fine in eGPUs on Linux.

### 3. Kernel and Driver Version Matter

Kernel 6.8 with any driver version was not sufficient for this setup. Kernel 6.17 combined with driver 580 has better USB4/Thunderbolt PCIe enumeration handling. The `NVreg_EnableGpuFirmware=0` parameter on 580 must be set via `/etc/modprobe.d/` — setting it as a GRUB kernel parameter is silently ignored.

---

## Why This Works

The root cause is not USB4 instability alone — it is **GSP firmware initialization**, which is stateful and cannot survive any link interruption.

- USB4 tunnels PCIe. The link resets during early boot.
- With GSP enabled, one interrupted init sequence causes permanent failure (`Xid 79`). The driver does not recover.
- Disabling GSP (`NVreg_EnableGpuFirmware=0`) makes the driver stateless enough to attach whenever PCIe appears — at boot or via hotplug.
- Kernel 6.17 improves USB4 PCIe enumeration handling over 6.8.

This means:
- Cold boot works reliably
- Reboot works reliably
- Hotplug works instantly (GPU detected immediately after plugging in)
- Suspend/resume works (VRAM preserved)

Verified via `nvidia-smi -q`:

```
    GSP Firmware Version                               : N/A
```

GSP is definitively disabled. No errors in dmesg on hotplug, boot, or resume.

---

## The Fix

### Step 1 — Authorize the Thunderbolt/USB4 Device

Linux requires Thunderbolt/USB4 devices to be authorized before use. Check the enclosure is seen:

```bash
boltctl list
```

Expected output for a working authorized device:

```
 ● Razer Core X V2
   ├─ type:       peripheral
   ├─ uuid:       8ab48780-0045-39a9-ffff-ffffffffffff
   ├─ generation: USB4
   ├─ status:     authorized
   │  ├─ rx speed: 40 Gb/s = 2 lanes * 20 Gb/s
   │  └─ tx speed: 40 Gb/s = 2 lanes * 20 Gb/s
   └─ stored:
      └─ policy:  iommu
```

If not yet stored, enroll it:

```bash
sudo boltctl enroll --policy iommu <UUID>
```

Replace `<UUID>` with the uuid shown in `boltctl list`.

Verify the GPU is visible on PCIe:

```bash
lspci | grep -i nvidia
```

Expected:

```
52:00.0 VGA compatible controller: NVIDIA Corporation AD104 [GeForce RTX 4070 SUPER] (rev a1)
52:00.1 Audio device: NVIDIA Corporation AD104 High Definition Audio Controller (rev a1)
```

---

### Step 2 — Install the Correct Kernel

The generic 6.8 kernel does not work reliably for this setup. You need **kernel 6.17 from Linux Mint's kernel manager**.

Open Linux Mint's Update Manager → View → Linux Kernels → install `6.17.0-19-generic`.

> ⚠️ Do **NOT** install `linux-nvidia-hwe-24.04` via apt — it is an Ubuntu-only package that breaks Linux Mint's initramfs and drops you to an emergency shell on boot. Only use the kernel from Mint's own kernel manager.

After installing, verify the initrd was built:

```bash
ls /boot/initrd.img*
```

You should see `initrd.img-6.17.0-19-generic`. Reboot and select it from GRUB → Advanced options.

---

### Step 3 — Install the NVIDIA Driver (580)

```bash
sudo apt install nvidia-driver-580
sudo update-initramfs -u -k all
sudo reboot
```

> Kernel 6.17 + proprietary driver 580 is the confirmed working combination on this hardware.

---

### Step 4 — Configure modprobe Options

Create or edit `/etc/modprobe.d/nvidia-gsp.conf`:

```bash
sudo nano /etc/modprobe.d/nvidia-gsp.conf
```

Contents should be exactly:

```
options nvidia NVreg_EnableGpuFirmware=0
options nvidia NVreg_DynamicPowerManagement=0x00
options nvidia NVreg_PreserveVideoMemoryAllocations=1
```

| Option | What it does |
|---|---|
| `NVreg_EnableGpuFirmware=0` | Disables GSP firmware — the core fix for the USB4 init crash |
| `NVreg_DynamicPowerManagement=0x00` | Disables GPU dynamic power management — prevents power-gating over USB4 |
| `NVreg_PreserveVideoMemoryAllocations=1` | Preserves VRAM across suspend/resume — prevents corruption on wake |

Rebuild the initramfs so these are included at boot:

```bash
sudo update-initramfs -u -k all
sudo reboot
```

Boot into `6.17.0-19-generic`.

---

### Step 5 — Verify

```bash
nvidia-smi
```

Expected output:

```
+-----------------------------------------------------------------------------------------+
| NVIDIA-SMI 580.126.09   Driver Version: 580.126.09     CUDA Version: 13.0              |
|   0  NVIDIA GeForce RTX 4070 SUPER   Off | 0000:52:00.0 Off |                  N/A |
|  0%   56C    P0             25W / 220W |       8MiB / 12282MiB |      0%      Default |
+-----------------------------------------------------------------------------------------+
```

Check for a clean boot log with no GPU errors:

```bash
journalctl -k -b | grep -i "nvrm\|gsp\|xid"
```

A working boot produces only one line:

```
kernel: NVRM: loading NVIDIA UNIX x86_64 Kernel Module  580.126.09  ...
```

No Xid 79, no GSP crash, no FWSEC errors — you're done.

Verify GSP is disabled:

```bash
nvidia-smi -q | grep -i gsp
```

Expected:

```
    GSP Firmware Version                               : N/A
```

---

## Running Apps on the eGPU (PRIME Offload)

The laptop's internal screen is wired to Intel Iris Xe, not the RTX 4070 SUPER. To use the eGPU for rendering while keeping the laptop screen alive, use **PRIME render offload**.

### Native apps

```bash
__NV_PRIME_RENDER_OFFLOAD=1 __GLX_VENDOR_LIBRARY_NAME=nvidia <your-app>
```

### Convenience wrapper (recommended)

Typing the env vars every time gets tedious. Create a reusable wrapper:

```bash
sudo nano /usr/local/bin/nv-run
```

Contents:

```bash
#!/bin/bash
__NV_PRIME_RENDER_OFFLOAD=1 \
__GLX_VENDOR_LIBRARY_NAME=nvidia \
"$@"
```

```bash
sudo chmod +x /usr/local/bin/nv-run
```

Then run any app through the eGPU:

```bash
nv-run steam
nv-run blender
nv-run <any-app>
```

### Flatpak apps (e.g. Bottles)

Set permanently so you don't retype it every time:

```bash
flatpak override --user \
  --env=__NV_PRIME_RENDER_OFFLOAD=1 \
  --env=__GLX_VENDOR_LIBRARY_NAME=nvidia \
  --env=__VK_ICD_FILENAMES=/var/lib/flatpak/runtime/org.freedesktop.Platform.GL.nvidia-580-126-09/x86_64/share/vulkan/icd.d/nvidia_icd.json \
  com.usebottles.bottles
```

### Required Flatpak NVIDIA runtimes

Both 32-bit and 64-bit runtimes are needed for full compatibility:

```bash
flatpak install flathub org.freedesktop.Platform.GL.nvidia-580-126-09
flatpak install flathub org.freedesktop.Platform.GL32.nvidia-580-126-09
```

> Version must match your driver exactly. For driver `580.126.09` the runtime name is `nvidia-580-126-09`.

### Verify the eGPU is being used

```bash
watch -n 1 nvidia-smi
```

GPU utilization should spike when running an app with PRIME offload active.

---

## Hotplug Behavior

Hotplugging works **instantly**. The GPU is detected immediately after connecting the USB4 cable.
`nvidia-smi` becomes available without any service restarts or manual intervention.

Clean hotplug dmesg output:

```
nvidia 0000:52:00.0: enabling device
NVRM: loading NVIDIA UNIX x86_64 Kernel Module
[drm] Initialized nvidia-drm
```

No Xid errors, no GSP failures. The GPU initializes as soon as PCIe enumerates it.

---

## Suspend / Resume

Suspend and resume work reliably with the eGPU connected.

- The GPU remains available after wake
- `nvidia-smi` continues to function normally
- No driver reload or manual intervention required

This works because:
- GSP firmware is disabled (no fragile state to restore)
- VRAM allocations are preserved (`NVreg_PreserveVideoMemoryAllocations=1`)
- Dynamic power management is disabled (`NVreg_DynamicPowerManagement=0x00`)

---

## Display Modes — What Works and What Doesn't

| Setup | GNOME Wayland | Cinnamon Wayland |
|---|---|---|
| Internal laptop screen | ✅ Works | ⚠️ Works but occasional issues |
| External monitor via eGPU | ✅ Works perfectly | ❌ Only cursor shows, desktop blank |
| PRIME render offload | ✅ Works | ⚠️ Unreliable |
| Screen wake after idle | ✅ Works | ❌ Black screen, requires Ctrl+Alt+F1 |

**This setup uses GNOME Wayland and it works perfectly.** Internal screen via PRIME offload, external
monitor connected directly to the eGPU — both work reliably.

### Why GNOME Wayland works here

On this setup the render path is:

```
RTX 4070 SUPER (eGPU via USB4) → renders frame → hands to Intel Iris Xe → Intel drives internal screen
```

This buffer handoff is **PRIME buffer sharing**, and it requires the compositor to handle it correctly.

- **GNOME's Wayland compositor (Mutter)** has explicit sync fixes for NVIDIA and handles PRIME buffer sharing reliably in this configuration.
- **Cinnamon's Wayland compositor (Muffin)** is still experimental and hasn't implemented proper PRIME sync or display wake handling for NVIDIA eGPU setups — causing cursor-only external monitor issues and black screens on idle.

### How GNOME Wayland renders on NVIDIA (technical details)

GNOME uses **Zink** (Mesa Vulkan) as its compositor buffer path for NVIDIA GPUs. This bypasses NVIDIA's limited DRM display handling on Wayland and routes rendering through Vulkan, which the NVIDIA driver handles natively.

Verified output from `glxinfo -B`:

```
Vendor: Mesa (0x10de)
Device: zink Vulkan 1.4(NVIDIA GeForce RTX 4070 SUPER (NVIDIA_PROPRIETARY))
Accelerated: yes
Video memory: 12528MB
```

- `nvidia-drm` loads and initializes correctly (`dmesg` shows `Initialized nvidia-drm for 52:00.0`)
- `Cannot find any crtc or sizes` in dmesg is expected — the eGPU has no display attached, so DRM cannot assign display controllers. This is not an error.
- No errors in kernel log (`journalctl -k -b | grep -iE "nvidia.*error|xid|gsp.*fail"`)

Verify your setup:

```bash
# Confirm Wayland session
echo $XDG_SESSION_TYPE

# Check rendering goes to NVIDIA
glxinfo -B | grep "Device:"

# Check for errors
journalctl -k -b | grep -iE "nvidia.*error|xid|gsp.*fail|nvidia-drm.*fail"
```

At the login screen, select **"GNOME"** (Wayland) — it works without issues on this hardware.

---

## What Was Tried and Failed (Don't Repeat These)

This took months of troubleshooting. These approaches all failed:

| Approach | Why it failed |
|---|---|
| Driver 535 proprietary + `NVreg_EnableGpuFirmware=0` as GRUB kernel param | Ignored by 580 driver as kernel param; on 535 still crashed due to USB4 timing |
| `nvidia-driver-535-open` | Same GSP crash, different error messages |
| `softdep nvidia pre: thunderbolt` in modprobe | Didn't delay loading enough — USB4 link still dropped mid-init |
| `pci=assign-busses,hpbussize=0x33,realloc,...` | No effect on the GSP timing failure |
| `NVreg_DeferredPCIePMEnable=1` | Not recognized by 580 driver — silently ignored |
| `linux-nvidia-hwe-24.04` via apt | Ubuntu-only package — breaks Linux Mint initramfs, emergency shell on boot |
| Kernel 6.8 + any driver | USB4 PCIe handling insufficient on 6.8 for this combination |
| `NVreg_EnableGpuFirmware=0` as GRUB kernel param | Silently ignored on driver 580 — must go in `/etc/modprobe.d/` |
| `NVreg_EnableResizableBar=1` | BAR allocations were already fine — not the issue |

**The exact working combination:** kernel `6.17.0-19-generic` (from Mint kernel manager) + `nvidia-driver-580` + the three options in `/etc/modprobe.d/nvidia-gsp.conf` + `sudo update-initramfs -u -k all`.

---

## Bluetooth (Intel AX211 CNVi)

The AX211 Bluetooth adapter has two separate issues.

---

### Issue 1 — Corsair Device Pairing

The default Mint kernel firmware for the AX211 fails to pair with certain devices, notably Corsair peripherals. This is resolved by replacing the firmware files with those from Parrot OS (version ~60).

> **Full documentation, firmware files, and installation instructions are available in my
> [separate Bluetooth GitHub repository](https://github.com/vik-the-guy).**

---

### Issue 2 — Bluetooth Crashes Under eGPU Load (Hardware Conflict)

> ⚠️ **This is a hardware-level conflict that cannot be resolved in software.**

#### What happens

After the Corsair firmware fix, the AX211 Bluetooth adapter still crashes when the Razer Core X V2 eGPU is connected and under sustained load. Kernel log shows:

```
Bluetooth: hci0: command 0x2039 tx timeout
Bluetooth: hci0: Error when powering off device on rfkill (-110)
Bluetooth: hci0: HCI reset during shutdown failed
```

`-110` is `ETIMEDOUT`. The controller crashes (`hci0 removed`) seconds after pairing under load. **Bluetooth works perfectly the moment the USB4 cable is disconnected.**

#### Why it happens

The T14 Gen 3 uses **Intel AX211 CNVi** — Bluetooth is connected via internal USB. When the USB4 controller is carrying eGPU traffic, it consumes enough bandwidth to starve the Bluetooth chip's USB connection. This is a hardware-level bandwidth conflict.

#### What was tried (all failed)

| Fix | Result |
|---|---|
| `btusb.enable_autosuspend=0` kernel param | No effect |
| `usbcore.autosuspend=-1` kernel param | No effect |
| udev rule disabling autosuspend for `8087:0033` | No effect |
| `AutoEnable=true` in `/etc/bluetooth/main.conf` | Adapter powers on but still crashes |
| udev rule restarting bluetooth on TB connect | Partial — recovers briefly then drops again |
| systemd service reinitializing BT at boot | No effect |
| `rfkill unblock`, `systemctl restart bluetooth`, `hciconfig hci0 up` | No effect |

#### The fix — USB Bluetooth dongle

The only reliable solution is a **USB Bluetooth dongle plugged directly into the laptop**.

**Recommended:** TP-Link UB500 (~€8) — works on Linux Mint out of the box, no configuration needed.

```bash
# Verify it's detected
lsusb | grep -i bluetooth

# Restart service to pick it up
sudo systemctl restart bluetooth

# Confirm it works
bluetoothctl show
```

---

## CUDA Verification

```bash
nvcc --version
```

Stress test to confirm stability under full GPU load:

```c
// cuda_stress.cu
#include <cuda_runtime.h>
#include <stdio.h>

__global__ void vecAdd(float *a, float *b, float *c, size_t N) {
    size_t idx = blockIdx.x * blockDim.x + threadIdx.x;
    if (idx < N) c[idx] = a[idx] + b[idx];
}

int main() {
    const size_t N = 1 << 28;
    float *d_a, *d_b, *d_c;
    cudaMalloc(&d_a, N * sizeof(float));
    cudaMalloc(&d_b, N * sizeof(float));
    cudaMalloc(&d_c, N * sizeof(float));
    cudaMemset(d_a, 1, N * sizeof(float));
    cudaMemset(d_b, 2, N * sizeof(float));

    int threadsPerBlock = 1024;
    int blocksPerGrid = (N + threadsPerBlock - 1) / threadsPerBlock;

    for (int i = 0; i < 1000; i++) {
        vecAdd<<<blocksPerGrid, threadsPerBlock>>>(d_a, d_b, d_c, N);
        cudaDeviceSynchronize();
        if (i % 50 == 0) printf("Iteration %d/1000\n", i);
    }

    cudaFree(d_a); cudaFree(d_b); cudaFree(d_c);
    printf("CUDA stress test completed successfully!\n");
    return 0;
}
```

```bash
nvcc cuda_stress.cu -o cuda_stress
./cuda_stress
```

Monitor simultaneously in another terminal:

```bash
watch -n 1 nvidia-smi
```

---

## Stability Matrix

| Scenario | Result |
|---|---|
| Cold boot with eGPU connected | ✅ Works reliably |
| Reboot with eGPU connected | ✅ Works reliably |
| Hotplug after full boot | ✅ Works instantly |
| Suspend / resume | ✅ Works |
| PRIME render offload | ✅ Works |
| External monitor via eGPU (GNOME Wayland) | ✅ Works perfectly |
| Bluetooth with internal AX211 | ❌ Broken under eGPU load |

---

## Quick Reference

```bash
# Check eGPU authorization and connection speed
boltctl list

# Check GPU is visible on PCIe
lspci | grep -i nvidia

# Check driver and GPU status
nvidia-smi

# Verify GSP is disabled
nvidia-smi -q | grep -i gsp

# Check boot log for GPU errors
journalctl -k -b | grep -i "nvrm\|xid\|gsp"

# Check for expected nvidia-drm init (no crtc error is fine)
journalctl -k -b | grep -i "nvidia.*drm"

# Check CUDA version
nvcc --version

# Verify what kernel params are actually active at boot
cat /proc/cmdline

# Check modprobe options are correct
cat /etc/modprobe.d/nvidia-gsp.conf

# Verify which NVreg options your driver supports
modinfo nvidia | grep NVreg

# Check Vulkan sees the eGPU inside Flatpak
flatpak run --command=vulkaninfo com.usebottles.bottles | grep deviceName

# Monitor GPU live
watch -n 1 nvidia-smi

# Launch any native app on the eGPU
__NV_PRIME_RENDER_OFFLOAD=1 __GLX_VENDOR_LIBRARY_NAME=nvidia <app>
```

---

## Known Limitations

- **USB4 vs native PCIe:** The eGPU connects at 40 Gb/s over USB4 vs the card's native capability of ~252 Gb/s over PCIe x16. Performance is real but bottlenecked — most noticeable in GPU-CPU bandwidth-heavy workloads.
- **Bluetooth:** Internal AX211 CNVi Bluetooth conflicts with the USB4 controller when the eGPU is connected. Use a USB Bluetooth dongle (TP-Link UB500) as the permanent fix.
- **Cinnamon Wayland:** External monitor via eGPU only shows cursor — use GNOME Wayland or Xorg instead.
- **Screen blanking:** Screen wake works reliably on GNOME Wayland. On Cinnamon Wayland, if the screen goes black and doesn't wake, press `Ctrl+Alt+F1` then `Ctrl+Alt+F7` to reset the display pipeline.

---

## References

- [Arch Wiki — PRIME](https://wiki.archlinux.org/title/PRIME)
- [Arch Wiki — NVIDIA](https://wiki.archlinux.org/title/NVIDIA)
- [Arch Wiki — Thunderbolt](https://wiki.archlinux.org/title/Thunderbolt)
- [NVIDIA Linux Driver Docs — GSP Firmware](https://download.nvidia.com/XFree86/Linux-x86_64/latest/README/gsp.html)
- [NVIDIA AllowExternalGpus option](https://download.nvidia.com/XFree86/Linux-x86_64/latest/README/xconfigoptions.html)
- [egpu.io community builds](https://egpu.io/forums/builds/)
- [Bluetooth AX211 Firmware Fix (separate repo)](https://github.com/vik-the-guy)

---

*Verified on: kernel `6.17.0-19-generic` · driver `580.126.09` · Linux Mint 22.3 Zena · Razer Core X V2 · RTX 4070 SUPER*

---

## Why This Guide Exists

There was no single place that had everything explained properly. Information was scattered across random
Reddit threads, forum comments, and half-finished GitHub issues. NVIDIA's own documentation is
technically detailed but complex and hard to apply to real-world eGPU setups. Most guides covered one
piece of the puzzle — kernel params, driver versions, Thunderbolt authorization — but never explained
how they fit together, why each step matters, or why certain combinations fail.

This guide puts everything under one roof: root cause, the exact fix, why it works, what was tried
and why it failed, and the technical details behind each decision. If something here helps you or
you find something that can be improved, feel free to reach out.

---

## Disclaimer

This guide documents exactly what worked for my setup. Your hardware combination, BIOS settings, USB4
implementation, and kernel driver versions may differ — results can vary. The core fix (disabling GSP)
is generally applicable to RTX 40-series eGPUs on Linux, but the specific kernel and driver version
requirements are specific to this hardware.

Everything in this guide is explained in detail — root cause, why the fix works, and why other
approaches fail. Read the "Why It Happens" and "Why This Works" sections before applying anything,
so you understand what each step does and how to adapt it if needed.

I take no responsibility if something breaks your system. You follow this guide at your own risk.
