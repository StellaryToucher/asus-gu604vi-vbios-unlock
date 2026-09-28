# Raising the Whole-Machine Power Budget on an ASUS ROG Zephyrus M16 (GU604VI) by Cross-Flashing a Donor vBIOS

**A reproducible case study — platform power budget 155 W → 175 W, CPU headroom +20 W, materially improved 1% lows, with average FPS unchanged.**

[中文说明请见 README.zh-CN.md](./README.zh-CN.md)

---

## TL;DR

Cross-flashing the RTX 4070 Laptop GPU vBIOS from an ASUS ROG Strix G16 (2023, G614JI) onto a
Zephyrus M16 (GU604VI) measurably raised the **whole-machine power budget** from roughly **155 W to 175 W**.

- **Total system power draw:** ~155 W → **~175 W**
- **CPU package power (in-game):** ~40 W → **~55–60 W**
- **GPU power draw (in-game):** ~110 W → ~110–120 W (unchanged; voltage/frequency-limited, not power-limited)
- **Average FPS:** essentially unchanged
- **1% lows:** **markedly more stable** (fewer CPU-bound hitches)

In other words, this modification did not raise the performance ceiling — it raised the floor.

---

## Background

The ASUS ROG Zephyrus M16 (2023, GU604VI) pairs an Intel Core i9-13900H with an RTX 4070 Laptop GPU
(device ID `10DE-2860`) behind a 240 W adapter. Under sustained CPU+GPU load the machine would settle at
roughly **155 W combined**, with the GPU consuming ~110 W and the CPU throttled to ~40 W — despite the
CPU's own software power limit (PL1) being configured far higher, and despite the GPU never reaching its
own 140 W TGP.

This is characteristic of a **coupled power-arbitration design**, where the embedded controller (EC)
enforces a platform-wide budget rather than letting the CPU and GPU power limits act independently.

## Hypothesis

On many gaming laptops the GPU vBIOS is not limited to GPU-side parameters. It also carries the
**platform power provisioning** that the EC reads when computing the machine's total power budget.
Consequently, a vBIOS taken from a machine with a substantially higher platform budget (an HX-class CPU
platform with a 280 W adapter) should cause the EC to raise the total budget — with the surplus flowing to
whichever subsystem still demands more power.

Because this GPU operates at its voltage/frequency sweet spot (~110 W) and is **not** power-limited, it
does not request the surplus; the CPU does. The expected result is therefore improved CPU headroom and
better frame-time consistency, without a change in average FPS.

## Hardware

| | Target machine | Donor machine |
|---|---|---|
| Model | ASUS ROG Zephyrus M16 (GU604VI), 2023 | ASUS ROG Strix G16 (G614JI), 2023 |
| CPU | Intel Core i9-13900H (H-series) | Intel Core i7-13650HX / i9-13980HX (HX-series) |
| GPU | RTX 4070 Laptop, Device ID `10DE-2860`, Subsystem `1043-1473` | RTX 4070 Laptop |
| GPU TGP | 140 W (115 W + 25 W Dynamic Boost) | 140 W (115 W + 25 W Dynamic Boost) |
| Adapter | 240 W | 280 W |

Donor vBIOS metadata:

- File: `vbios/274670_Asus_ROG_Strix_G614_95.06.15.00.F2.rom`
- VBIOS version: `95.06.15.00.F2`
- Subsystem ID: `1043-14D3`; board: `E3757 SKU 10`; ASID: `N151G614JI.001`
- Reference: TechPowerUp VGA BIOS Collection, entry `274670`

Donor platform, as measured by third-party reviews (dual-load, 30 min):

- GPU ~129 W + CPU ~56 W ≈ **185 W** (Performance mode)
- GPU ~117 W + CPU ~78 W ≈ **195 W** (Manual mode, fans at max)

Notably, the donor's CPU draw under dual load (~56 W) closely matches the ~55–60 W observed on the
target machine after flashing — a strong indication that the EC adopted the donor's platform behaviour.

## Procedure

1. Back up the stock vBIOS with GPU-Z (two independent copies).
2. Confirm the GDDR6 memory vendor matches the donor ROM's supported list (Samsung / Hynix / Micron).
3. Disable Secure Boot in UEFI.
4. From an elevated terminal:

   ```text
   nvflash64 --protectoff
   nvflash64 -6 "vbios\274670_Asus_ROG_Strix_G614_95.06.15.00.F2.rom"
   ```

