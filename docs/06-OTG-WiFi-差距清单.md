# 950XL 基建线：OTG/USB + WiFi 差距清单（2026-09-26 离线整理）

> 目标：让 950XL（跑 mainline 5.18）**能被宿主 SSH 进去**（USB gadget 网卡），并尽量修好 WiFi/蓝牙（无线键鼠）。  
> 现状：gadget 侧**已通**（UDC `f9200000.usb` 绑定成功、usb0 出现），缺的是**控制器/PHY 把设备呈现给宿主**。

## 一、USB/OTG 现状与根因

### 手机 dmesg 实证（关键）
```
[0.318] dwc3-qcom f92f8800.usb: error -ENXIO: IRQ hs_phy_irq not found
[0.318] dwc3-qcom f92f8800.usb: error -ENXIO: IRQ dp_hs_phy_irq not found
[0.318] dwc3-qcom f92f8800.usb: error -ENXIO: IRQ dm_hs_phy_irq not found
[0.318] dwc3-qcom f92f8800.usb: error -ENXIO: IRQ ss_phy_irq not found
[7.485] systemd: lumia-usb-gadget.service ordering cycle → job deleted
```
- **根因 A**：`dwc3-qcom` wrapper 要求 `hs_phy_irq/dp_hs_phy_irq/dm_hs_phy_irq/ss_phy_irq`，DT 没给 → probe `-ENXIO` → 控制器未完整起来 → 宿主看不到设备
- **根因 B**：`lumia-usb-gadget.service` 有 ordering cycle，开机被 systemd **删掉**（gadget 没自动配）
- **根因 C**：手机 rootfs 无 `iproute2`（`ip: not found`）→ usb0 配不上 `10.55.0.2/24`

### mainline 现状（`msm8994.dtsi`）
```dts
usb3: usb@f92f8800 {
	compatible = "qcom,msm8996-dwc3", "qcom,dwc3";   /* 抄的 8996 */
	clocks = ...; assigned-clocks = ...; power-domains = <&gcc USB30_GDSC>;
	qcom,select-utmi-as-pipe-clk;
	usb@f9200000 {                                    /* snps,dwc3 */
		compatible = "snps,dwc3";
		maximum-speed = "high-speed";
		dr_mode = "peripheral";                       /* 已是 gadget 模式 */
	};
};
```
- **没有 `phys` / `phy-names`** → dwc3 core 没挂 PHY
- **没有 PHY 子节点**（HS PHY / SS PHY）
- **没有 PHY 的 4 个中断**
- Lumia 的 `msm8994-msft-lumia-octagon.dtsi` **对 USB 零覆盖**
- PHY 驱动：`phy-qcom-qusb2`（HS）**没有 msm8994 compatible**；`phy-qcom-qmp`（SS）也**没查到 8994 USB3**

## 二、待办（按性价比排序）

1. **给 dwc3-qcom 补 PHY 中断**（最可能是"一眼通"的那步）
   - 从 ACPI/DSDT 或下游 CAF dtsi 找 `hs_phy_irq/dp_hs_phy_irq/dm_hs_phy_irq/ss_phy_irq` 的 GIC 号
   - 或先用 `interrupts-extended`/可选化绕开（改成 not-found 不致命）
2. **加 HS PHY 节点**（`qusb2`）并接到 dwc3 core 的 `phys`
   - `maximum-speed = high-speed` → **SS PHY 不是必需**，先攻 HS PHY
   - 试探：`compatible = "qcom,msm8996-qusb2-phy"` 复用到 8994（寄存器布局可能相近）
3. **修 gadget 服务 ordering cycle**：`lumia-usb-gadget.service` 的 `After/Before` 去掉循环
4. **手机装 `iproute2`**（或用 busybox `ip`）→ usb0 配 `10.55.0.2/24`；手机 sshd 已 enable
5. （可选）**SS PHY（QMP USB3）**：若要 USB3 速率/主机模式，需给 `phy-qcom-qmp` 加 8994 表

