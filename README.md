# Lumia 950 XL (msm8994) — mainline Linux 移植记录

> 一台 2015 年的 Windows Phone，跑上了 mainline Linux 5.18。
> **状态：暂停。** 显示链路、内核态 GPU 初始化已打通；用户态 3D 卡在 MMSS NoC，暂未解决。
> 留档给后来者：这里记录了做到哪一步、怎么做的、以及所有踩过的坑。

---

## 1. 这是什么

Nokia/Microsoft Lumia 950 XL 的 mainline Linux 移植实践记录。
不是发行版，不是完整固件包，而是一份**可复现的工程日志 + 补丁 + DT 草稿**。

硬件要点：

- SoC：Qualcomm MSM8994（骁龙 810）
- 屏幕：Samsung 5.7" 1440x2560 AMOLED，MIPI-DSI 4 lane
- GPU：Adreno 430（A4xx）
- 引导：Renegade edk2-msm UEFI（替换原 boot 分区）
- 内核：Linux 5.18（mainline）

---

## 2. 已经实现的

| 项目 | 状态 | 关键证据 |
|---|---|---|
| 面板驱动 `panel-samsung-950xl.ko` | ✅ | `panel probe OK` / `panel insmod rc=0` |
| 面板时序 | ✅ | 从富可视（FIH）PLAT 分区 XML 逆向出完整时序 + DCS 初始化序列 |
| MDP5 显示全链路 | ✅ | `mdp5_init_DONE` → `hw_init_done` |
| DRM card0 + fbdev | ✅ | `[drm] Initialized msm 1.9.0` / `fb0: msmdrmfb` |
| GPU 内核态初始化 | ✅ | `a420_pfp.fw` / `a420_pm4.fw` 加载，微引擎 `a4xx_me_init_ok` |
| 桌面 | ✅ | MATE (X11) 与 Phosh (Wayland) 均能出画面 |
| SMMU/TZ 交权 | ✅ | `restore_sec_cfg(18,0) -> 0`，MMU3 的 NS 窗口打开 |
| USB OTG 根因定位 | ✅ | 真凶 = `lumia-usb-gadget.service` 的 systemd ordering cycle |
| 稳定性 | ✅ | 去掉 kmscube 后连续运行 10 分钟无卡死、无 panic |

**没有做到的**：GPU 用户态 3D（OpenGL ES）、WiFi/蓝牙、摄像头、指纹。

---

## 3. 核心卡点：MMSS NoC

```
A430: no IOMMU, fallback to VRAM carveout!
a4xx_gpu_init: *ERROR* No memory protection without IOMMU
```

地址"明暗地图"显示：**MMU0~4 与 MMSS NoC 全暗，MDSS/GPU 亮**。
MDSS/GPU 走独立时钟源，所以能点亮；而所有 IOMMU 都挂在 MMSS NoC 后面，NoC 不通 → IOMMU 不可达 → GPU 无法分配受保护缓冲 → Mesa 只能退回 `llvmpipe`。

已排除/验证：

- 本机内核**没有** `msm_bus` / ICB fabric 驱动 → 无法照抄 QoS 直写
- 主线 `msm8994.dtsi` 的 `iommu|smmu` **零命中**，没有现成参照
- TZ 侧：`restore_sec_cfg(18,0)` 返回 0，说明 GPU 的 SMMU 授权已拿到，卡点不在 TZ 授权层

剩余可能方向（未验证）：

1. MMSS NoC 的 AXI 时钟链（MMCC CBCR 0x5064 / 0x506c，父 `axi_clk_src` @0x5040）
2. RPM 侧 ICB 投票
3. xPU/TZ 对 mnoc 的额外授权

---

## 4. 仓库结构

