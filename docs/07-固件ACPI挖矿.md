# Lumia 950 XL（MSM8994）固件 ACPI 挖矿成果 — 2026-09-25 凌晨

> 目的：给 GPU IOMMU 挪移找"Windows 是怎么做的"官方答案。
> 结果：**找到了**。固件直接指出了我们唯一的差异（NoC 投票 + XPU 权限）。

## 0. 数据来源（全部离线可复现）

| 素材 | 位置 | 说明 |
|---|---|---|
| **PLAT 分区**（`sda34`, 8M vfat） | 手机；已导出 `~/lumia-backup/plat.bin`(8M) | **里面直接躺着完整 ACPI 表**！（EFIESP 里的 "DSDT" 是字符串误报） |
| 抠出的 ACPI | `~/lumia-backup/acpi/`：`DSDT.aml`(184851B) `DSDT.dsl`(1.8M 反汇编) `FACP/SSDT/APIC/GTDT/MCFG.aml` | `iasl -d DSDT.aml` |
| UEFI 分区（`sda19` 2M） | `~/lumia-backup/phone-uefi.bin` | 2M **ARM ELF**（EDK2 固件），内部含 2 个 `_FVH` 固件卷；**无明文 ACPI** |
| EFIESP（`sda35` 32M） | `~/lumia-backup/efiesp.bin` | ACPI 签名全是误报 |
| 原厂固件 FFU（RM-1085） | CachyOS `~/ffu_extract/.../RM1085_1078.0053.10586.13169.11302.034DF6_retail_prod_signed.ffu`(3.0G) | 出厂 Windows 10 Mobile；**压缩格式**，需 `ffu_tool` 之类才能解 |
| 分区表参考 | 手机 `lsblk`：`sda19 UEFI / sda34 PLAT / sda35 EFIESP / sda37 MainOS(ntfs,WOA) / sda43 rootfs(Linux)` | |

DSDT 头：`OEM="QCOMM " TABLEID="MSM8994 "`；FACP/APIC/GTDT/MCFG = `QCOM/QCOMEDK2`。

## 1. ★ GPU IOMMU 的 ACPI 设备定义（地址疑问彻底终结）

```
Device (GPU0)   _HID="QCOM246F"   _UID=0
  _DEP = { \_SB.MMU0, \_SB.MMU3, \_SB.MMU4, \_SB.PEP0, \_SB.OMEM, \_SB.PMIC, \_SB.DISP, \_SB.HDDP }
  _CRS: fd900000+CC000(MDP_REGS) / fd828000+108(MMSS_MISC_REGS) / IRQ 0x73
        fdb00000+10000(GFX_REGS) / fdb20000+1F000(GFX_SHADER_MEMORY) / IRQ 0x41
        fdc00000+84000 / fdcc0000+40000(VIDEO) / IRQ 0x4C
        fc4b8000+8000(SE_CTRL_REGS) / GPIO 0x4E(DSI_PANEL_RESET)
  ※ fdb10000 不在 GPU0 的 _CRS 里 —— 它属于独立设备 MMU3
  RESI: RESOURCE 名称表(MDP_REGS/MMSS_MISC_REGS/GFX_REGS/GFX_SHADER_MEMORY/...)
  ENGS: ENGINES=5，含 "GRAPHICS/Adreno4XX"+MMUIDS=ToUUID("9833c712-3292-4ffb-b0f4-2bd20e1f7f66")

Device (MMU3)   _HID="QCOM2400"  _UID=3      ← **= GPU IOMMU**
  _CRS: Memory32Fixed 0xFDB10000 + 0xB000
        IRQ 0x46 (=70) / 0x110 / 0x111 / 0x112
  GUID(): ToUUID("9833c712-3292-4ffb-b0f4-2bd20e1f7f66")   ← 与 GPU0 的 MMUIDS 一致
  CTXI(): Buffer{ 0x03, 0x00, 0x00, 0x01 }
  S2CB(): Package 3 组 (SID→CB): (0,0,0,0) (1,1,0,0) (2,2,0,2)
  XPUE(): Buffer{ 0x01 }        ← XPU 权限条目

其他 MMU（每个都有 CTXI/OFFI/XPUE）：
  MMU0 = 0xFD9CC000 (MDP/显示，与下游 mdp_iommu_8994 一致)
  MMU1 = 0xFDA44000
  MMU2 = 0xFDA64000
  MMU4 = 0xFDC84000 (venus，下游同地址；下游给它配了 clk_base=0xfdce0004)
  MMU5 = 0xFDA84000
  MMU6 = 0xFD864000
```
→ **固件自己证明 GPU IOMMU 就是 `0xFDB10000`**；并给出 SID→CB 映射与 XPU 条目。

