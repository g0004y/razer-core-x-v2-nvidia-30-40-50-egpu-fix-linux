# eGPU on Linux — ThinkPad T14 Gen 3 + Razer Core X V2 + RTX 4070 SUPER

> A real-world guide to getting an NVIDIA RTX 40-series eGPU working on Linux via USB4/Thunderbolt.
> Documented from actual troubleshooting — every detail here was verified against a working system.

---

## Background

I am not a Linux expert. I already owned the ThinkPad T14 Gen 3 and spent nearly **6 months**
unable to use the eGPU on Linux after buying the RTX 4070 SUPER and Razer Core X V2. I tried
countless kernel versions, driver versions, kernel parameters, and firmware tweaks. Nothing worked
reliably. This guide is everything I learned — the hard way.

After six months of trial and error, the combination documented works

---

## Hardware

| Component       | Details                                  |
| --------------- | ---------------------------------------- |
| Laptop          | Lenovo ThinkPad T14 Gen 3 (Intel)        |
| iGPU            | Intel Iris Xe Graphics (ADL GT2)         |
| eGPU Enclosure  | Razer Core X V2                          |
| eGPU Card       | NVIDIA GeForce RTX 4070 SUPER (ZOTAC)    |
| Connection      | USB4 @ 40 Gb/s (2 lanes × 20 Gb/s)       |
| Kernel          | `6.17.0-19-generic`                      |
| OS              | Linux Mint 22.3 Zena (Ubuntu Noble base) |
| Desktop session | GNOME on Wayland                         |
| NVIDIA Driver   | `580.126.09` (proprietary)               |
| CUDA            | 13.0                                     |

