# Lumia 950 XL GPU 加速方案(2026-08-04)

> 昨晚结论"GPU = 内核级开发工作" **已推翻**。实际是 **dts 移植工作**,驱动层零改动。

## 核心发现:万事俱备,只差 dts

| 组件 | 状态 | 证据 |
|------|------|------|
| msm DRM 驱动 | ✅ A430 完整支持 | `adreno_device.c:125` `.revn=430, .name="A430"`;`a4xx_gpu.c` 有 `adreno_is_a430` 特殊时序(早期 A430 SP/TP power collapse) |
| GPU 时钟 | ✅ mmcc-msm8994.c (2620行) | `oxili_gfx3d_clk`(ID 115)、`oxili_rbbmtimer_clk`(116)、`oxilicx_ahb_clk`(117),line 2107-2146 |
| GDSC 电源域 | ✅ | `oxili_cx_gdsc`(ID 10)、`oxili_gx_gdsc`(ID 9, supply VDD_GFX),line 2364-2383 |
| 平台电源 | ✅ | `rpmpd.c:249` `DEFINE_RPMPD_PAIR(msm8994, vddcx...)`;dt-bindings 有 `MSM8994_VDDCX=0` / `MSM8994_VDDGFX=5` |
| 内核配置 | ✅ | `CONFIG_DRM_MSM=m` + DSI PHY 全系 + `CONFIG_PINCTRL_MSM8994=y` |
| 固件 | ✅ 无需 zap | a4xx 无 zap shader 代码路径(a5xx/a6xx 才要) |
| **dts GPU 节点** | ❌ **唯一缺的** | `msm8994.dtsi` 无 `gpu@fdb00000`,全 dts 树无 msm8994 GPU/mdss 节点 |
| dts mmcc 时钟节点 | ❌ 也要补 | `msm8994.dtsi` 没有 `clock-controller@fd8c0000`(qcom,mmcc-msm8994) |

## 数据来源

- 下游 GPU 节点:`LineageOS/android_kernel_huawei_angler` (cm-14.1) `arch/arm/boot/dts/qcom/msm8994-gpu.dtsi`
  - `qcom,kgsl-3d0@fdb00000`: reg `<0xfdb00000 0x40000>`, interrupt SPI 33, chipid `0x04030000` = Adreno 430 rev0
  - 时钟: `clock_mmss` 的 `clk_oxili_gfx3d_clk` / `clk_oxilicx_ahb_clk` / `clk_oxili_rbbmtimer_clk`
  - 电源管理(downstream 独有): kgsl-busmon / devbw / bimc-bwmon —— mainline **不需要**
- 格式模板:`msm8996.dtsi` `gpu@b00000`(compatible `"qcom,adreno-530.2","qcom,adreno"`,reg-names `kgsl_3d0_reg_memory`,power-domains 用 mmcc GDSC)
- mmcc 模板:`msm8996.dtsi:742` `clock-controller@8c0000`(`qcom,mmcc-msm8996`,`#clock-cells`+`#reset-cells`+`#power-domain-cells`)

## 补丁内容(dts,两段)

### 1. mmcc 时钟控制器节点(msm8994.dtsi, soc 下)

```dts
mmcc: clock-controller@fd8c0000 {
    compatible = "qcom,mmcc-msm8994";
    reg = <0x0fd8c000 0x40000>;
    #clock-cells = <1>;
    #reset-cells = <1>;
    #power-domain-cells = <1>;
};
```
> reg 大小需对照下游确认(0xfd8c0000 是 MSM8994 MMSS_CC 基址)

### 2. GPU 节点(msm8994.dtsi, soc 下)

```dts
gpu: gpu@fdb00000 {
    compatible = "qcom,adreno-430.0", "qcom,adreno";

    reg = <0x0fdb0000 0x40000>;
    reg-names = "kgsl_3d0_reg_memory";

    interrupts = <GIC_SPI 33 IRQ_TYPE_LEVEL_HIGH>;

    clocks = <&mmcc OXILI_GFX3D_CLK>,
        <&mmcc OXILICX_AHB_CLK>,
        <&mmcc OXILI_RBBMTIMER_CLK>;
    clock-names = "core", "iface", "rbbmtimer";

    power-domains = <&mmcc OXILI_GX_GDSC>;

    status = "okay";
};
```

## 风险点 / 待验证

1. **IOMMU**:msm8996 用 `iommus = <&adreno_smmu 0>`;8994 dtsi **无 arm-smmu 节点**。msm 驱动 iommu 缺失时行为待验证(可能需要补 `qcom,msm8994-smmu` 节点,下游 `msm8994-iommu.dtsi` 有定义)。若必须,加 smmu 节点后 GPU 的 iommus 引用它。
2. **mmcc reg 大小**:核对下游 `msm8994-mdss.dtsi`/clock 定义。
3. **验证方法**:编译后 dmesg 看 `msm` 驱动 probe;`ls /dev/dri/` 应出现 card0(即使无显示输出,GPU 节点在就 probe);`glxinfo`/kmscube 测试。
4. **显示管线(mdss)**:GPU 起来 ≠ 屏幕亮。要 Plasma Mobile/KWin 还得移植 `mdp@fd900000` + `dsi@fd922000` + dsi-phy + 面板(950 XL OLED)。下游 `msm8994-mdss.dtsi` 有完整定义,第二阶段做。
5. **优先级建议**:先 GPU 节点(小改,验证 /dev/dri),成功后再啃 mdss。

## 结论

从"mainline 没人实现过 msm8994 GPU"→ 实际上 **A430 驱动 + 时钟 + 电源全在 mainline 5.18**,只差 ~40 行 dts。移植成功后 950 XL 可以上 KWin Wayland / Plasma Mobile 级别的 GPU 加速。原 Arch ARM 7.0.11 dtb 线索**不需要了**。

---

## 实战进展(2026-08-04 全天攻坚实录)

### 关键里程碑

| 时间 | 突破 | 说明 |
|------|------|------|
| 12:45 | MMCC 启用 | `CONFIG_MSM_MMCC_8994=y` → msm 驱动被触发(`VRAM carveout`) |
| 13:0x | mdp5 abort 定位 | 读 MDP5_HW_VERSION(0xfd900000)external abort |
| 21:0x | memread 铁证 | 旧内核(时钟保持)0xfd900000=`0x10090000` 可读;新内核 Bus error |
| 22:49 | **root= 修复** | `CONFIG_CMDLINE="root=/dev/mmcblk0p43 acpi=no rw"` + `CMDLINE_FORCE=y` → 早期 hang 根因(CMDLINE choice 被 scripts/config 破坏) |
| 00:06 | 单 PD dtb | mdp 去掉 rpmpd VDDCX(只留 mmcc MDSS_GDSC)→ 修复早期 hang,系统稳定活着 |
| 00:19 | **19.2MHz 时钟突破** | core_clk 用 P_XO(19.2MHz)enable 成功(200MHz 用 P_GPLL0 时 -110)→ gpll0 是 -110 元凶 |
| 00:5x | 新卡点 | 时钟全开(CBCR bit0=1)但读 0xfd900000 仍 abort → **电源域/GDSC 时序** |

### 当前精确状态(00:58)

- ✅ 系统稳定活着(root= 内置 + 单 PD dtb + 19.2MHz msm.ko)
- ✅ 时钟树全通:ahb/axi enable=0,core_clk 19.2MHz(P_XO)能成功(偶发 -110)
- ✅ gpll0 PLL 正常(600MHz);`gpll0_out_mmsscc` enable=0(待使能)
- ✅ gdscr=0xa0282000 与"MDSS 可读状态"一致(GDSC 硬件正常)
- ❌ 读 MDP5_HW_VERSION 仍 abort → 嫌疑:GDSC 使能时序 / RCG 配置时序 / reset

### 下一轮实验(待做)

1. **GDSC 上电后硬件仍不响应(当前卡点)**:gdscr=0xa0282000 与 msm 禁用时可读状态完全一致,时钟门也全开,但 MDSS 域整体 Bus error(连 memread 用户态都失败)
2. **XPU/TZ 防火墙嫌疑(Lumia UEFI 特有)**:msm 禁用时可读,msm 启用后不可读;gdscr/CBCR 全部一样 → 可能 UEFI 把 MDSS 划为 secure 域
3. **MMCC 基址 0xfd8c0000 被改写嫌疑**:memread 铁证 0xc0100000→0x40100000(bit31 变 0)——下次 memread 对比 msm 禁用/启用时的值
4. **供电稳定性**:同样配置时好时坏(22:14 活到 MATE,22:44 hang)——换充电器供电/拔电池测试
5. **mdss_gdsc 对照 msm8996**:8996 有 `parent = &mmagic_mdss_gdsc.pd`(MMAGIC 父域 ALWAYS_ON),8994 没有;若 XPU 排除后仍不响应,考虑补父域/调 GDSC 时序
6. 终极目标:读到 `0x10090000` → 进入 DSI/panel 阶段