## 2. ★★ PEP（电源引擎插件）里 GPU 的**上电序列**（Windows 的 "标准答案"）

DSDT 中的 PEP 设备电源状态表（FSTATE）里，GPU 组件 `ENTER` 序列（顺序即固件意图）：

```
CLOCK:  oxili_rbbmtimer_clk        ← v15 实测：它原本 bit0=0（关着），我们补开了
CLOCK:  oxilicx_ahb_clk
CLOCK:  oxili_gfx3d_clk
BUSARB: ICBID_MASTER_APPSS_PROC → ICBID_SLAVE_GFX3D_CFG   ← ★ AP 先对 NoC 从端口投票
PSTATE_SAVE / PSTATE_ADJUST
FOOTSWITCH: VDD_OXILI   (0x02)
FOOTSWITCH: VDD_OXILICX (0x02)
REQUIRED_RESOURCE: (1, "/clk/oxili", 0)
（EXIT 逆序，含 BUSARB 0x11E1A300）
```
同类条目大量出现在其他子系统（如 `ICBID_SLAVE_DISPLAY_CFG`），说明这是 WOA 的统一"总线投票"机制。

**对照**：下游 `msm8994-bus.dtsi` 有 `qcom,msm-bus-device`（`reg = <0xFC460000 0x5880>` + snoc/bimc/pnoc/mnoc/cnoc 基址）→ 8994 的 NoC 是 **AP 可直接编程的 QoS 寄存器** 集合；下游给 kgsl_iommu 投的票是 `<26 512>`（APPS→EBI）。

## 3. 结论：我们只差最后一项

- 电源域（GX/CX）、稳压器（VDD_OXILI/CX）、三个时钟 → 我们**全部已验证在跑**（v12/v15 实测）
- 唯一没做的：**`BUSARB`（APPS_PROC → GFX3D_CFG 的 NoC 投票）**，外加 **XPU 权限**（ACPI 每个 MMU 都带 `XPUE`）
- 与实测现象完全自洽：**写被静默丢弃（posted write）、读报错（需要从端口响应）** = NoC 到该从端口的通路没打开

## 4. 下一步候选

1. **Windows 热重启旁路**（最省事）：UEFI 菜单 → Windows → 桌面 → **重启（非关机）** → 选 Linux(v16，开机 <1s 就有 early 读) → 拍屏幕。
   - 读到值 → 证明"运行期配置 + 跨热重启保留"，并直接给出可用绕行
   - 还是死 → 配置不跨复位保留 → 必须自己复刻 NoC 投票
2. **复刻最小 NoC 投票**：照下游 `msm_bus` 的 QoS 寄存器（`0xFC460000` 段）给 APPS_PROC→GFX3D_CFG 编程
3. **XPU 权限**：找 XPU 寄存器（或用 SCM）读/改
4. **解 FFU**（`ffu_tool`）拿原厂 UEFI/PLAT/ACPI 做对照；同 FFU 也是**刷回原厂的安全网**

## 5. 素材清单（都在 CachyOS `~/lumia-backup/`）

```
plat.bin(8M)  efiesp.bin(32M)  phone-uefi.bin(2M)
acpi/{DSDT.aml, DSDT.dsl, SSDT.*, FACP.aml, APIC.aml, GTDT.aml, MCFG.aml}
phone-20260925/{vmlinuz-5.18.0-rc1.v16, cityman.dtb.v14}
```
手机 `/boot` 备份：`vmlinuz-...bak-{probe,planA,planB,v3,v4,v5,v6b,v6c,v7,v8?...}`、`...cityman.dtb.bak-{probe,gfx3d,dtfix1,safe,v7,v8,v9,v13,v14}`