## 二·五、实战结果（2026-09-26 晚）
- **v-usb1**（加 HS PHY 节点 @0xF9B3A000 + 挂 dwc3 + dr_mode=otg）：能进桌面，但 USB 无反应；dmesg **无 qusb2 probe 日志**；`dwc3-qcom` 仍 -ENXIO（4 个 PHY IRQ）。
- **v-usb2**（PHY 驱动编进内核 `=y` + 补 3 个 dummy 供电轨）：同上，且**触摸失灵/系统迟钝** → 判定为负面。
- ⇒ **结论**：卡点在 **dwc3-qcom wrapper 的 4 个 PHY 中断**（`hs_phy_irq`/`dp_hs_phy_irq`/`dm_hs_phy_irq`/`ss_phy_irq`；ACPI 里 USB0 给了 GIC **0xA3(163)** 与 **0xD4(212)**）。下次先补这几个中断再试。
- 手机已刷回 **v47（稳定）**，DTB 也恢复原版。

## 二·六、v-usb4（自写驱动）与下一步（2936 晚更新）
- 拿到下游真源码后确认：**DWC3 的 HS PHY = `hsphy@f92f8800`（`qcom,usb-hsphy`）**，core `0xF92F8800` + phy_csr `0xF9B3A000`，`qcom,hsphy-init = 0xD191A4`，只需 `phy_sleep_clk` + 4 路供电
- **v-usb4**：自写最小驱动 `drivers/phy/qualcomm/phy-msm8994-usb-hs.c`（兼容 `qcom,msm8994-usb-hsphy`）→ **编译通过、能开机，但 OTG 仍未通（同之前）**
- **下一步（明天）**：**加探针**——异步读 `0xF92F8814`(期望 `0xD191A4`，验 PHY init 有没有写) / `0xF9200000`(dwc3 GCTL) / `0xF9200008` / `0xF9A55000` / `0xF9B39000`；另：探针条目要插在 `mo_rd_list[]` 内（用括号配对定位，别手插错位置）
- ⚠️ 已踩坑：探针插错位置连错 3 版+误刷旧 Image → 已恢复 **v47 + 原版 DTB**

## 三、WiFi/蓝牙（第二步）

- 芯片：**Qualcomm WCN3680B**（WiFi+BT combo，集成在 8994）
- 硬件能力：**802.11 a/b/g/n/ac 双频（2.4G+5G）**；无线键鼠走**蓝牙**（同芯片）
- mainline 需要：`wcn36xx`（WiFi）+ `btqca`（BT）+ `qcom,wcnss` remoteproc + 固件 `wcnss.mbn`（在手机分区里）
- 现实目标：**先通 2.4GHz**；5G/ac、蓝牙看驱动成熟度
- 要补：DT 的 wcnss/pronto 节点、irq、regulator、固件路径

## 四、验证路径（通了长啥样）

OTG 一通：
- 手机上 `usb0 = 10.55.0.2/24`，宿主侧出现对应 usb 网卡 → `ssh wmq@10.55.0.2`（手机 sshd 已 enable + 公钥已装）
- 之后**读日志/改文件/换内核都不用插拔** → 迭代提速

---

## 二·七、2026-09-26 深夜续攻（v-usb6 → v-usb9）★ 重大进展 + 新工具

### 🎉 新工具：离线读手机内核日志！
手机 rootfs 有**持久化 systemd journal** → 在 CachyOS 上直接读（不用启动手机）：
```bash
mount /dev/sda43 /mnt/lumia
journalctl -D /mnt/lumia/var/log/journal --no-pager -k          # 内核日志
journalctl -D /mnt/lumia/var/log/journal -k | grep -iE "usb|xhci|dwc3|phy|Bluetooth"
umount /mnt/lumia
```
（`probe.log` 是我们自己的探针；journal 是**内核原生日志**——从此不用猜）

### 关键修正（DT）
1. **外层 `qcom,dwc3` 地址错了**：原来写 `0xf92f8800`（**那是 HS PHY 的地址！**）→ 正确 = `0xf9288000`（= core `0xf9200000` + `0xf8800`，**主线 msm8996 就是这个规律**，对照 `msm8996.dtsi` 的 `usb@6af8800` = `0x6a00000+0xf8800`）
2. **补上中断**：`hs_phy_irq`=GIC_SPI 133 + `pwr_event_irq`=GIC_SPI 180（下游 `interrupt-map` 挖出）
3. **`dr_mode` 试 `"host"`**（内核 `DWC3_DUAL_ROLE=y` + `XHCI_HCD=y` ✅ 支持）

