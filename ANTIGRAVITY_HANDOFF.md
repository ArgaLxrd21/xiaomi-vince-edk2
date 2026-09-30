# ANTIGRAVITY HANDOFF — Xiaomi Redmi 5 Plus (vince) / MSM8953 EDK2 Port

## Mission

Primary goal: make EDK2/UEFI boot reliably on Xiaomi Redmi 5 Plus (`vince`, Qualcomm MSM8953), following the Renegade Project architecture and using `msm8953-mainline/linux` as the primary hardware reference.

This is a **NEW MSM8953 SoC port**, not a simple device-config port.

After EDK2 successfully boots and is stable: continue toward **Windows 10 ARM64 on Vince**. Do not jump to Windows before EDK2 actually boots.

## User/workflow

- Indonesian, casual/direct communication.
- One meaningful action at a time.
- Prefer concrete shell commands and repository inspection.
- Do not invent Qualcomm register addresses.
- Mainline MSM8953 Linux is the hardware source of truth.
- `edk2-msm` is the EDK2 framework/reference.
- Preserve source evidence and do not make the user repeat already-recorded commands.

## Project root

    /workspaces/xiaomi-vince-edk2

Contents:

    .git/
    README.md
    android-mkbootimg/
    backup/
    edk2-msm/
    msm8953-linux/

Git remote:

    https://github.com/ArgaLxrd21/xiaomi-vince-edk2

Main EDK2 tree:

    /workspaces/xiaomi-vince-edk2/edk2-msm

Main hardware source:

    /workspaces/xiaomi-vince-edk2/msm8953-linux

## Device

    Xiaomi Redmi 5 Plus
    codename: vince
    Snapdragon 625
    Qualcomm MSM8953
    RAM: 4 GB
    storage: 64 GB
    China MEE7

Runtime model:

    Qualcomm Technologies, Inc. E7 QRD SKU3

Compatible:

    qcom,msm8953-qrd-sku3
    qcom,msm8953-qrd
    qcom,msm8953
    qcom,qrd
    xiaomi,vince

Bootloader:

    MSM8953_DAISY1.0_20191118220928

Architecture:

    ARM64 / aarch64

Old Qualcomm boot chain:

    sbl1 -> aboot

No xbl/abl by-name partitions.

## Existing local reference files

    vince.dts
    vince-test.dtb
    backup/vince.dtb
    backup/msm8953-xiaomi-vince.dts
    backup/boot.img

`vince.dts` compiles successfully:

    dtc -I dts -O dtb vince.dts -o vince-test.dtb

Resulting DTB:

    330077 bytes

Runtime FDT previously measured:

    330077 bytes

Prefer the known-good `vince.dts`; do not waste time re-investigating the malformed direct DTB decompilation unless necessary.

## Renegade state

The guide supplied by the user showed:

    git clone --recursive git@github.com:edk2-porting/edk2-msm.git
    cd edk2-msm
    ./build.sh -d DEVICECODENAME

Later builds should use:

    --skip-rootfs-gen

SSH cloning failed due to missing GitHub SSH credentials. HTTPS clone succeeded.

The existing `edk2-msm` tree has no native:

    configs/devices/vince.conf
    configs/msm8953.conf
    Platform/Qualcomm/msm8953/
    Silicon/Qualcomm/msm8953/

Therefore this is a new SoC port.

Existing platforms such as msm8998, sdm660, sdm845, sm6xxx and sm8xxx are references only. Never copy their SoC addresses blindly.

## Mainline source

Repository:

    https://github.com/msm8953-mainline/linux.git

Path:

    /workspaces/xiaomi-vince-edk2/msm8953-linux

Important files:

    arch/arm64/boot/dts/qcom/msm8953.dtsi
    arch/arm64/boot/dts/qcom/msm8953-xiaomi-vince.dts
    drivers/clk/qcom/gcc-msm8953.c
    drivers/clk/qcom/clk-branch.c
    drivers/clk/qcom/clk-rcg2.c
    drivers/tty/serial/msm_serial.c
    include/dt-bindings/clock/qcom,gcc-msm8953.h