### 踩坑宝典(血泪)

- **CONFIG_CMDLINE choice 坑**:arm64 的 CMDLINE choice 只在 `CMDLINE != ""` 时生效;scripts/config 操作会破坏 choice(丢掉 FROM_BOOTLOADER)→ 内核忽略 bootloader 参数 → root= 丢失 → 早期 hang。改配置后必须检查 choice 完整性
- **早期 hang 无 journal** = 卡在挂载 rootfs 前,先查内核命令行有没有 root=
- **保护时钟(CLK_IGNORE_UNUSED/CLK_IS_CRITICAL)会导致早期 hang**(电源域状态机),不是正确路径;正确做法是 `CONFIG_CMDLINE` 内置 `clk_ignore_unused pd_ignore_unused`(grub 传参本来就失效)
- **双 PD dtb(rpmpd VDDCX)早期 hang**;单 PD(mmcc MDSS_GDSC)正常
- **msm 早期加载 = 启动早期 abort → 系统 hang**;正确做法是 msm.conf 禁用 + msm-load.service(After=multi-user.target)延迟加载
- **调试探针会自埋雷**:PRE-PM 探针在 pm_runtime(GDSC 上电)之前 readl = 在没上电硬件上读 → 自己 abort 挡住真实流程!探针必须放在正确的电源时序点之后
- **systemd service 里 `Before=sysinit.target` 会导致 ordering cycle 被删除**;只留 `After=journald` + `WantedBy=sysinit.target`
- **早期(sysinit)memread MDSS 死地址会触发 abort 干扰启动**,检测脚本只在 45 秒后读
- **dtb 备份版本混乱教训**:bak-gpu2 其实是 GPU-only 版,mdss 版是 bak-gpu3/源码树最新——操作前先反编译验证节点
- **12:45 的 Image 正常是因为混合 .o**(16.1.1 旧 .o + 16.1.0 新 .o),全量重编暴露配置问题
- journal 时间戳完全不可靠(RTC 无电池),用 `journalctl --list-boots` + 关键字定位
- 950 XL 盘插 CachyOS 设备名每次变(sda→sdb→sdc→sdd→sde),用 `lsblk` 找 43 分区;旧挂载残留会干扰,先 `umount` 全部

### 诊断工具链(全部就位)

- `memread`:静态交叉编译的 /dev/mem 读取工具(rootfs /usr/local/bin/,863K)
- `diag.sh` v7 + `lumia-diag.service`:开机自动诊断(gdscr/CBCR/clk_summary/regulator_summary/DBG)
- `boot-check.sh` v6 + `boot-check.service`:开机自检 → /home/lumia/boot-check.txt(v6 新增 MMCC 基址 0xfd8c0000 读取)
- 分步探针版 msm.ko:mdp5_dev_probe/bind/init 各步插桩(get_clk/set_rate 返回值打印)
- 多 AI 协作:GPT(电源时序理论)+ Gemini(读屏)+ Claude(读图分析)+ DeepSeek(执行/编译)
- **重要**:boot-check.txt 只有第一阶段 = 系统 45 秒内死了;有 LATE CHECK = 系统活着

### 2026-08-05 全天进展(战报)

| 时间 | 突破 | 说明 |
|------|------|------|
| 上午 | root= 修复验证 | 系统稳定跑一上午(MATE + 触控) |
| 14:4x | mmcc clocks 补丁 | gpll0_out_mmsscc 使能,core_clk -110 消失 |
| 15:31 | clk_ignore_unused 内置 | CONFIG_CMDLINE 补齐,系统完整启动到 MATE |
| 16:4x | assigned-clocks 补丁 | AHB 40MHz/AXI 150MHz 配齐,时钟树彻底全通 |
| 17:5x | msm 禁用实验 | **memread 0xfd900000 = 0x10090000 可读!凶手锁定 msm 驱动** |
| 21:1x | msm 延迟加载 | msm.conf 禁用 + msm-load.service,系统不再早期 hang |
| 21:50 | 分步探针 | get_clk/set_rate 全通,排除时钟问题 |
| 22:12 | 删 PRE-PM 探针 | 探针自埋雷,删后流程真正走到 get_sync |
| 22:2x | GDSC 上电确认 | get_sync ret=0,mdp5_enable 全 ret=0 |
| 23:00 | 当前卡点 | GDSC 上电后 MDSS 硬件仍不响应;basic.target 后偶发 hang(疑供电) |

### 2026-08-06 全天进展(收官战报) 🎉

**战果：MDSS 域锁死问题解决 → MDP5 显示控制器绑定成功 → /dev/dri 出现（显示模式落地）**

| 时间 | 突破 | 说明 |
|------|------|------|
| 21:39 | 探针 A-D 第一轮 | mdp5_init ENTER 就 abort → 锁死在更早环节（不是时钟/GDSC） |
| 21:59 | 探针 P1-P4 第二轮 | P3（mdss ioremap 后）可读、P1（devprobe）abort → 锁死在 msm_pdev_probe 窗口 |
| 22:10 | 探针 P5a/P5b 第三轮 | **P5a 可读 → of_platform_populate() → P5b abort：实锤 populate 触发锁域** |
| 22:20 | 注释 populate 对照实验 | P5b 变 read OK ✅ 因果成立 |
| 22:39 | gdscr 分析 | bit31 1→0 = **GDSC 被 power off**（不是 XPU 锁，是 genpd 电源管理） |
| 22:50 | 删 power-domains 实验 | 错误方向：RPM "tx channel not available" 刷屏 + readl 卡死（hang） |
| 23:02 | **GDSC 保活修复** | mdp5_mdss_init 末尾 `pm_runtime_get_sync`（引用计数+1，populate 时 genpd 不关 GDSC）→ **所有探针 read OK + MDP5 version v1.9** |
| 23:13 | MMPLL0 绕过 | 第三次 clk_set_rate(400MHz) 切 MMPLL0 abort → 改 300MHz（GPLL0，够 1440p60） |
| 23:24 | allow_vram_carveout | A430 无 IOMMU 拒绝（-6）→ 主线模块参数 `allow_vram_carveout=1`（modprobe.d + service 双保险） |
| 23:39 | GPU 供电坑 | 开 GPU 后重启循环（vdd/vddcx dummy regulator 无供电）→ dts 暂禁 GPU（status=disabled） |
| 23:47 | **✅ /dev/dri 出现** | 显示模式注册成功，系统稳定（0 failed），不再 abort/hang/重启 |

**最终修复三件套：**
1. `mdp5_mdss.c`：mdss pm_runtime_get_sync 保活 MDSS_GDSC（解决 populate 触发 GDSC power-off）
2. `mdp5_kms.c`：clk_set_rate 改 300MHz（绕开 MMPLL0，GPLL0 档位）
3. dts `&gpu { status = "disabled"; }` + `allow_vram_carveout=1`（GPU 待电源专项后复活）

**当前状态（23:47）：**
- ✅ MDSS 域全程可读（GDSC 保活）
- ✅ MDP5 v1.9 绑定成功，DRM 显示模式注册，/dev/dri 出现
- ✅ 系统稳定（LATE CHECK 正常，0 failed）
- ⏸ A430 GPU 禁用中（vdd/vddcx 供电未解决）

**下一课题（电源管理专场）：** PMIC/regulator（fan53555、pm8994 l13、GPU vdd/vddcx）→ RPM tx channel → 电池充电管理 → 供电稳定后 GPU status=okay 复活 → 屏幕点亮（DSI 面板节点）

**8/6 新增踩坑：**
- **挂载点叠罗汉**：多个 sdX43 同时挂 /mnt/950xl 导致配置文件写错盘（msm.conf 写进旧设备）→ 写文件前先 umount 清场 + 确认设备名
- fish shell 的 `$?`/引号展开坑 → 一律 heredoc + `bash -s`；lsblk 用 `-rno` 避免树形字符
- **allow_vram_carveout 是主线自带后门**：`module_param_named(allow_vram_carveout, bool, 0600)`（adreno_device.c）——无 IOMMU 时默认摆烂只做显示，开参数可裸奔（无内存保护，需后续 SMMU）
- GPU 无供电（dummy regulator）时强行初始化 = 系统重启循环（journal 落盘前就崩）

