# ASUS ROG Zephyrus M16 (GU604VI) — Cross-Flash vBIOS to Lift the Whole-Machine Power Budget

> 通过给华硕幻16 GU604VI 刷入同厂 ROG 魔霸新锐（Strix G16, G614JI）的 RTX 4070 Laptop vBIOS，
> 实测把**整机功耗预算从约 155W 抬到约 175W**，**CPU 拿到 +20W（40W→55~60W）**，**1% low 显著变稳**，
> 平均帧基本不变（GPU 受电压/频率限制，并非功耗受限）。

---

## 一句话结论 / TL;DR

- **整机总盘**：155W → **175W**
- **CPU**：40W → **55~60W**
- **GPU**：~110–120W（未变，电压墙限制）
- **效果**：平均帧不变，**1% low 明显改善**（卡顿减少）

Flashing an ASUS ROG Strix G16 (2023, G614JI) 4070 Laptop vBIOS onto a Zephyrus M16 GU604VI
raises the platform power budget (~155W → ~175W). The CPU gains ~20W, **1% lows improve markedly**,
average FPS unchanged (the GPU is voltage/frequency-limited, not power-limited).

---

## 目标机 / Target machine

| | |
|---|---|
| 型号 | ASUS ROG Zephyrus M16（幻16）GU604VI, 2023 |
| GPU | NVIDIA RTX 4070 Laptop 8GB, Device ID `10DE-2860`, Subsystem `1043-1473` |
| CPU | Intel Core i9-13900H |
| 适配器 | 240W |

## 刷入的 vBIOS / Donor vBIOS

- 文件：`vbios/274670_Asus_ROG_Strix_G614_95.06.15.00.F2.rom`
- 出处机型：ASUS ROG 魔霸新锐 / Strix G16（2023）**G614JI**
- Subsystem `1043-14D3`；板号 `E3757 SKU 10`；ASID `N151G614JI.001`
- 供体平台规格：HX CPU（i7-13650HX / i9-13980HX）、RTX 4070 140W（115W + 25W DB）、280W 适配器
- 供体实测双烤：GPU 129W + CPU 56W ≈ **185W**（增强模式）；手动全速 **195W**

## 原理 / Why it works

1. GPU vBIOS 里不只有显卡的 TGP/DB，还带着 **EC 用来计算整机功耗预算的平台配置**。
2. 供体是**更高功耗的 HX 平台**，EC 读到新值后把**整机天花板**从 155W 抬到了 175W。
3. 因为本机 GPU **被电压/频率限制在 ~120W**（并非功耗受限），多出来的预算就分给了**唯一还想要更多电的 CPU**。
4. 于是 CPU 松绑 → **CPU 瓶颈的瞬间不再被压 → 1% low 变稳**；而 GPU 侧没变 → 平均帧不变。

> 软件设 PL 只是“请求”；vBIOS 改的是 EC 用来算预算的“输入”。

## 操作步骤 / How to flash

1. 用 **GPU-Z** 备份原厂 vBIOS（两份）。
2. 确认**显存品牌**与要刷的 ROM 匹配。
3. BIOS 里**关闭 Secure Boot**。
4. 管理员终端：
   ```text
   nvflash64 --protectoff
   nvflash64 -6 "vbios\274670_Asus_ROG_Strix_G614_95.06.15.00.F2.rom"
   ```
5. 重启，用 GPU-Z / HWiNFO 核对。

## 实测对比 / Measured results

| 项目 | 原厂 vBIOS | 刷入 G614JI vBIOS |
|---|---|---|
| VBIOS Version | (stock) | `95.06.15.00.f2` |
| Default Power Limit | 80W | **100W** |
| Max Power Limit | 140W | 140W |
| 整机总盘（实测峰值） | ~155W | **~175W** |
| CPU 功耗（游戏） | ~40W | **~55–60W** |
| GPU 功耗（游戏） | ~110W | ~110–120W |
| 平均帧 | 基准 | 基本不变 |
| **1% low** | 基准 | **明显更稳** |
| 副作用 | — | CPU 热经共享散热传导，GPU 会掉一点频 |

## 风险与注意 / Warnings

- **刷写有变砖风险**，务必先备份原厂 ROM，并准备 U 盘盲刷环境。
- 跨机型刷写可能影响：**外接 HDMI/DP、风扇策略、Dynamic Boost、MUX/独显直连**，刷后请立刻验证。
- 供电（VRM）是主板硬件，跨刷不会改变；4070 Laptop 各厂多为同一参考板（板号 `E3757`）。
- 本机为**轻薄模具**，总盘抬高后**发热明显增加**；建议用 CPU PL 裁量（SPL 与 sPPT 一起限）。
- 本机共享散热：**CPU 的每一瓦会部分转嫁到 GPU**，需在“1% low 收益”与“GPU 掉频”之间取舍。

## 文件 / Files

- `vbios/274670_Asus_ROG_Strix_G614_95.06.15.00.F2.rom` — **主 ROM（成功案例）**，源自 G614JI
- `vbios/268423_Asus_ROG_Strix_G713_95.06.15.40.56.rom` — 备选：魔霸 G17（G713PI）
- `vbios/271309_Asus_ROG_Strix_G814_95.06.15.00.F3.rom` — 备选：魔霸 G18（G814JI）
- `vbios/265613_Asus_Zephyrus_M16_GU603VI_95.06.1D.00.19.rom` — 备选：幻16（GU603VI，同系）

## 来源 / Sources

- vBIOS 均取自 [TechPowerUp VGA BIOS Collection](https://www.techpowerup.com/vgabios/)（未验证上传区）。
- 主 ROM 对应 TPU 条目 id `274670`。
- 供体双烤数据来源：什么值得买《ROG魔霸新锐2023真机深度实测报告》。

## 免责声明 / Disclaimer

- 本项目与 ASUS / NVIDIA 无任何关联。
- 固件版权归 ASUS / NVIDIA 所有，此处仅用于研究与技术交流。
- 刷写风险自负。
- Not affiliated with ASUS or NVIDIA. Firmware remains the property of its respective owners;
  provided for research/reference only. Flash at your own risk.