5. Reboot (a vBIOS change only takes effect on the next power-on).
6. Verify with GPU-Z and HWiNFO.

## Results

| Metric | Stock vBIOS | Donor (G614JI) vBIOS |
|---|---|---|
| VBIOS version | stock | `95.06.15.00.f2` |
| Default power limit | 80 W | **100 W** |
| Max power limit | 140 W | 140 W |
| Combined power budget (observed peak) | ~155 W | **~175 W** |
| CPU package power (in-game) | ~40 W | **~55–60 W** |
| GPU power draw (in-game) | ~110 W | ~110–120 W |
| Average FPS | baseline | unchanged |
| 1% low | baseline | **materially improved** |

The combined budget did not reach the donor's full 185–195 W, indicating that the target machine's own
EC and 240 W adapter impose an additional ceiling at approximately 175 W.

## Mechanism

1. A GPU vBIOS carries not only the GPU's TGP/Dynamic Boost values but also the platform power
   provisioning that the EC consumes when arbitrating the machine-wide budget.
2. The donor is a higher-power (HX-class) platform; on reading the new values the EC raised the total
   budget from ~155 W to ~175 W.
3. Because the target GPU is voltage/frequency-limited (~120 W) rather than power-limited, it did not
   absorb the surplus.
4. The surplus therefore went to the CPU — the only subsystem still demanding more — reducing CPU-bound
   hitching and improving 1% lows.

Put simply: setting a software power limit is a *request*; the vBIOS changes the *input* the EC uses to
compute what is actually permitted.

## Caveats

- **VRM / power delivery** is board hardware and is untouched by a vBIOS flash. For the same GPU tier the
  VRM is comparable across vendors (all observed 4070 Laptop ROMs share the reference board ID `E3757`).
  The relevant question is only whether the requested power exceeds the board's design — 140 W is within
  the target machine's factory rating.
- **The real risks of cross-flashing are not VRM-related**, but: EC/subsystem-ID mismatch (Dynamic Boost
  and power arbitration), display connector mapping (external HDMI/DP), fan tables, and MUX/Optimus
  behaviour. Verify all of these immediately after flashing.
- **Thermals.** The target machine is a thin chassis with a shared CPU/GPU cooling module; the higher
  budget increases heat output noticeably, and CPU heat bleeds into the GPU, causing some GPU clock loss.
  In this case the 1% low improvement was judged to outweigh the GPU clock reduction. CPU power can be
  trimmed via its PL (set both SPL and sPPT to avoid short bursts).
- With a 175 W budget and the GPU drawing ~120 W, the CPU's practical ceiling is roughly 55 W
  (175 − 120).

## Reproducibility

This is a single-machine case study, not a controlled benchmark. Results depend on the specific EC
firmware, adapter, and silicon. Reproduce at your own risk, and always retain a stock-ROM backup plus a
USB recovery path (PE + nvflash) for blind re-flashing.

## Files

- `vbios/274670_Asus_ROG_Strix_G614_95.06.15.00.F2.rom` — **primary donor ROM** (Strix G16 G614JI)
- `vbios/268423_Asus_ROG_Strix_G713_95.06.15.40.56.rom` — alternate: Strix G17 (G713PI)
- `vbios/271309_Asus_ROG_Strix_G814_95.06.15.00.F3.rom` — alternate: Strix G18 (G814JI)
- `vbios/265613_Asus_Zephyrus_M16_GU603VI_95.06.1D.00.19.rom` — alternate: Zephyrus M16 (GU603VI, same family)

## Sources

- vBIOS files: [TechPowerUp VGA BIOS Collection](https://www.techpowerup.com/vgabios/) (unverified uploads).
- Donor dual-load figures: independent review of the ROG Strix G16 / 魔霸新锐 2023 (smzdm.com).

## Disclaimer

Not affiliated with, sponsored by, or endorsed by ASUS or NVIDIA. Firmware remains the property of its
respective owners and is provided here for research and reference only. Flashing firmware carries a risk
of bricking the device. Proceed at your own risk.

## License

The documentation and any original code in this repository are released under the MIT License
(see [LICENSE](./LICENSE)). This license does **not** extend to the vBIOS firmware files under `vbios/`,
which remain the property of their respective owners.