---

## 2026-08-08 电源管理修复（对齐 kitakami 方案）

### 诊断结论（3 处缺口）
1. **vdd_gfx 缺 `regulator-always-on`**（核心）：对照索尼 kitakami 参考实现，Lumia dtsi 的 `s2@1700` 只有电压定义，没常开 → GPU 上电缺 VDD_GFX → 8-06 重启循环根因
2. **GPU 被实验性禁用**：`&gpu { status = "disabled" }`（8-06 留的封印）
3. gdsc supply 绑定：驱动 `oxili_gx_gdsc.supply = "VDD_GFX"` + `parent = oxili_cx_gdsc`（CX/GX 绑定驱动已处理，照抄 kitakami always-on hack 即可）

### 修复内容
- dtsi：vdd_gfx 加 `regulator-name` + `regulator-always-on`；`&gpu` 改 `status = "okay"`
- 备份：`msm8994-msft-lumia-octagon.dtsi.bak-gpu-0808-2206`
- 编译验证：dtb 反编译确认 `status="okay"` + `s2@1700 regulator-always-on` ✅

### 探针版内核（3 插桩，只打状态不读寄存器）
- `gdsc.c` gdsc_enable：打 `gdsc名 supply= rsupply=OK/NULL`（VDD_GFX 是否拿到）
- `adreno_device.c` adreno_bind：打 rev + find_chipid 失败
- `a4xx_gpu.c` a4xx_gpu_init：入口打印
- 备份：三个文件 `.bak-probe-0808-2208`
- 产物：Image.gz (12.3MB, 22:09) + cityman.dtb (30KB, 22:06) + msm.ko

### 设备端验证（待执行）
- `dmesg | grep LUMIA-PROBE`
- 关键行：`gdsc_enable: oxili_gx_gdsc supply=VDD_GFX rsupply=OK` → 供电卡点击穿
- `ls /dev/dri/` 应出现 card0

---

## 2026-08-08 深夜攻坚（第二轮，23:00-23:59）

### 供电线接通 ✅（上半场）
- **mmcc 节点加 `vdd_gfx-supply = <&vdd_gfx>`**（指向 pmi8994 S2 s2@1700, 980mV, regulator-always-on）
- 探针实锤：dummy regulator 警告消失，`VDD_GFX: Bringing 770000uV into 980000uV`，`gdsc_reg rsupply=OK`
- 参考：索尼 kitakami dtsi（反编译成品 dtb 对比）GPU 节点几乎一样——单 GX GDSC、无 vdd_gfx-supply 接线、靠 always-on
- ⚠️ 注意：vdd_gfx-supply 必须加在**板级 dtsi**（msm8994-msft-lumia-octagon.dtsi）的 `&mmcc {}` 覆盖里，不能直接改公共 msm8994.dtsi（其他板子编译报错：找不到板级标签）

### 调试基建 ✅
- 探针 ×3：gdsc_reg（注册）/ gdsc_enable / adreno_bind + a4xx_gpu_init（只打状态不读寄存器，防自埋雷）
- boot-check.sh 增强：PHASE 0 立即抓 `early-dmesg.txt` + LATE CHECK 抓 LUMIA-PROBE/gdsc 日志
- **ramoops 就位**：CONFIG_PSTORE_RAM + dtb 保留区 0xf200000 1MB（record 0x20000/console 0x40000/pmsg 0x20000）
- **blacklist msm**（/etc/modprobe.d/blacklist-msm.conf）：阻止 udev 自动加载 msm（modules.alias 有 OF 别名，GPU 设备一出现 udev 就自动 modprobe → 早期崩溃，绕过了 msm-load.service 延迟加载）

### 🎯 重大定位（下半场，对照实验）
- **GPU status=okay → 必崩（45s 内硬件级复位，journal/pstore 全无）**
- **GPU status=disabled → 系统稳定**（即使 blacklist/ramoops/PHASE0 全保留）
- **结论：崩溃与 msm 驱动加载无关**（blacklist 后 16.9s 内无 msm 痕迹，系统照崩）——**GPU 节点存在本身（dts status=okay）就是崩溃源**，问题在设备树层面/udev 枚举/早期内核路径
- 证据链：early-dmesg.txt 显示系统活到 16.9s（Bluetooth 初始化），无 msm/adreno 痕迹，VDD_GFX 正常，gdsc_reg rsupply=OK，然后无 LATE CHECK = 45s 内崩

### 下一步（待做）
1. **查 GPU 节点哪个属性在早期触发崩溃**（嫌疑：`power-domains = <&mmcc OXILI_GX_GDSC>` 引用、udev 对 gpu platform device 的处理、或其他 dts 属性）
2. 思路：GPU 节点逐个属性剥离测试（先只留 compatible/reg，再逐步加回 clocks/power-domains/status）
3. 或查 udev 枚举 gpu 设备时的行为（blacklist 后设备无驱动，是否有其他处理路径）
4. 屏幕点亮（DSI 面板节点）仍在前方，等 GPU 这关过了再攻

### 文件状态（CachyOS ~/lumia-kernel）
- 内核 #15（ramoops 版）在盘上：vmlinuz + cityman.dtb（GPU disabled 稳定版 fe23ce7e）+ msm.ko
- 备份齐全：*.bak-preGPU-0808-2220、dtsi 备份等
- 盘当前状态：950 XL 里稳定运行（GPU disabled），未拔回

---

## 2026-08-14 深夜攻坚（P1-P6 实验系列，重大结论）

### 实验矩阵（本轮全部，同一内核 #15/#17 + 变体 dtb）

| 实验 | GPU 节点 | GDSC 上电 | msm | 结果 |
|------|---------|----------|-----|------|
| E1 | 最小（无clk/irq/pd） | 无 | 绑定失败(-ENXIO) | ✅ 稳定 |
| P1 | 最小（E1） | 48s 运行时探针 | 绑定失败 | ✅ 稳定 + **GDSCR 写入安全**（PWR_ON=1） |
| E2 | clk+irq（无pd） | 无 | 绑定成功 | ❌ Xorg oops（GPU 无电） |
| E3 | 完整（含pd） | 无 | 未加载 | ❌ ~20-30s 静默死 |
| E4 | 完整（含pd） | 0.86s ALWAYS_ON 全序列 | - | ❌ 极早死 |
| E5 | clk+irq | 0.86s 裸写 | 未加载 | ❌ ~20-30s 静默死 |
| P2 | clk+irq | ~18s（multi-user） | msm-load 启用 | ❌ 4/4 死 |
| P2b | clk+irq | ~18s | 无 msm 服务 | ✅ 稳定（5min+） |
| P3 | clk+irq | ~18s | msm-late(30s sleep) | ❌ sleep 中死 |
| P4 | clk+irq | ~18s | msm-late(立即) | ❌ 3 次死 |
| P5 | clk+irq | ~18s | **msm.ko 物理移除** | ✅ 稳定（5min+） |
| P6 | clk+irq | ~18s | msm-load 恢复 | ❌ modprobe 后 4s 内死（dmesg-pre 有、post 无）|

### 重大结论
1. **GDSC 寄存器（0x4024/0x4034）写入完全安全**——P1 运行时探针铁证：裸写 + 上电（PWR_ON=1）后系统稳定 6 分钟。推翻"TZ 锁死 GPU 电源"假设。
2. **power-domains 属性不是唯一触发源**——E5（无 pd）+ GDSC 写也死；P2b/P5（有 clk/irq 无 pd）稳定。
3. **死亡与 msm 加载强相关**——P6 判决：msm-load 执行 modprobe 后 4s 内死亡（dmesg-post 未落盘）。P2 4/4、P5 无 msm 稳定。
4. **但非 100% 必然**——P6 最后一次启动似乎活到桌面（可能是随机性/时序问题）。
   - ⚠️ 2026-08-14 23:15 更正：寄晴确认 P6 全程是反复重启（非电池关机）；电池已换新，**电池理论排除**。"Power key pressed short" 事件是复位循环产生的 PMIC 事件，非用户按键。
5. **稳定基座已确认**：E2 dtb + 内核 #15 + gpu-on（GDSC 上电）+ 无 msm = 长期稳定。**这是后续所有实验的安全底座。**