---

## 6. ★ 重大补充：Qualcomm 官方文档说明"xPU 从端口访问控制"（2026-09-25 01:20）

来源：寄晴从 GPT 那拿到的 **Qualcomm Linux Security Guide → Access control**（`docs.qualcomm.com/doc/80-70014-11/topic/access-control.html`，Updated 2024-08-06）

原文关键句（摘录）：

> The Qualcomm access control uses an **external protection unit (xPU)** to control access **from the slave side for registers**, fixed address, and dynamic memory regions.
> The system MMU controls access from the master side …
> **There are two levels of control:**
> - **The TZ manages the TZ domains and controls access from the slave side using xPUs.** An xPU is a combination of multiple security blocks known as protection units. It allows conditional access based on **a set of programmable access control registers**.
>   **If access is denied, the xPU generates an error output signal and, if necessary, an interrupt request signal.**
> - The hypervisor (EL2) manages non-secure domains … configures access control from the master side through second-stage page tables …

**这正好解释了我们的现象**：
- xPU 管的是"**从端口（slave side）的寄存器访问**" ← 我们的 `0xFDB10000` 就是一个 slave
- 权限由"**一组可编程的访问控制寄存器**"决定，**由 TZ 管理**
- **访问被拒 → xPU 输出错误信号（并可能中断）** ← 我们的"读报错 + 平台硬复位"，写被丢弃

→ 结论：`0xFDB10000`（MMU3）**极可能是被 xPU 拒绝给 AP（非安全世界）访问**，而不是单纯的 NoC 未投票。
→ 而 ACPI 里每个 MMU 设备都带的 `XPUE` 方法（MMU3 返回 `Buffer{0x01}`）= **xPU 权限条目**，正是"谁被允许访问这个从端口"的规格来源（Windows 侧由 UEFI/PEP/ICB 消费并配置）。

**下一步（更新）**：
1. 找出 **MSM8994 的 xPU 寄存器基址与格式**（从 ACPI `XPUE` 条目 + UEFI/下游代码推），看能否由 Linux 直接读/配
2. 或走 **TZ/SCM**（xPU 由 TZ 管）：找有没有"配置 xPU / 授权访问"的 SCM 调用
3. 与 NoC 投票（BUSARB）两条线并行验证：**两者都可能是同一套"授权链"的环节**
4. 旁证：UEFI 是否已按 ACPI 配好 xPU（若已配好，我们被拒的原因可能变成"master ID/VMID 不匹配"）

---

## 7. ★★★ 找到 xPU 的"开关"：CAF `drivers/soc/qcom/scm-xpu.c`（2026-09-25 01:33）

线索链：CAF `drivers/soc/qcom/Kconfig` 的 `QCOM_SCM_XPU`（"Qualcomm XPU configuration driver"，`depends on QCOM_SCM`）
→ Makefile：`obj-$(CONFIG_QCOM_SCM_XPU) += scm-xpu.o`
→ **源码 `drivers/soc/qcom/scm-xpu.c`**（已抓到，1445 字节）

```c
#if defined(CONFIG_QCOM_XPU_ERR_FATAL)
#define ERR_FATAL_VAL 0x0
#elif defined(CONFIG_QCOM_XPU_ERR_NONFATAL)
#define ERR_FATAL_VAL 0x1
#endif
#define XPU_ERR_FATAL 0xe            /* SCM 命令号 */

static int __init xpu_err_fatal_init(void)
{
	struct { unsigned int config; unsigned int spare; } cmd;
	struct scm_desc desc = {0};
	desc.arginfo = SCM_ARGS(2);
	desc.args[0] = cmd.config = ERR_FATAL_VAL;   /* 0=fatal, 1=non-fatal */
	desc.args[1] = cmd.spare = 0;
	if (!is_scm_armv8())
		ret = scm_call(SCM_SVC_MP, XPU_ERR_FATAL, &cmd, sizeof(cmd), &response, sizeof(response));
	else
		ret = scm_call2(SCM_SIP_FNID(SCM_SVC_MP, XPU_ERR_FATAL), &desc);
	...
}
early_initcall(xpu_err_fatal_init);
```