```
.
├── README.md
├── docs/
│   ├── 01-项目现状与卡点.md          # 事实底稿（成果/卡点/下一步）
│   ├── 02-进度-20260923.md
│   ├── 03-进度-20260924.md
│   ├── 04-进度-20260925.md           # 明暗地图 + Windows 开机配方
│   ├── 05-进度-20260926.md           # SMMU/TZ 交权、NoC 时钟实验
│   ├── 06-OTG-WiFi-差距清单.md
│   ├── 07-固件ACPI挖矿.md
│   └── 08-GPU加速方案.md
├── patches/
│   ├── 950xl-v31-noc-clk-wake.patch      # MMSS NoC 时钟链强制点亮（实验）
│   ├── 950xl-v32-noc-rescan.patch        # NoC 重扫
│   ├── 950xl-v33-mmu-discriminate.patch  # MMU 判别
│   ├── 950xl-v34-smmu-clk-test.patch     # SMMU 时钟测试
│   ├── 950xl-v35-noc-cfgscan.patch       # NoC 配置扫描
│   ├── 950xl-v36-rescan2.patch
│   ├── 950xl-v37-fix.patch
│   ├── 950xl-v38-sec-cfg.patch           # SMMU/TZ 交权 restore_sec_cfg(18,0)
│   ├── 950xl-usb-vusb11.patch            # USB dwc3 节点修复（dr_mode=peripheral）
│   ├── 950xl-iommu-defer.patch
│   ├── 950xl-gpu-iommu-DT-draft.dtsi
│   └── panel_samsung_950xl.dtsi
└── resources/
    └── MANIFEST.txt                  # 固件/驱动/DTS 来源清单，不入库
```

---

## 5. 复现步骤（高层）

1. **解锁并替换引导**：Renegade edk2-msm UEFI，写入 `boot` 分区
2. **准备内核**：Linux 5.18 + `CONFIG_DRM_MSM` + `CONFIG_QCOM_IOMMU` + 自编 panel 驱动
3. **准备 rootfs**：CachyOS/Arch 的 aarch64 rootfs（或 postmarketOS）
4. **部署 DTB**：`msm8994-msft-lumia-octagon.dtsi` 基础上加 DSI 节点 + panel 子节点 + `power-domains = <&mmcc MDSS_GDSC>`
5. **加载顺序**：`p13.service` 里 modprobe msm → 再起显示；依赖 DRM 的服务必须 `After=p13.service`

> 具体寄存器、时序、DTS 片段见 `docs/` 与 `patches/`。

---

## 6. 踩过的坑（血泪）

1. **探针禁止在 spin_lock 关中断区做文件 IO** —— 原子上下文睡眠，直接死机
2. **禁止用抢 DRM master 的工具测试**（如 kmscube）—— 整机冻死
3. **依赖 DRM 的 systemd 服务必须 `After=p13.service`** —— 否则 DRM 节点还没出现
4. **systemd ordering cycle 会静默删除服务** —— `lumia-usb-gadget.service` 因此从未启动
5. **不要盲扫 SCM sec_id** —— 错误 ID 会卡死 SoC
6. **手机无 RTC** —— 日志时间戳不可信，只能看顺序
7. **dwc3 找 PHY 用 `usb-phy` 属性**，不是 `phys`/`phy-names`（前者是 8994 的老式写法）
8. **危险读永远放最后** —— 读错地址会 abort，先做安全的时钟/寄存器写

---

## 7. 已知问题

- GPU 用户态无 3D（依赖 IOMMU，见卡点）
- WiFi：QCA6174（PCIe/ath10k），固件与 DT 节点未适配
- 蓝牙、摄像头、指纹未开始
- 合盖/休眠会进 suspend，NVIDIA 无关但内核 VRAM 保留参数需要另配

---

## 8. 免责与许可

- 刷机会变砖、丢数据，**风险自负**
- 文档：CC BY 4.0
- 补丁与 DTS：GPL-2.0（与内核一致）
- 不包含任何微软/高通的专有固件或系统文件（仅文档、DTS 与 GPL 补丁）；`resources/MANIFEST.txt` 只列出来源
- 内核源码不入库，只注明版本（Linux 5.18 mainline）与上游链接

---

## 9. 致谢

- Renegade Project（edk2-msm UEFI）
- postmarketOS / mainline msm8994 维护者
- 以及所有在 IRC/论坛里留下寄存器线索的人