### 调试基建（本轮新增）
- gpu_on.ko：运行时打开 CX+GX GDSC 的模块（CachyOS ~/lumia-kernel 外编译，交叉编译）
- gpu_probe.ko：逐步寄存器写入探针（P1 用，已证明安全）
- gpu-on.service：multi-user 时 insmod gpu_on.ko（带 boot marker）
- heartbeat.service：每秒写时间戳（死亡时刻 ±1s）——注意 systemd ExecStart 内联 shell 引号坑，改脚本文件（/usr/local/bin/heartbeat.sh）解决
- 经验：EXPERIMENT.txt 标记要写绝对路径 /tmp/lp-root/boot/（shell cwd 坑）；scp -r 目标目录已存在会嵌套

### 下一步（V4 Pro 建议）
1. **P1c 寄存器扫掠**：在稳定基座上（P5 配置 + 运行时 modprobe msm 或探针模块），逐寄存器读/写 A430（0xfdb00000 块），从良性（RBBM scratch/status）到高危（CP_PROTECT_REG、RBBM_POWER_CNTL_IP、CP 微码），判定"全部写被拦"（TZ 硬锁，收尾 display-only）还是"特定寄存器被拦"（裁剪 a4xx init 翻盘）
2. A4xx 头号嫌疑：CP_PROTECT_REG_*（DRM 保护寄存器）、RBBM_POWER_CNTL_IP（SP/TP power collapse）、CP 微码加载
3. 退路（体面收尾）：&gpu status=disabled → msm 只绑 mdp5 显示 → /dev/dri/card0（KMS 显示加速）+ llvmpipe 软渲染 = 8-06 配置，可进简历
4. 注意：GMU 是 a5xx 才有，A430 无 GMU，排除相关嫌疑

## 2026-08-15 凌晨 V4 Pro 第四轮分析（P8-P10 之后）

### 关键反转
- P8 ✅：A430 除 VBIF 外**全部寄存器可写**（CP_PROTECT/POWER_CNTL_IP 均安全，SP_TP_PWR_ON observed，rc=0 稳定 10 分钟）
- P9 ❌：VBIF 跳过补丁版 msm.ko → 仍崩（补丁没机会执行）
- P10 🎯：journal 抓到死亡点 = **mdp5（显示控制器）probe 的 P2 readl fault**，不是 GPU init！modprobe rc=0，异步 probe 0.3s 内死
- 对比：GDSC 关时 mdp5 正常绑定（E2）；GDSC 上电后 mdp5 P2 readl fault；但 45s 后读 MDSS 正常（P5 boot-check）

### 机制结论（V4 Pro）
- msm8994 上 **GPU-CX 与 MDSS 共用 VDD_CX 电源轨**（各带本地 GDSC，上游同一 RPM VDDCX regulator）
- gpu_on.ko 裸写 GDSCR 绕过 genpd→RPM→regulator 链路 → 共享轨投票状态不一致 → 上电瞬态 → mdp5 probe 踩尾流读 MDSS → 外部中止
- 沉降后会自愈（45s 可读）→ 时序/沉降为主因（V4 Pro 先验 70%）

### 下一步实验（V4 Pro 排序）
1. **P12**（零成本）：无 msm 高频读 MDSS 画沉降窗口
2. **P11e**（低）：mdp5 P2 前 msleep(500)——活了就是 workaround
3. **P11d**（低）：跳过 mdp5_setup_interconnect
4. **P11**（低）：msm 延迟 45s 再 modprobe
5. **P13**（⭐ 最看好）：msm 先加载（GDSC 关，mdp5 绑定出 /dev/dri）→ 再上电 GDSC → 再起 Xorg/glxinfo——首次真正测 VBIF 跳过版 GPU init
6. **P14**（低）：gpu_on.ko 改正确上电序列（VOTABLE 语义 + 沉降）
7. **P1d**（中）：CP 微码/RB 预验证（寄存器清单已在分析文档）

---

## 📌 2026-08-15 凌晨 01:05 进度快照（从这里续）

### 当前盘上配置（P10 状态，未动）
- **vmlinuz**：内核 #15（bak-E3-0814 那份）
- **dtb**：E2 版（clk+irq，无 power-domains）
- **msm.ko**：P9 补丁版（a4xx_hw_init 跳过 A430 的 6 个 VBIF 写入），已部署在 rootfs
- **服务**：gpu-on.service（GDSC 上电，启用）｜ heartbeat.service（启用）｜ msm-run.service（200ms 快照版 modprobe msm，启用）｜ msm-load.service（已禁用改名）｜ p1c.service（已禁用）
- **工具**：/opt/p1c/（p1c.ko VBIF 跳过版 + runner）｜ /opt/msm/（msm-run.sh 快照版）｜ /root/gpu_on.ko（GDSC 上电模块）

### 已确认的事实（别再重复验证）
1. GDSC 写入安全（P1）；除 VBIF 外 A430 全部寄存器可写（P8）
2. 稳定基座 = E2 dtb + 内核#15 + gpu-on + 无 msm
3. msm 死亡点 = **mdp5 probe P2 readl**（不是 GPU init！），机制 = GPU-CX 与 MDSS 共享 VDD_CX 轨 + gpu_on 裸写绕过 genpd 导致瞬态
4. 45s 后 MDSS 可读（自愈）→ 时序/沉降为主因

### 下一步（V4 Pro 推荐顺序，明天直接做）
1. **P12**（零成本）：无 msm + 高频 memread MDSS（0xfd900000）/GPU（0xfdb00000）/MMCC（0xfd8c0000），画沉降窗口——脚本模板在 V4Pro 第四轮分析文档里
2. **P13**（⭐ 翻盘路径）：msm 先加载（GDSC 关→mdp5 正常绑定出 /dev/dri）→ 再 insmod gpu_on.ko 上电 → sleep 5 → 起 Xorg/glxinfo——首次真正测 VBIF 跳过版 GPU init（a4xx hw_init/CP 微码在首次用 GPU 时才跑）
3. **P11e**：mdp5 P2 前 msleep(500)（若 P12 确认沉降，这个就是 workaround）
4. **P14**：gpu_on.ko 改正确上电序列（VOTABLE 语义 + 沉降）
5. **P1d**：CP 微码/RB 预验证（寄存器清单在分析文档）

### 明天开场动作
- 拔盘 → 插 CachyOS → 部署 P12 脚本（或直接上 P13 配置）→ 插回手机测试
- 测试循环流程和坑清单都在本文件"2026-08-14 深夜攻坚"段落

---

## 📌 2026-08-18 凌晨 01:10 进度快照（P13 系列收官，从这里续）

### 当前盘上配置（P13c + 探针 v3 状态）
- **vmlinuz**：内核 #15（bak-E3-0814）不变
- **dtb**：E2 版（clk+irq，无 power-domains）不变
- **msm.ko**：探针 v3 版（2026-08-18 00:54 编译，1312728 字节）——mdp5_kms.c + mdp5_mdss.c + msm_drv.c 三文件探针，probe_log 写 /home/lumia/probe.log + vfs_fsync
- **服务**：gpu-on.service（启用）｜ heartbeat.service（启用）｜ p13.service（启用，GDSC 先上电 + 45s 沉降 + modprobe msm + 200ms 快照）｜ msm-run.service 已禁用（改名为 .disabled-p13，注意 wants 目录里残留文件 systemd 会警告但不影响）｜ p1c.service（禁用）
- **blacklist**：/etc/modprobe.d/blacklist-lumia.conf → blacklist msm（防 udev 自动加载，手动 modprobe 不受影响）
- **探针插桩点**（21 处）：mdss_init_enter / P3 / mdss_ioremap_done / vbif / clocks / irq / gdsc_keepalive_done / mdp5_dev_probe_enter / P1 / interconnect_of_icc_get_done / interconnect_warn_nosupport / after_setup_interconnect / P2 / after_P2_read / before_component_add / msm_drm_bind_enter / msm_drm_init_enter / after_drm_dev_alloc / after_msm_init_vram / before_component_bind_all / after_component_bind_all / mdp5_kms_init_enter / after_mdp_kms_init / after_pm_runtime_get_sync / after_write_loop / after_mdelay16 / after_pm_runtime_put / after_modeset_init + P4-bind-enter（mdp5_bind 自带）