**关键信息**
- SCM 服务 = `SCM_SVC_MP`（memory protection），命令 = `0xe`（XPU_ERR_FATAL）
- 参数：`config`（**0x0 = 违规视为 fatal；0x1 = non-fatal**）+ `spare`
- 这是 `early_initcall`，即"启动时通过 SCM 设置 xPU 违规行为"

**推论（与现象高度吻合）**
- 我们的"读一下 `0xFDB10000` 整机硬复位"= **xPU 违规，且被配置为 FATAL**（CAF 默认就是 FATAL）✓
- **可立刻验证**：在 mainline 用 `qcom_scm` API 复刻同一个调用、把违规改成 **non-fatal(0x1)**，再读 `0xFDB10000`：
  - 变成"报错但机器活着" → **xPU 假设实锤**，且我们获得稳定可调试的平台
  - 注意：**non-fatal ≠ 授权**（只是不再复位），授权那一步还需另找（可能在 TZ 启动配置 / UEFI 的 ACPI XPUE 消费 / 另一条 SCM 命令）

**v17（待做，离线可写好）**：`early_initcall` 调 `qcom_scm` → `SCM_SVC_MP, 0x0e, config=1`，随后读 `0xFDB10000` 并记录返回。

---

## 8. v18–v22 实测（xPU 假设被证实 + 拿到"稳定平台"）

| 版本 | 做法 | 结果 |
|---|---|---|
| v18 | 移除 v16 的 `early_initcall` 早读 | 早读**在 framebuffer console 注册之前**就读 `fdb10000` → fatal 违规 → **连横幅都不打印就复位**（这解释了 v16/v17 的"EFI stub 后无输出"；也纠正了"电量掉压"的误判）|
| v19 | 只留 `qcom_scm_set_xpu_violation(false)`（non-fatal） | **`-> 0`（TZ 认这条 SCM 命令）**；本轮 27s 读仍硬复位，但**下一轮行为就变了** ↓ |
| v20 | 试 `qcom_scm_io_readl()`（借 TZ 的手读） | **全部 `ret=-22 (EINVAL)`，连对照组 MMCC(0xfd8c4024) 也 -22** → **SCM IO 服务在零售机上不可用**，此路不通 ✗；**但**本轮读 `fdb10000` 变成 **`synchronous external abort` 的 oops（可存活）**，机器一路进桌面 ✅ |
| v21 | non-fatal + 重试 `assign_mem` / `restore_sec_cfg(18,0)` | **又 ~4s 硬复位（没活到 rootfs）** → 这两招有副作用，**永久放弃** ✗ |
| v22 | 稳定平台版：每次开机设 non-fatal；危险读推迟到 **+90s** | 待验证 |

**★★★ 核心结论（xPU 实锤）**
- `set_xpu_violation(non-fatal)` 返回 0，且**行为从"硬复位"变成"可存活的 synchronous external abort (oops)"** → **这就是 xPU 违规行为开关，假设成立** ✅
- 该设置**跨热重启保留**（v20 没再调用它，行为仍是 non-fatal）→ 但保险起见内核每次开机都设 ✓
- 由此得到**稳定调试平台**：一读不再是"死"，而是"报错（oops）" → 后续所有实验都不必再担心复位丢日志 🎉
- pstore 为空 + "无 panic 记录" → 复位是**平台级硬复位**（非内核 panic）

**下一步**
1. 在稳定平台上**系统地扫寄存器**（哪些 offset 被拒/哪些能读），据此判断"整块被拒"还是"部分"
2. 继续找**授权路径**：CAF 里 xPU/ACC 权限的配置者（TZ 启动配置 / UEFI 消费 ACPI `XPUE` / 别的 SCM 命令）——`assign_mem`/`restore_sec_cfg` 已证明不可用
3. 备选：NoC 路由（`BUSARB: APPSS_PROC→GFX3D_CFG`）那条线（GPT 认为"未投票→不响应"无公开证据，优先级降级）
4. 可选旁路实验：Windows → 热重启 → Linux（看 NoC/xPU 状态是否保留）