## Confirmed MSM8953 hardware

    8 x Cortex-A53
    2 clusters

    XO = 19.2 MHz
    sleep clock = 32.768 kHz

    GIC distributor = 0x0B000000
    GIC CPU interface = 0x0B002000
    APCS = 0x0B011000
    architectural/memory timer = 0x0B120000

    GCC = 0x01800000
    TLMM = 0x01000000, 142 GPIOs

    UART = 0x078AF000
    UART IRQ = GIC SPI 107

    SDHCI1 = 0x07824900
    SDHCI1 core = 0x07824000

    SDHC2 = 0x07864900
    SDHC2 core = 0x07864000

    USB/DWC3 around 0x070F8800
    MDSS = 0x01A00000
    MDP5 = 0x01A01000

    SCM compatible = qcom,scm-msm8953

## Vince UART DTS

Mainline `msm8953.dtsi`:

    uart_0: serial@78af000 {
        compatible = "qcom,msm-uartdm-v1.4", "qcom,msm-uartdm";
        reg = <0x078af000 0x200>;
        interrupts = <GIC_SPI 107 IRQ_TYPE_LEVEL_HIGH>;
        clocks = <&gcc GCC_BLSP1_UART1_APPS_CLK>,
                 <&gcc GCC_BLSP1_AHB_CLK>;
        clock-names = "core", "iface";
        status = "disabled";
    };

Thus Vince uses:

    Qualcomm UARTDM 1.4
    base = 0x078AF000
    size = 0x200
    IRQ = GIC SPI 107
    core clock = GCC_BLSP1_UART1_APPS_CLK
    iface clock = GCC_BLSP1_AHB_CLK

Linux match table confirms:

    msm-uartdm-v1.4 -> UARTDM_1P4

## UART MMIO

Linux:

    msm_write(port, val, off):
        writel_relaxed(val, port->membase + off)

    msm_read(port, off):
        readl_relaxed(port->membase + off)

So EDK2 can use direct MMIO:

    UART_BASE + register_offset

Known offsets:

    MR1  = 0x00
    MR2  = 0x04
    CSR  = 0x08
    SR   = 0x08
    TF   = 0x0C
    CR   = 0x10
    IMR  = 0x14
    IPR  = 0x18
    TFWR = 0x1C
    RFWR = 0x20
    MREG = 0x28
    NREG = 0x2C
    DREG = 0x30

UARTDM:

    DMRX   = 0x34
    DMEN   = 0x3C
    NCF_TX = 0x40
    RXFS   = 0x50
    TF     = 0x70
    RF     = 0x70

Important: CSR and SR both use offset 0x08 in the Linux definitions. Do not assume two separate physical registers.

## UART reset sequence

Linux `msm_reset()`:

    CR = RESET_RX
    CR = RESET_TX
    CR = RESET_ERR
    CR = RESET_BREAK_INT
    CR = RESET_CTS
    CR = RESET_RFR

Then:

    MR1 &= ~RX_RDY_CTL

For UARTDM:

    DMEN = 0

Preserve this ordering when implementing the EDK2 UART initialization unless source inspection proves otherwise.

## UART TX path

Linux UARTDM PIO path uses:

    UARTDM_TF = base + 0x70
    UARTDM_NCF_TX = 0x40

It polls TX status and writes up to 4 bytes at a time.

Linux wait logic checks:

    SR & TX_EMPTY

and:

    ISR & TX_READY

with microsecond delays, then resets TX-ready:

    CR = RESET_TX_READY

For EDK2 DEBUG output, use polling. DMA is unnecessary.

## UART baud table

Linux table:

    divisor code rxstale
       1     ff    31
       2     ee    16
       3     dd     8
       4     cc     6
       6     bb     6
       8     aa     6
      12     99     6
      16     88     1
      24     77     1
      32     66     1
      48     55     1
      96     44     1
     192     33     1
     384     22     1
     768     11     1
    1536     00     1

Do not choose a code without considering the selected UART clock.

## UART init behavior from Linux

`msm_startup()`:

    msm_init_clock()
    configure automatic RFR level
    request Linux DMA for UARTDM

For EDK2 DEBUG:

    no Linux DMA dependency