### P13 系列实验结论（推翻 8-15 假设！）
1. **GDSC 时序/沉降假设被排除**：P13b 实测 45s 沉降后 modprobe msm 依然崩——P10 的"45s 后可读"只对 memread 成立，驱动 probe 的完整序列与 GDSC 上电时间无关
2. **insmod vs modprobe 坑**：insmod 不解析依赖（drm.ko/drm_kms_helper/drm_sched 缺失 → Unknown symbol rc=1）；必须 modprobe（自动拉依赖）。P13 第一轮 rc=1 是这个原因，不是时序
3. **崩溃点锁定**：P13c 探针证明 P1/P2（GDSC 先上电后）稳定通过 → 崩点后移 → bind 链路探针（v3）显示 msm_drm_init 正常（VRAM: daa00000->dba00000）→ **mdp5_bind 从未被调用（P4 不打印）**
4. **当前状态**：系统活着（uptime 76s+，P13C END 打印，多次稳定复现）、VRAM 正常、**但没有 card0**（/sys/class/drm 只有 version）= mdp5 component 没被 bind = KMS 显示设备没注册

### 根因候选（待验证）
- dmesg：`OF: graph: no port node found in /soc/mdss@fd900000/mdp@fd901000`
- msm 的 component match 构建（add_components_mdp）遍历 mdp 节点 of_graph endpoint 添加显示接口组件；**mdp 节点无 port → graph 匹配失败** → mdp5 不进 component match → component_bind_all 不 bind 它
- 注意：add_display_components 里其实有 `drm_of_component_match_add(..., mdp_dev->of_node)`（把 mdp 本身加进 match），但 P4 依然不打印——compare_of 匹配失败的细节待查（可能 of_node 指针不一致或时序问题）

### 下一步（方案 B，寄晴 01:08 拍板：不搞手动 bind 的 hack，走正路）
1. **摸清 950XL 显示拓扑**：mdss@fd900000 / mdp@fd901000 节点现状、DSI 控制器节点、panel 节点、port/endpoint 连接（dts 源在 CachyOS 内核树 arch/arm64/boot/dts/qcom/）
2. **补 mdp 节点 port/endpoint 定义**（符合 DT graph 规范）→ 重编 dtb 刷入 → 测试
3. 预期风险：mdp5_bind 之后的 mdp5_kms_init（mdp5_write 循环 / modeset_init）可能还有崩点——探针黑匣子继续抓
4. 备选（若 dts 修好仍不 bind）：查 component_compare_of 匹配细节（of_node 指针 vs 设备名）

### 工具链升级（可复用方法论）
- **探针黑匣子**：探针日志写 rootfs 持久文件 + vfs_fsync 立即落盘 → 崩溃/复位后拔盘可读完整轨迹（比 dmesg 快照可靠）
- 教训：探针函数要加锁（并发 probe 写文件会丢数据）；探针别放 return 后面（死代码永远不打印）；改 rootfs 必须 sync + umount 再拔盘（delayed allocation 会丢未落盘文件）
- 坑：设备名每次插拔变（lsblk -rno 找 ubuntu-rootfs）；sudo -S 的 stdin 不能和管道/heredoc 混用

---

## 📌 2026-08-20 凌晨 00:06 进度快照（v9 收官：DRM 设备注册成功！）

> 延续 08-18 快照。今晚连排三个雷，从"卡在 msm_drm_init 原因未知"推进到"**drm_dev_register 成功，只剩 msm_fbdev_init**"。

### 三个雷（按发现顺序）
1. **探针挤坏的语法 bug（v6 修复）**：探针插入时把 `if (ret) return ret;` 拆成 `if (ret) probe_log(...);` + 无条件 `return ret;` → msm_drm_init 在 msm_init_vram 后直接 return 0，bind/KMS 全不执行、日志静默——**之前所有"卡死"分析全是假象**。修复：`if (ret) { probe_log; return ret; }`
2. **幽灵 HDMI 接口（v7 修复）**：INTF 循环卡在 i=3 —— `msm8x94_config` 里 `connect[3]=INTF_HDMI`（高通参考设计残留，950XL 手机无 HDMI 硬件），写 0x6b800 寄存器 → 总线挂起。修复：`intf.base = { 0x6a000, 0x6a800, 0x6b000, 0, 0 }`（base=0 → 循环 continue）
3. **modeset_init 之后盲区（v8 探针）**：补探针区分 modeset_init 成败 / kms_init 完成 / DRM 注册

### v9 探针战报（完整通关链）
```
mdp5_init_DONE ✅ → adreno_bind ✅（GPU bind 成功！）
→ modeset_init_OK ✅ → mdp5_kms_init_DONE2 ✅
→ after_mdp5_kms_init_call ✅ → before_drm_register ✅
→ after_drm_register ✅ ← drm_dev_register 成功！/dev/dri/card0 应已创建
→ after_mode_config_reset ✅ → before_fbdev_init ✅
→（卡在 msm_fbdev_init，after_fbdev_init 未打）
```

### 当前盘上配置（v9 状态，2026-08-20 00:00）
- **vmlinuz**：内核 #15（bak-E3-0814）不变
- **dtb**：E2 版（clk+irq，无 power-domains）不变（GPU status=okay）
- **msm.ko**：v9（2026-08-20 00:00 编译，1321688 字节，md5 fc703ebd）——探针：mdp5_bind/mdp5_init 全流程 + adreno_bind + msm_drv 全流程 + modeset_init 成败 + fbdev 前后
- **mdp5_cfg.c 修改**：msm8x94_config intf.base={0x6a000,0x6a800,0x6b000,0,0}（排雷永久有效，可考虑上游 patch）
- **探针版本史**：v3 → v4（+mdp5_bind/init）→ v5（+adreno/msm_drv）→ v6（修复 return bug）→ v7（幽灵 HDMI）→ v8（+modeset/kms 区分）→ v9（+register/fbdev）
- 备份：rootfs 里 msm.ko.bak-v3 ~ .bak-v8 齐全

### 下一步（v10）
1. **拆 msm_fbdev_init**：内部加探针（drm_fb_helper_init / initial_config 前后）或先验证 card0 存在 + dmesg
2. 无 connector 时 fbdev client 的行为确认（modeset_init 因无 DSI 节点没建 encoder/connector，fbdev 初始化可能卡在无显示设备的路径）
3. **补 DSI 节点**（mdss 下 dsi@ + dsi_phy + panel + port/endpoint）——屏幕点亮的正路
4. DSI 通后：panel 驱动 + 背光 + fbcon → 屏幕出画面

### 方法论补充
- python patch 的 heredoc 坑：`"%d\n"` 里的 `\n` 会被 python 当换行符 → 用 `\\n`；断言失败先查转义
- mainline 配置表的"参考设计残留"（幽灵接口）是嵌入式移植常态雷——排雷可考虑回馈上游

### 📱 对外动态存档（2026-08-20 发布版）
#微软Lumia950xl# 【更新】显示链路攻坚：从"静默卡死"到"DRM 设备注册成功" 🔥
（正文见 2026-08-20 00:06 快照：语法 bug → 幽灵 HDMI → drm_dev_register ✅ → 卡 msm_fbdev_init）

---

## 📌 2026-08-23 15:2x v10 探针确认（重大转折：从"卡死"到"静默复位"）

> v10 给 `msm_fbdev_init` 插 9 处 `FBDEV_V10` 探针（dev_info）。部署到 rootfs 后让寄晴手动重启 950XL。

### 探针结果（dmesg-snapshot.txt）
- **msm 成功加载**，走到 mdp5_bind/init 全链：global_obj_init→ioremap→get_clk 全 0→set core_clk 200MHz→mdp5_enable 时钟全 on→read_mdp_hw_revision→**MDP5 version v1.9**→ `bound fd901000.mdp`
- **GDSC 上电成功**（P2 无 VDD_GFX 报错），比 v9 前进一大步

### 🎯 决定性一行（实锤根因）
```
[68.887] OF: graph: no port node found in /soc/mdss@fd900000/mdp@fd901000
```
- mdp5 节点下**没有 port 节点** → DRM 建不出显示拓扑（encoder/connector）→ 验证了"零 connector"假设

### ⚠️ FBDEV_V10 未出现的原因：静默复位循环
- p13.log 大量不同 boot_id、每个停在 `msm modprobe rc=0` 就跳下一个 START
- **dmesg-post.txt 从未生成**（modprobe 后脚本收尾前设备就复位）
- 复位发生在 `component_bind` 之后、`drm_dev_register`/`fbdev_init` 之前 → **kms_init/modeset 阶段**，因无 port/DSI 触发硬件级复位