**教训（新增）**
- 早读探针若在 console 注册前触发 fatal 复位 → **屏上什么都没有**，极易误判成"电量/别的故障"
- SCM 安全类调用（assign_mem / restore_sec_cfg）**可能有破坏性副作用**，必须一次只试一个、并先确认有"可存活"兜底

---

## 9. 收尾（2026-09-25 02:21）：v22/v23 + "授权不在源码里"结论 + 明日计划

### v22 / v23 实测
- **v22**（non-fatal + 危险读推迟到 +90s）：`[v22] set_xpu_violation(non-fatal) -> 0` ✓，但 **90s 读 `fdb10000` 仍硬复位**（随后机器重启）✗
- **对比 v20**：v20 那次读是 **可存活 oops**，而它的 non-fatal 是**上一次开机（v19）设的**
- ⇒ **实测不对称**：`set_xpu_violation(non-fatal)` **要"下一次开机"才真正生效**（同一开机周期内设了没用）
- **v23（已刷，稳定版）**：只设 non-fatal，**扫描里去掉 `fdb10000` 危险读**（保留 MDSS/GPU 对照）→ 手机可稳定长跑
- **两步走协议（明天验证）**：开机 A（只设 non-fatal）→ 重启 → 开机 B（带读）→ 读应只 oops 不重启

### "授权"到底在哪：源码搜索结论（GPT + 本地）
**在 CAF/下游源码里找到的：**
- `scm-xpu.c`：只管 **违规行为**（fatal/non-fatal），**不管授权**
- `SCM_SVC_IO`（IO 读写）：**实测 -22（EINVAL），零售机不可用**
- `SCM_SVC_MP / RESTORE_SEC_CFG`：**实测有破坏性副作用**（硬复位）
- `qcom_scm_assign_mem()`（主线）：语义是"内存区间改归属"，对我们无用且有副作用
- CAF `msm_iommu-v1.c`：secure 处理（`__program_iommu_secure`、"set security bit override to Non-secure"、"**Set VMID to non-secure HLOS**"）→ 参考价值
- 新线索：`TZ_INFO_GET_SECURE_STATE`（SCM_SVC_INFO）——可查 TZ 安全状态
**没找到（=我们缺的）**：**"把某 MMIO 寄存器块授权给非安全世界"的调用**
⇒ 推断：**授权发生在更早环节**（TZ 启动配置 / UEFI 消费 ACPI `XPUE`），**不在 Linux 侧**

### 明日计划（**不依赖 GPT——免费额度已用完**，全部我自己能离线做）
1. **dump 手机 TZ(`sda21`,1M) + TZAPPS(`sda24`,16M)**（一次都没挖过）→ 搜 xPU/XPU/权限表/`ICB`/`GFX3D` 字符串与结构
2. **两步走验证"保护模式"**（开机 A 设 non-fatal → 重启 → 开机 B 带读）→ 拿到"可随便扫寄存器"的平台
3. 拿到保护后：**系统扫 `fdb10000` 各 offset**，判断"整块被拒"还是"部分被拒"
4. 继续找授权路径：`TZ_INFO_GET_SECURE_STATE` 诊断 + EDK2/ACPI `XPUE` 消费链 + `SCM_SVC_TZ`(0x4)/`SCM_SVC_SMMU_PROGRAM`(0x15) 命令表
5. 备选：Windows → 热重启 → Linux 旁路实验（看 TZ/NoC 状态是否保留）
6. 效率提升（可选）：大容量模式下给手机 rootfs 装 SSH 公钥 + 启用 sshd → 以后 USB gadget 直连 SSH，免去刷机循环

### 当前状态
- 手机：**v23（稳定版）**；DTB v14；`/boot` 全套 `.bak-*`；**建议不用时关机/断电池**
- 本机备份：`~/lumia-backup/`（固件 + ACPI + 内核/DTB）
- 工作区文档：本文件（9 节）+ `950XL-IOMMU-进度-20260924.md`（v11→v23）+ `给GPT的简报/海外资料清单/固件ACPI挖矿`

---

## 10. 追加（02:30）：SSH 直连尝试 + USB gadget 疑点