`msm_init_clock()`:

    dev_pm_opp_set_rate(port->uartclk)
    clk_prepare_enable(core)
    clk_prepare_enable(iface)
    msm_serial_set_mnd_regs()

For UARTDM:

    msm_serial_set_mnd_regs() returns immediately

Therefore:

**UARTDM does not use the old UART MND register setup. Its input clock is changed through the GCC clock framework.**

`msm_set_baud_rate()`:

    choose rate/divisor
    set clock rate
    write CSR baud code
    configure RX stale
    RFWR = RX watermark
    TFWR = 10
    protection enable
    reset
    enable RX/TX
    UARTDM stale configuration

## GCC MSM8953

File:

    drivers/clk/qcom/gcc-msm8953.c

GCC base:

    0x01800000

BLSP1 AHB:

    halt_reg = 0x01008
    halt_check = BRANCH_HALT_VOTED
    enable_reg = 0x45004
    enable_mask = BIT(10)

Absolute enable register:

    0x01845004

mask:

    0x400

UART1 APPS branch:

    halt_reg = 0x0203c
    halt_check = BRANCH_HALT
    enable_reg = 0x0203c
    enable_mask = BIT(0)

Absolute:

    0x0180203C

mask:

    0x1

UART1 APPS RCG:

    cmd_rcgr = 0x02044
    hid_width = 5
    mnd_width = 16

Absolute:

    CFG = 0x01802048
    M   = 0x0180204C
    N   = 0x01802050
    D   = 0x01802054

## UART clock frequency table

Mainline `ftbl_blsp_uart_apps_clk_src[]`:

    3686400   P_GPLL0_DIV2  1  144   15625
    7372800   P_GPLL0_DIV2  1  288   15625
    14745600  P_GPLL0_DIV2  1  576   15625
    16000000  P_GPLL0_DIV2  5  1     5
    19200000  P_XO          1  0     0
    24000000  P_GPLL0       1  3     100
    25000000  P_GPLL0       16 1     2
    32000000  P_GPLL0       1  1     25
    40000000  P_GPLL0       1  1     20
    46400000  P_GPLL0       1  29    500
    48000000  P_GPLL0       1  3     50
    51200000  P_GPLL0       1  8     125
    56000000  P_GPLL0       1  7     100
    58982400  P_GPLL0       1  1152  15625
    60000000  P_GPLL0       1  3     40
    64000000  P_GPLL0       1  2     25

Parent mapping:

    P_XO = hardware select 0
    P_GPLL0 = hardware select 1
    P_GPLL0_DIV2 = hardware select 4

## RCG2 details

Linux `clk-rcg2.c`:

    CFG_REG = 0x4
    CFG_SRC_DIV_SHIFT = 0
    CFG_SRC_DIV_LENGTH = 8
    CFG_SRC_SEL_SHIFT = 8
    CFG_SRC_SEL_MASK = (0x7 << 8)
    CFG_MODE_SHIFT = 12
    CFG_MODE_MASK = (0x3 << 12)
    CFG_MODE_DUAL_EDGE = (0x2 << 12)
    CFG_HW_CLK_CTRL_MASK = BIT(20)

    M_REG = 0x8
    N_REG = 0xc
    D_REG = 0x10

For UART1 RCG (`cmd_rcgr=0x02044`):

    CFG = 0x01802048
    M   = 0x0180204C
    N   = 0x01802050
    D   = 0x01802054

For MND entries, follow the Linux `clk-rcg2.c` algorithm exactly. Do not invent an alternate formula.

## GCC branch behavior

Linux `drivers/clk/qcom/clk-branch.c` shows `clk_branch2_enable()` calls `clk_branch_toggle()` which enables the register bit then waits for halt status.

Normal halt polling:

    up to 200 iterations
    udelay(1)

Thus roughly 200 us timeout.

BLSP1 AHB uses:

    BRANCH_HALT_VOTED

UART1 APPS uses:

    BRANCH_HALT

Before implementing the EDK2 branch helper, inspect the exact `clk_branch2_check_halt()` / `check_halt()` implementation to establish halt polarity and bit semantics. Do not guess.