### 探针（v-usb8/9，同步读，放 `mo_late_work` 里）
- ⚠️ **重大发现**：**异步探针（msleep）在这台机器上会失效**——设备**疯狂 suspend/resume**（日志里 `[resume]` 出现 **580 次**！），一挂起 msleep 就不走了 → 探针永不触发。**以后探针一律同步读**。
- 结果：
  - `hsphy+0x14 = 0x00d191a4` ✅（init 写入）
  - `GSNPSID = 0x5533250a` ✅（DWC3 核活着）
  - `GCTL = 0x00111401` → **PRTCAPDIR = HOST** ✅（host 模式生效）
  - `f9288000` 可读 ✅（新地址对）
- DWC3 寄存器偏移（`drivers/usb/dwc3/core.h`）：`GCTL=0xc110` `GSTS=0xc118` `GSNPSID=0xc120` `DCFG=0xc700` `DCTL=0xc704` `DSTS=0xc70c` `GUSB2PHYCFG0=0xc200`

### 内核日志结论（v-usb9）
```
✅ msm8994-usb-hsphy: registered (init=0x00d191a4)
✅ msm8994-usb-hsphy: hsphy on: wrote 0x00d191a4
✅ xhci-hcd: xHCI Host Controller / new USB bus registered, bus 1 & 2
✅ hub 1-0:1.0 found / hub 2-0:1.0 found
⚠️ usb usb2-port1: Cannot enable. Maybe the USB cable is bad?   （每 4s 重试，SS 口；我们没配 ssphy，属正常）
❌ USB2 口**完全没检测到设备**（无任何枚举日志）
```
+ 彩蛋：**蓝牙在探** → `Bluetooth: hci0: Reading QCA version information failed (-110)` / `Retry BT power ON` → **QCA6174 的 BT 链路是活的**（走串口）

### 结论
- **PHY + DWC3 + XHCI 全绿**，就卡在**设备没上电**：
- **命门 = VBUS 供电**：host 模式要给设备供 5V，归 **PMIC（SMBCHARGER）的 OTG boost** 管，**主线没这个驱动** → 设备不通电 → 端口啥也看不到
- ⚠️ **副作用**：开 host 模式后**触摸坏**（和 v-usb2/3 同一模式，多半是供电/GPIO 被搅）

### 下一步（明天）
1. **VBUS/PMIC（smbcharger OTG）**——OTG 的命门，硬骨头
2. 或试**独立供电的 USB hub**（绕开手机供 VBUS）
3. **或转 WiFi/PCIe**：不依赖 OTG；且**蓝牙已在探 QCA** → 也许从 BT 侧突破更近
4. 补 **SS PHY**（`ssphy@f9b38000`，`qcom,usb-ssphy-qmp`）——顺带把 SS 口那个报错消掉

### 当前状态
手机 = **v47 + 原版 DTB**（稳定，触摸正常）；镜像 vusb6~9 + 日志都在 CachyOS `~`

---

## 二·八、2026-09-27 白天：vusb10/vusb11 → **真凶抓到 = systemd ordering cycle** ★

### 时间线
- **16:23~16:27 刷 vusb10**：= vusb9 的 Image + `dr_mode: host→peripheral` 的新 dtb（只 DTS 变）
- **16:31 开机进桌面**（MATE，截图为证；屏幕/键盘正常）→ 但 **PC 侧完全没认到**（`lsusb` 无手机，dmesg 停在 `045e:0a02 Lumia BootMgr` 的 disconnect）
- **16:44 离线读 journal**：**整轮 journal 里一条 usb/dwc3/phy 日志都没有**（蓝牙 hci0 的日志却在 → journal 没坏）→ 判定 **dwc3 根本没 probe**

### 对比 mainline 节点（手搓的错在哪）
| 项 | 我们的（vusb10） | mainline（对） |
|---|---|---|
| compatible | `qcom,msm8996-dwc3` | `qcom,msm8994-dwc3`, `qcom,dwc3` |
| reg | `0xf9288000` | `0xf92f8800` |
| interrupts | 2 个（`hs_phy_irq`=133, `pwr_event_irq`=180） | 4 个：`pwr_event`=180, `qusb2_phy`=311, `hs_phy_irq`=133, `ss_phy_irq`=310 |
| clock-names | 6 个（clocks 只有 4） | 4 个（`core/iface/sleep/mock_utmi`）|
| core 子节点 | 挂了 `phys=<&usb_hs_phy>` | 无 phys |