### ✅ 结论
探针成功区分病因：**不是卡 fbdev_init，而是 msm 加载了但缺 DSI/port 节点 → kms 阶段静默复位**。
→ **必须补 DSI 节点（路线 B）**。已知参数：msm8994 DSI 双通道 `0xfd998000(DSI0)/0xfd9a0000(DSI1)`，20nm PHY，`msm_dsi_6g_host_ops`。给 `mdp@fd901000` 补 output port 指向 dsi0 + 加 `dsi@`/`dsi_phy` 节点。

---

## 📌 2026-08-23 下午（接续 v10 探针）DTS + C 代码层攻坚 + 面板时序定位

> 从 v10 探针确认后，今天下午一路推进，最终把"点亮屏幕"的路径彻底定位清楚。

### 1. DSI 节点补全（v10b/c/d）
- 按驱动源码确认 msm8994 DSI 参数：ctrl `0xfd998000(DSI0)/0xfd9a0000(DSI1)`，20nm PHY，`msm_dsi_6g_host_ops`
- 给 mdp@fd901000 加 output port + 补 dsi0/dsi0_phy/dsi1/dsi1_phy 节点
- 按 msm8974/msm8916 权威布局修 DSI PHY reg-names：`dsi_phy/dsi_pll/dsi_phy_regulator`（20nm cfg has_phy_regulator=true，无 has_phy_lane）
- PHY 3 regs：DSI0 `0xfd998500/0xfd998400/0xfd998780`，DSI1 `0xfd9a0500/0xfd9a0400/0xfd9a0780`
- 补 core_mmss 时钟：msm8994 mmcc 有 `MMSS_MISC_AHB_CLK`(111)，msm8996 的 dsi0 有 7 个时钟(mdp_core/byte/iface/bus/core_mmss/pixel/core)
- IRQ 编号对齐 msm8996 mdp5 约定：dsi0=`<4>`，dsi1=`<5>`（dsi_phy_get_id 用 io_start 校验）

### 2. ⚠️ 核心发现：DSI 循环依赖打断 mdp5 bind（GPT 判断验证）
- v9(无DSI): mdp5_bind✅ → drm_dev_register✅ → 卡 fbdev_init
- v10(加DSI,双向port): 卡在更早的 mdp5_dev_probe，mdp5 不 bind
- 根因：mdp↔dsi 双向 `remote-endpoint` 造成 **component 循环依赖**（`Fixing up cyclic dependency with fd998000.dsi`），拖住 `component_bind_all`
- **A 实验验证（dsi0/dsi0_phy 设 disabled）**：mdp5_bind 恢复执行 → 读到 `MDP5 version v1.9` → 实锤 DSI 循环依赖是 bind 元凶
- 但 bind 深处(modeset_init/register)仍复位，p13.log 最后 3 boot 有 START + modprobe rc=0 无 END
- 结论：DSI 卡 bind 这层解开了，bind 深处还有新复位点待查

### 3. DSI IRQ 容错（C 代码层，dsi_host.c）
- `irq_of_parse_and_map` 失败返回 0（非负数），原码只拦 `<0`，漏 0 → request_irq(0) 报 -22
- 加 `LUMIA-PROBE-DSI` 补丁：irq==0 跳过请求 + enable/disable_irq 保护，DSI probe 不硬崩
- 重编 msm.ko（1323040B）部署，IRQ -22 消失，但 FBDEV_V10 仍没到（因 mdp5 要 panel 才有输出）

### 4. 面板时序定位（最终锁定唯一来源）
- mainline MSM8994(Sony kitakami/angler/cityman)：❌ 都没做显示，靠 bootloader framebuffer
- angler pmOS(2017 zhuowei MR)：❌ 靠 fbdev 半残("Almost"),splash 都不亮
- WOA Lumia950XLPkg：❌ README 明确 "MDP is not fully implemented，借原厂 UEFI framebuffer"
- **stock Qualcomm UEFI/LK(在 FFU 里) = ✅ 唯一有真 panel init 的地方**（开机亮屏必需）

### 5. 待办（下一步，等 FFU/LK 到手）
- 解 FFU → 拆 LK/XBL/UEFI → 反编译挖 dsi/panel/mdss/samsung 的 init 序列 + 时序
- 时序喂 lmdpdg 转 mainline panel 驱动
- 或从已解锁设备直接 dump LK 分区（更稳，不依赖外站）
- FFU 获取：用户已买百度企业级下载服务(两块八,小号)拉 FFU，min 授权提示

---

## 🎯 主攻方向定为：面板点火（逆向 FFU/LK 挖面板时序）

> 2026-08-23 定调：950XL 面板点亮是长期攻坚主线。FFU 全盘 strings 扫过，确认面板 init 数据埋在下游二进制，需逆向，非快速通道。

### 攻坚路径（已验证的线索）
1. **面板时序来源**：stock Qualcomm UEFI/LK（开机亮屏必需）——已验证三路（mainline Sony/angler/cityman、angler pmOS、WOA UEFI）都无显示，只此一处有真时序
2. **FFU 到了**：`/home/wmq/ffu_extract/.../RM1085_..._retail_prod_signed.ffu`（3.0GB，港行黑色，型号对）
   - 分区表已解析：`RPM/TZ/HYP/WINSECAPP/TZAPPS/BACKUP_*/PLAT/EFIESP/MMOS/MainOS/Data`（**无独立 LK 分区**，LK 嵌在 PLAT 等 blob 内）
   - strings 扫过：显示相关字符串多是 Windows 组件名，**面板 init 数据在二进制深处，需反汇编**
3. **下一步动作**：
   - 解 FFU 里的**显示驱动 dll/sys**（在 MainOS/EFIESP/Win10 Mobile 镜像里）
   - **反汇编（IDA/Ghidra）** 找 DSI 面板初始化 cmd 序列（sleep out / display on / gamma / voltage）+ 时序
   - 或从已解锁设备直接 dump 引导分区（`/dev/sdg*` 有完整 eMMC 分区表）
   - 时序喂 lmdpdg 转 mainline panel 驱动

### 平行待办（低优先级，可穿插）
- 第二步（headless+VNC 让 950XL 先用起来）——不依赖面板数据，可先行，作为"能用"退路
- mdp5 bind 深处复位点——bind 之后、drm_dev_register 之间仍有复位待查

### 关键事实备注
- mdm5_bind 恢复：dsi 设 disabled 后读到 `MDP5 version v1.9`（DSI 循环依赖是 bind 元凶，GPT 判断验证）
- 显示链路：GPU 通 ≠ 屏幕亮，MDP5→DSI→panel 是显示输出必经链，panel 时序是最后一环

---

## 🛠️ 面板点火 · 逆向方案（2026-08-23 定稿）

> 目标：从 950XL 港行 FFU（已到手 3.0GB）提取**三星 1440p AMOLED 面板的 DSI 初始化序列 + 时序**，喂给 lmdpdg 生成 mainline panel 驱动。

### 资源现状
- FFU：`/home/wmq/ffu_extract/.../RM1085_..._retail_prod_signed.ffu`（3.0GB，港行）
- 分区表(INI)：`RPM/TZ/HYP/WINSECAPP/TZAPPS/BACKUP_*/PLAT/EFIESP/MMOS/MainOS/Data`（无独立 LK 分区，LK 嵌 PLAT/blob；显示驱动在 MainOS）
- FFU payload 是 **chunk 压缩**，需专用工具解，非裸 GPT

### 逆向路线（分阶段）

#### Phase 1：FFU → GPT 镜像
- 工具：`gitbisect/ffu2img`（Python 脚本，社区 t0x0/random 传下来的，把 FFU chunk 解成 raw img）
- 产出：完整的 GPT 磁盘镜像，能 `losetup` / `fdisk` 挂载
- 注意：截图确认可解出分区目录，python2/3 版本兼容性可能有坑

#### Phase 2：挂载/解出关键分区
- **EFIESP**（FAT）：bootloader 相关，可能含 UEFI/显示早期代码
- **MainOS**（NTFS）：Windows Mobile 系统镜像 → 里面是**显示驱动 dll/sys**
- 用 `mount -o loop,offset=xx` 或 `7z`/`ntfs-3g` 挂

#### Phase 3：定位显示驱动（目标文件）
- 在 MainOS 系统目录找显示相关驱动：`qcdx`、`display`、`dcomp`、`gdi`、`d3d` 相关 dll/sys
- Qualcomm 手机显示链通常是 **QCOM 显示驱动（kgsl/mdss 对应 Windows 侧）**
- 核心目标：**DSI panel 初始化 cmd 序列**（sleep out / display on / gamma / voltage），是一串 DCS hex