## GPLL0 warning

The inspected MSM8953 GCC source defines GPLL0 structures but does not provide a simple explicit GPLL0 frequency configuration that can safely be assumed.

Therefore:

**Do not assume GPLL0 frequency.**

Investigate whether the UART can use the 19.2 MHz XO directly for the initial 115200 debug path before making the EDK2 implementation depend on a guessed GPLL0 rate.

## EDK2 serial-library warning

Existing `edk2-msm` commonly uses:

    QcomGeniSerialPortLib

That is GENI/QUPv3-oriented.

Vince uses Qualcomm UARTDM 1.4.

Therefore:

**Do not simply point PcdDebugUartPortBase at 0x078AF000 while retaining QcomGeniSerialPortLib.**

A dedicated MSM8953 UARTDM SerialPortLib is expected.

Inspect existing EDK2 library conventions before finalizing its path.

Possible architecture:

    Silicon/Qualcomm/msm8953/Library/Msm8953UartSerialPortLib/

but do not create the final structure blindly.

## Expected eventual platform layout

Likely, after inspecting existing conventions:

    Platform/Qualcomm/msm8953/
        msm8953.dsc
        msm8953.fdf

    Silicon/Qualcomm/msm8953/
        Include/
        Library/
            PlatformPrePiLib/
            PlatformPeiLib/
            Msm8953UartSerialPortLib/

    configs/
        devices/
            vince.conf
        msm8953.conf

Use existing platforms as structural examples only.

## Memory layout

Renegade `/proc/iomem` step has already been completed.

Important System RAM:

    0x10000000-0x849fffff
    0x86800000-0x86bfffff
    0x8ef00000-0xffffffff

Kernel areas:

    0x10080000-0x123fffff
    0x12400000-0x127fffff
    0x12800000-0x12f67fff

Use the complete stored `/proc/iomem` dump as authoritative.

## Rules to prevent mistakes

1. Never treat MSM8953 as SDM660.
2. Never copy MSM8998 GIC/register addresses.
3. Never use GENI SerialPortLib for Vince UARTDM.
4. Never guess watchdog addresses.
5. Never assume GPLL0 frequency.
6. Do not add DMA just for EDK2 DEBUG.
7. Do not throw away known-good `vince.dts`.
8. Do not make the user repeat commands already documented here.
9. Do not modify `edk2-msm` randomly before understanding DSC/FDF/config architecture.
10. Do not attempt Windows 10 before EDK2/UEFI boots.
11. When a hardware fact is not source-verified, label it unverified.
12. Prefer mainline source inspection over memory/guesswork.

## Recommended next steps

The current investigation ended at Linux `clk-branch.c`.

Next source check:

    grep -n -A100 -B20     'clk_branch2_check_halt\|check_halt'     drivers/clk/qcom/clk-branch.c

Then:

1. Finish MSM8953 GCC UART clock helper.
2. Determine a verified safe initial UART clock for 115200 8N1.
3. Implement MSM8953 UARTDM polling SerialPortLib.
4. Test DEBUG UART independently.
5. Create MSM8953 platform DSC/FDF.
6. Add `vince.conf`.
7. Build `boot-vince.img`.
8. Boot on physical Vince.
9. Debug early boot.
10. Establish stable EDK2/UEFI boot.
11. Begin Windows 10 ARM64 bring-up only after EDK2 is working.

## Long-term Windows plan

Once EDK2 is actually booting:

    EDK2/UEFI
        ↓
    storage / framebuffer / input / USB
        ↓
    ACPI/device description
        ↓
    Windows 10 ARM64 boot
        ↓
    device drivers
        ↓
    graphics / storage / USB / Wi-Fi / touchscreen / etc.

Windows 10 is the long-term objective, not the current UART debugging target.

## Core AI rule

The next AI has access to all project files.

Before claiming a register, clock, device behavior, or driver property:
- inspect the relevant repository source
- identify the exact source location
- distinguish verified facts from inference
- never replace an unknown with a confident guess

The goal is a technically defensible, buildable MSM8953 EDK2 port that can eventually lead to Windows 10 ARM64 on Vince.