**已做（手机侧）**
- 生成密钥（CachyOS `~/.ssh/id_ed25519_lumia`），公钥装到手机 `/home/lumia/.ssh/authorized_keys`（700/600, uid1000）✓
- 确认手机 `sshd` 已装、`ssh.service` 已 enable、`PubkeyAuthentication yes` ✓
- 读到 gadget 脚本 `lumia-usb-gadget.sh`：**手机侧 `usb0` 固定 `10.55.0.2/24`**（宿主侧应对应 `10.55.0.1/24`）✓

**卡点**
- 手机正常启动 + 线插在 CachyOS 上，**宿主 `lsusb` 完全看不到手机**（dmesg 最后 USB 事件仍是退出大容量时的 `Lumia BootMgr ... disconnect`）
- 关键对比：**大容量模式走 bootloader 的 USB 栈**（好的）；**Linux 侧要自己把 dwc3 切到 device 角色**（PHY/角色检测/UDC bind）→ 怀疑**没切成功**（可能与"OTG/USB 角色未修"同源）
- 验证一步：读手机 `/home/lumia/usb-gadget.log`（脚本记录了每步 rc；含 `modprobe` 结果、`UDC=[...]`、bind rc）

**加入明日计划**：修 Linux 侧 USB device/OTG 角色 → 打通 SSH → 以后全流程"活系统"调试

### 10.1 USB gadget 失败根因（02:32 定位）
- DT: `snps,dwc3` 节点 **`dr_mode = "peripheral"` 已配好**（所以不是"OTG 角色"问题）；但**没有 `phys`/`phy-names`**
- 整个 DTB 里**只有 dsi-phy**，**没有任何 USB2 PHY 节点**
- 内核配置：`CONFIG_PHY_QCOM_QUSB2=m`（模块）、`CONFIG_PHY_QCOM_USB_HS=y` —— **有驱动但 DT 里没节点/没绑定 → PHY 不 probe → 控制器拉不起 D+ → 宿主看不到设备**
- 修法：①`PHY_QCOM_QUSB2` 改 `=y` ②DT 加 QUSB2 PHY 节点 + dwc3 加 `phys=<&qusb2_phy>; phy-names="usb2-phy";` ③参照 mainline `msm8996.dtsi` 的 qusb2-phy（同代）+ 下游 8994 PHY 定义
- 结论：所谓"OTG 未修"，本质是 **USB PHY 层未 bring-up**（大容量模式走 bootloader 栈，所以不受影响）

---

## 11. 剩余路线图（"还有多长"）— 2026-09-25 02:34

| # | 环节 | 难度/不确定性 | 参照物 | 粗略估计 |
|---|---|---|---|---|
| 1 | **解锁 xPU 权限**（当前这道门） | ⚠️ **最不确定**（答案可能在 TZ 固件/UEFI，或根本没有 Linux 侧入口） | 固件 ACPI `XPUE`、TZ(`sda21`)/TZAPPS(`sda24`)、SCM 命令表 | 1 天 ~ 数周；也可能要换思路（如借 TZ/驱动分层） |
| 2 | GPU IOMMU 驱动 bring-up（qcom_iommu：ctx/域 attach） | 中（结构清楚，但 8994 无上游节点） | msm8916 主线 + CAF `msm_iommu-v1.c` | 几天 |
| 3 | adreno a430 内核驱动（msm/a4xx；**不用 zap** ✅） | 中（同代有参照） | msm8996/8998 的 adreno 结构 | 几天~1 周 |
| 4 | 用户态 3D（Mesa/Freedreno **fd4 已支持 a430** ✅） | 较低（依赖内核侧先通） | Mesa freedreno | 几天 |
| 5 | 杂项稳定性（桌面 ~1 分钟卡死/BT QCA6390、USB PHY/OTG、WiFi、关机重启、电池） | 分散小坑，数量多 | 各自独立 | 分散数周 |
| 6 | 收尾（文档 / 毕业设计 / 简历素材） | 低 | 已积累大量素材 | 随时可做 |

**一句话**：**1 道"最不确定的门"（xPU 授权）+ 4~5 段"有参照的常规工程量"**。
只要第 1 步过了，后面都是有参照、可拆解、能估工期的活。