#### Phase 4：反汇编挖 panel init
- 工具：**Ghidra**（免费，推荐）或 IDA Pro
- 找驱动 dll 里的 **panel 配置表 / init 序列数据结构**：
  - `hfront-porch`/`hback-porch`/`hactive`/`vactive`/`hsync-len` 等时序常量
  - `on-command`（`0x11` sleep out、`0x29` display on、`0x35`、`0x51` 亮度）DCS 序列
- 从字符串 `dsi`/`panel`/`mdss` 交叉引用定位结构体

#### Phase 5：喂给 lmdpdg → mainline panel 驱动
- 拿到时序 + init 序列 → lmdpdg 生成 `panel-xyz.c` + `panel-xyz.dtsi`
- 接进我们已有的 dsi0 节点（用**正确的单向 graph**，避免循环依赖）

### 备选路线（更直接，可优先尝试）
- **从已解锁设备直接 dump 引导/显示分区**：950XL 现在跑着 Linux，`/dev/sdg*` 是完整 eMMC，直接从 eMMC 读 MainOS/EFIESP 分区 → 比解 FFU 还直接（绕过 chunk 解包）
- 前提：eMMC 分区能否被读取（之前挂载过 sdg43 rootfs 成功，说明可读）

### 风险与对策
- FFU chunk 解包工具 python2/3 兼容 → 备选直接读设备 eMMC
- 显示驱动符号被 strip → 靠字符串交叉引用 + 模式匹配
- panel init 可能不在驱动而在 **LK/EFI** → Phase 1 同时解 EFIESP/PLAT 分区

### 时间估算
- Phase 1-2（解包挂载）：1-2h
- Phase 3-4（定位+反汇编）：数小时到数天（核心攻坚）
- Phase 5（lmdpdg + 点亮验证）：数小时

---

## 🎯🎯 2026-08-23 深夜：Samsung 面板完整时序挖到（从 PLAT 分区）——点亮屏幕核心数据齐了！

> 突破：之前以为面板时序在 UEFI/驱动，实际在 **PLAT 分区（sda34）** 的 XML 配置里！`dsiPanel` 完整规格全部拿到。

### 面板时序（mainline panel 节点直接可填）
```
HorizontalActive     = 1440
HorizontalFrontPorch = 180
HorizontalBackPorch  = 44
VerticalActive       = 2560
VerticalFrontPorch   = 8
VerticalBackPorch    = 4
InvertVsyncPolarity  = False
InvertHsyncPolarity  = False
```

### DSI 配置
```
DSILanes             = 4
DSIColorFormat       = 36 (RGB888)
DSIRefreshRate       = 0x3C0000 (Q16.16) = 60Hz
DSITrafficMode       = 2 (video mode)
DSIVirtualId         = 0
DSIControllerMapping = 00 01
DSIHsaHseAfterVsVe   = False
DSIPhyDCDCMode       = True
DSIChannelId         = 1
DSIInitMasterTime    = 0
```

### DSIInitSequence（完整 DCS 初始化命令，从 PLAT 分区 dump）
```
15 36 80
15 3A 70
23 B0 04
05 00 00
05 00 00
23 D6 01
29 B3 14 00 00 00 00 00
29 B4 0C 00
29 B6 3A D3
29 C3 01 00 0F
29 C0 00 00 00 01 00 00
29 C1 84 61 c0 21 9D 32 96 7A 30 C2 08 00 00 00 00 10 61 84 4F 2D 41 0C 02 00 00 00 00 00 62 01 08 0A 00 01 11
29 C2 31 F0 0A 00 08 08 00 00
29 C4 70 00 00 44 00 44 44 00 44 00 00 06 03 00 00 00 44 00 44 44 00 44 00 00 01 03
29 C6 5A 06 50 06 50 00 00 00 00 00 00 00 00 00 00 00 00 13 1A 07 5A 06 50 06 50 00 00 00 00 00 00 00 00 00 00 00 00 13 1A 07
29 CB 1F 0F F0 F8 20 00 00 04 C0 00
23 CC 1E
29 D2 00 00
29 D7 84 F0 7F A8 CE 38 FC C1 18 E7 8F 1F 3C 10 FA C3 0F 04 41 00 00 00 00
29 D9 00 00 00
29 C7 02 10 15 1E 2A 38 43 53 37 3F 4C 5A 66 70 7C 09 0E 15 1E 2A 38 43 53 37 3F 4C 5A 66 70 7F
29 C8 01 00 00 00 00 FC 00 00 00 00 00 FC 00 00 00 00 00 FC 00
29 D0 55 81 BB 19 9A 4C 19 19 0C 00
29 D3 1B 33 BB AB A3 33 33 33 00 01 00 A0 F8 A0 0F 47 49 22 3B 37 72 07 3D BF 33
29 D5 06 00 00 01 35 01 35
FF 78
05 29 00     ← display on
FF 78
05 11 00     ← sleep out
```

### 数据来源
- 分区：`/dev/sda34`（PLAT / Platform 分区，8MB）
- 格式：XML 配置（`dsiPanel` 结构），直接读出的明文
- dump 位置：CachyOS `/home/wmq/plat.bin`

### ✅ 意义
**至此，点亮 950XL 屏幕的全部数据齐了**：时序（porch/active）+ DSI（lane/colorformat/refresh/traffic mode）+ **完整 DCS init 序列**。下一步：用 lmdpdg 或手写 mainline panel 驱动，把这份配置填进去。

---

## 23:10 收工备忘：面板链条战果 + 下一步判断（2026-08-23）

### 已达成（里程碑级）
- **panel 驱动绑定成功**：`panel-samsung-950xl.ko`（mipi_dsi_driver），`panel probe OK` 多次验证
- **DSI 循环依赖彻底解开**：mdp↔dsi 双向锁死 bind 的问题，靠 panel 驱动注册修复
- **`mdp5_bind: ENTER` + `mdp5_init` 跑起来**（以前从没到这）
- **`read_mdp_hw_revision` 读到 `MDP5 version v1.9`**、clk_set_rate 200MHz 实际到位 200000000
- 修复：dsi 节点补 `power-domains = <&mmcc MDSS_GDSC>`（挂点从 get_clk vsync 推到 MDP5 v1.9，有效）

### ⚠️ 关键结论：供电不是当前挂点主因（重要）
- 挂点 = **mdp5_init 后半段读硬件寄存器**（SMP/CTLM/hwpipe），是 **MDP5 自身时钟/电源域**问题，**不是面板供电**。
- 时序解析：
  1. **先**解决 mdp5_init 硬件读挂（MDP5 自身时钟/电源域）→ 才能到 drm_dev_register/card0
  2. **再**配面板 lab/ibb/vddio 真实供电 → 才能让屏发光
- 挖 regulator 映射是**第 2 步**料，当前卡在**第 1 步**。

### 供电 rail 线索（备料，第 2 步用）
- `plat.bin`（PLAT 分区 8MB）命中：`voltage/vdd/lab/ibb/ldo/vreg/rail/boost`
- 显示驱动 dump 命中：oempanel.sys（lab/ibb/bst/power）、qcdxkm8994.sys（lab/ldo/voltage/bst/power/rail）、DisplayManager.dll（lab）
- **注意**：这些主要是 PE 段名（`.text_cbstring`等）或 timing 字段（`DSITimingHSTrailValue`），**不是干净 rail→LDO 映射**。需 Ghidra 反汇编追踪（像挖 DSI 字段那样）。

### 下一步主攻（明天清醒再干）
1. **攻 mdp5_init 硬件读挂**（真正的拦路虎）：MDP5 自身时钟/电源域铺稳，可能涉真实时钟树 / PM8994 供电 rail。
2. 通了后：用 Ghidra 从 oempanel.sys/qcdxkm8994.sys 挖 lab/ibb/vddio 的 rail→LDO→电压映射，DTS 声明真实 regulator。
3. 目标：drm_dev_register → card0 → fbdev → 屏亮（Plasma Mobile/postmarketOS 终局）。

---

## 📌 2026-08-26 凌晨 00:40 收官（持久探针 + mdp5_hw_init 定位）

> 接 8-23 快照。本夜用"持久探针 + 二分法"把 mdp5_hw_init 崩溃点钉死，确认根因是竞态。