- ⚠️ **重要更正**：`dwc3` 核心找 PHY 用的是 **`usb-phy`** 属性（`devm_usb_get_phy_by_phandle(dev,"usb-phy",0)`），**根本不看 `phys`/`phy-names`** → 之前"挂了自写 PHY 才 defer"的锅**甩错了**；真凶更可能是 **reg 地址写错**（`f9288000` 不是 glue 的 QSCRATCH）。

### 16:48 **vusb11** = 整块换回 mainline 原版 `usb3` 节点 + `dr_mode="peripheral"`；并停掉自写 PHY 节点（它和 glue 抢 `0xf92f8800`）
- 产物：`~/vusb11-Image`（md5 同旧，只 DTS 变）+ `~/vusb11-cityman.dtb`（`93b33ce2…`）；patch 存档 `~/950xl-usb-vusb11.patch`

### 18:05 实测：**端口第一次"活"了** 🎯
PC 侧 dmesg：
```
usb 1-4: new high-speed USB device number 56 using xhci_hcd     ← 有挂号了（vusb10 时零反应）
（之后无 "New USB device found" → 描述符读不出，枚举失败）
```
手机 journal：`dwc3-qcom f92f8800.usb: error -ENXIO: IRQ dp_hs_phy_irq not found`（可选 IRQ，不致命）→ **glue 终于 probe 起来了**

### 18:01 读 journal（vusb11 启动）→ **真凶现形**：gadget 服务被 systemd 删了
```
systemd[1]: multi-user.target: Found ordering cycle on lumia-usb-gadget.service/start
systemd[1]: multi-user.target: Found dependency on p13.service/start
systemd[1]: multi-user.target: Found dependency on gpu-on.service/start
systemd[1]: multi-user.target: Found dependency on multi-user.target/start
systemd[1]: Job lumia-usb-gadget.service/start deleted to break ordering cycle
```
环 = `gadget →After→ p13 →After→ gpu-on →After→ multi-user →Wants→ gadget`
（`p13`/`gpu-on`/`msm-run` 都是**故意排到 multi-user 之后**等桌面起来，而 gadget 又挂在 multi-user 上并依赖 p13 → **必成环**）
→ **gadget 服务从未启动** → UDC 从未绑定 → 设备不响应描述符请求 = "看到插入却读不出描述符"的根因。

### 18:07 修复（备份 `.bak-20260927-ordering`）
1. `lumia-usb-gadget.service`：去掉 `After=p13.service` + `Wants=p13.service`，只留 `After=local-fs.target`（gadget 不需要 p13：`dwc3` 是内建驱动，开机就 probe 了）→ **环破**
2. gadget 脚本：加"等 UDC 最多 30s"重试（防时序抖动）

### ⏸ 未验：18:39 手机没电
- **当前状态**：手机 = **vusb11 + 破环后的 gadget 服务**（待测）；`/boot` 备份 `.bak-pre-vusb11-20260927-1648`
- 顺带查了 USB-C 那条线：`uc120: ice5lp2k@0`（Lattice iCE40 = USB-C PD PHY，走 `blsp2_spi4`；主线注释："pre-programmed by the device firmware … enabling USB Type C PHY functionality"）—— 但 `.config` 里 **`CONFIG_FPGA_MGR_ICE40_SPI` 未开**（依赖 `OF && SPI`，随时能开）；且 mainline 说该 FPGA 由固件预编程 → **未必需要 Linux 再灌**
- ⚠️ 乌龙：红头"充电线"是**纯供电线**（无 D+/D-）→ 插 PC 永远认不到；必须用**能传数据的线**

### 待办（下次开机）
1. 换**数据线** → 手机开机进 Linux → 插 CachyOS
2. 预期：gadget 服务跑起来 → UDC 绑定 → PC 侧出 `1d6b:0104` + `usb0` → 宿主侧配 IP → 手机 `usb0=10.55.0.2/24` → `ssh wmq@10.55.0.2`
3. 若仍不出：读 journal + `/home/lumia/usb-gadget.log` + `ls /sys/class/udc`
4. （备选）开 `CONFIG_FPGA_MGR_ICE40_SPI` / 把自写 PHY 挂到 **`usb-phy`** 属性上

### 资料囤积（2026-09-27）
`workspace/950xl-resources/`（44 文件 / 3.3MB）：mainline DTS（msm8994/octagon/kitakami）、驱动（pcie-qcom、dwc3-qcom、msm_iommu、a4xx_gpu、ice40-spi、ath10k…）、**QCA6174 全套固件**、DT bindings