### 一、本夜推进（战果）
1. **mdp5_init 全通 🎉**：probe.log 铁证 `[mdp5_init_DONE]`（ctlm/hwpipe/hwmixer/interface 全 OK）+ `adreno_bind_chipid_ok` + `modeset_init_OK` + `mdp5_kms_init_DONE2`。→ **8-23 说的"mdp5_init 读 SMP/CTLM/hwpipe 挂"已解决**。
2. **崩溃点锁定 `mdp5_hw_init`**：msm_drm_init 里 `kms->funcs->hw_init()`（mdp5_kms.c）——`before_hw_init`✓ → `hw_init_after_runtime`✓ → **崩在访问 MDP5 寄存器区（DISP_INTF_SEL 写/寄存器）**。
3. **关键证据（决定性的）**：多次 boot（t=632/666/609/672/689）走到 `before/after_drm_register` + `before_fbdev_init` → **"屏幕亮"路径本身能到 card0/fbdev**，只是被竞态卡得**时好时坏**。→ 确认**时序/沉降竟态**，非硬锁死（V4 Pro P10/P11e 判断应验）。
4. **试修无效**：①跳过 DISP_INTF_SEL 写 → 仍崩；②`msleep(500)` 沉降 → 仍崩在 hw_init。→ **需"容错/就绪轮询"系统性改造，盲延时/跳过撞不中**。

### 二、方法论大坑（本夜填的）
- **静默复位平台上，内核探针必须写持久文件（+fsync），不能只 pr_info/dmesg**（复位即丢失）。
- 已建**持久 MHPR 宏**（写 `/home/lumia/probe.log` + fsync + ktime 时间戳），放 mdp5_kms.h，各 mdp5 文件可用。

### 三、探针链（当前部署的持久 msm.ko）
- mdp5_kms.c(4) + mdp5_smp.c(5) + mdp5_ctl.c(19) + mdp5_plane.c(36) + mdp5_encoder.c(22)
- msm_drv.c：before/after_vblank_init、after_irq_install、before/after_hw_init
- mdp5_kms.c `mdp5_hw_init`：hw_init_after_runtime / hw_init_dispintf_done / hw_init_ctlr_done（3 探针）
- 全部落 `/home/lumia/probe.log`（黑匣子，复位后仍可读）。

### 四、下一步（换打法，别盲试）
1. **MDP5 寄存器访问"等就绪/容错重试"**：给 `mdp5_read`/`mdp5_write` 包容错包装（读到 fault 就重试/轮询就绪位），把非确定性 fault 兜住。
2. **或把 `mdp5_hw_init` 挪到 MDP5 时钟/power 完全 up 后的稳定时点**。
3. **面板供电（vddio/lab/ibb→LDO→电压）**：仍是第 2 步料，从 oempanel.sys 用 Ghidra 挖（已定位 .sys 在 CachyOS `/home/wmq/display_driver/`，Ghidra 12.1.3 就位）。
4. **终局链**：mdp5_hw_init 稳住 → `drm_dev_register`/card0 → fbdev → **面板供电(第2步)** → DSI init 发面板 → **屏亮** → A430 GPU 供电/时钟/CP 微码 → GPU 加速桌面。

### 五、资源/工具就位
- 持久探针 MHPR(mdp5_kms.h)、patch 脚本：`workspace/950xl-panel/*.sh|*.py` + CachyOS `/home/wmq/*.sh`
- Ghidra 12.1.3（CachyOS `/home/wmq/ghidra_12.1.3_PUBLIC`）；oempanel.sys/qcdxkm8994.sys（CachyOS `/home/wmq/display_driver/` + `ghidra_work/`）
- plat.bin（PLAT 分区 8MB）已拉至 Debian `workspace/950xl-panel/`，面板时序 + DCS init 序列齐全
- 手机盘设备名每次变（sda/sdb/sdc），写前先 lsblk 确认 + sudo umount 残留

---

## 📌 2026-08-28 深夜：崩溃性质实锤 = 硬件级复位（用崩溃捕获器判决）

> 接 8-26 快照。本夜从"改驱动碰运气"转为"先判崩因性质"，用崩溃捕获器拿到决定性结论：**mdp5_hw_init 的崩不是内核异常，是硬件级复位（静默复位）。**

### 一、本轮实践（三连实验）
1. **探针细化（方案A）**：给 `mdp5_hw_init` 加 4 个细粒度持久探针（mdp5_kms.c）：`hw_init_after_settle`/`hw_init_pre_ctlr_reset`/`hw_init_post_ctlr_reset`/`hw_init_done`。编译成功（msm.ko 1,367,312B）部署。→ **崩点后移实锤**：上次只到 `hw_init_after_runtime`，这次完整跑完 msleep(500) 到 `hw_init_after_settle`。证明崩溃点是**异步 fault 漂移**，非某条指令。
2. **PWRHOLD（方向② 电源域连续供电）**：去掉 `mdp5_kms_init` 结尾 put_sync(692) + `mdp5_hw_init` 开头 get_sync(29)，让 MDP5 全程不断电（引用计数平衡，源码备份 `.bak-prePowerHold-*`，模块备份 `.bak-pwrhold-*`，部署哈希 `ece528...`）。→ **仍崩同一位置** → **排除"断电→再上电"假设**。
3. **崩溃捕获器（决定性 🎯）**：加 die notifier（读 ARM64 `esr_el1`/`far_el1`）+ panic notifier，都写持久 `/home/lumia/crash.log`（源码备份 `.bak-preCrash-*`，模块备份 `.bak-crash-*`，部署哈希 `1af720...`，msm.ko 1,369,032B，需补 `#include <linux/kdebug.h>` + 去重 dump_stack 声明才能过编译）。

### 二、⚡ 实战死判决（22:21，关键）
- `/home/lumia/crash.log` **不存在（空）**；probe.log 仍崩在 `hw_init_after_settle` 之后。
- **挂 die+panic 两个 notifier 都没写 log → 这不是内核能捕获的软件异常，而是硬件级复位**（快到绕过内核异常处理链）。**排除 kernel panic + 数据中止 → 实锤 hardware-level reset（watchdog 超时 / 总线或电源域 fault 触发 SoC 复位）。**

### 三、课题转型（重要定性）
- 之前一直当"软件 bug / 驱动竞态"打，现在证明是**硬件供电 / 时钟 / watchdog 层**的问题。
- 真正敌人 = **MDP5 时钟/供电在 msleep 后没真正跑起来**，或 **MDP5 的 rail / watchdog 没被正确供给**。
- 下一步（GDSC 深挖方向）：
  1. **确认 MDSS_GDSC 是否真被 enable**：mdss_gdsc 已确认定义在 mmcc-msm8994.c:2302（gdscr=0x2304），注册在数组（MDSS_GDSC 坐标），gdsc.c 走正常路径（未被 E5 GPU 旁路）。已加 **`lumia_gdsc_probe()`**（读 `0xfd8c0000+0x2304` GDSCR + `+0x2308` 状态，写 `/home/lumia/gdsc.log`），部署哈希 `8c1da88...`（msm.ko 1,370,640B）。
  2. 若 GDSC 在 msleep 后被 collapse → 找为什么（genpd/regulator 链）
  3. 若 GDSC 没被 enable → 查 power-domain 绑定 / mmcc 供电

### 四、⏸ 待明日实测（手机没电，已冻结）
手机深挖之前需先测 **GDSC 探针版 msm.ko**（哈希 `8c1da88...`，已在 rootfs，gdsc.log 已清空）。插机崩一次后读 `/home/lumia/gdsc.log`：
- **GDSCR=0x0**（或 SW_COLLAPSE 位=1）→ 电源域 collapse 实锤 → 转入 genpd/regulator 全链条排查
- **GDSCR 非 0（on 状态）** → 电源域活着 → 往 watchdog / 总线 fault 方向查

### 五、方法论沉淀（本夜新增）
- **崩溃捕获器是"软硬判官"**：在静默复位平台上，挂 die+panic notifier 写持久文件——**不写 log = 硬件级复位（绕过内核）；写了 = 软件异常（能精确定位）**。这比 ramoops 更可靠（ramoops 只在软件 panic/oops 时落盘，硬件复位照样空）。
- **bug 定性先行，别盲目改代码**：今晚最大的价值不是"修好了"（没修好），而是**把课题从软件层彻底划到硬件层**，避免了继续在驱动里瞎撞。
- **gdsc.c / mmcc-msm8994.c 是 builtin（CONFIG_QCOM_GDSC=y / CONFIG_MSM_MMCC_8994=y）**，改它们需重编整个内核 Image，成本高——优先用模块（msm.ko）内探针读寄存器验证，不动 builtin。
- 源码备份链（本夜）：`.bak-preFINE-*` → `.bak-prePowerHold-*` → `.bak-preCrash-*` → `.bak-preGDSCprobe-*`。
