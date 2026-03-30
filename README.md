# STM32MP157F-DK2 LVGL Gauge Dashboard

This project runs an LVGL-based gauge dashboard on the **STM32MP157F-DK2** board using **DRM** for display output. It also documents the workflow for **UART**, **CAN**, **device tree modification**, and **OpenSTLinux build/deploy** steps.

## Features

- LVGL graphical dashboard.
- DRM display backend for Linux framebuffer rendering.
- UART access on STM32MP157F-DK2.
- CAN enablement through device tree.
- OpenSTLinux SDK-based cross compilation.
- Kernel and DTB rebuild workflow.

## Hardware Used

- Board: STM32MP157F-DK2
- Display: onboard DRM display
- UART: `/dev/ttySTM2`
- CAN: `m_can1`
- Host PC: Ubuntu Linux
- SDK: OpenSTLinux cross toolchain

## Project Structure

```text
project/
├── main.c
├── Makefile
├── lvgl/
├── lv_drivers/
└── README.md
```

## SDK Setup

Before building, source the OpenSTLinux SDK:

```bash
source /opt/st/stm32mp1/5.0.8-openstlinux-6.6-yocto-scarthgap-mpu-v25.06.11/environment-setup-cortexa7t2hf-neon-vfpv4-ostl-linux-gnueabi
```

Check the compiler:

```bash
$CC --version
```

## Build the Application

### Using Makefile

```bash
make clean
make -j$(nproc)

scp XXXXFile... root@192.168.2.1:/usr/local/
```

### Manual Compile Command

```bash
$CC -O3 main.c $(find lvgl/src -name "*.c") lv_drivers/display/drm.c \
-I./ -I./lvgl/ -I./lv_drivers/ \
-I$SDKTARGETSYSROOT/usr/include/libdrm \
-DLV_CONF_INCLUDE_SIMPLE \
-DLV_COLOR_DEPTH=16 \
-ldrm -lpthread -lm -lrt -o gauge_app
```

## Run on Target Board

Copy the application to the board:

```bash
scp gauge_app root@192.168.2.198:/usr/local/
```

On the board:

```bash
chmod +x /usr/local/bin/gauge_app
/usr/local/bin/gauge_app
```

## Display Setup

Before running the GUI application, stop Weston and release DRM:

```bash
systemctl stop weston psplash 2>/dev/null
fuser -k /dev/dri/card0
chmod 666 /dev/dri/card0
```

Then start the app:

```bash
/usr/local/bin/gauge_app
```

## UART Test

List UART ports:

```bash
ls /dev/ttySTM*
```

Set permission:

```bash
chmod 666 /dev/ttySTM2
```

Read UART output:

```bash
cat /dev/ttySTM2
```

Send data from another terminal:

```bash
echo "Hello UART" > /dev/ttySTM2
```

## CAN Enablement

CAN is enabled in the board device tree file:

```text
stm32mp157f-dk2.dts
```

Example CAN configuration:

```dts
&uart7 {
    status = "okay";
};

&m4_m_can1 {
    status = "disabled";
};

&m_can1 {
    pinctrl-names = "default", "sleep";
    pinctrl-0 = <&fdcan1_pins_mx>;
    pinctrl-1 = <&fdcan1_sleep_pins_mx>;
    status = "okay";

    clocks = <&scmi_clk CK_SCMI_HSE>, <&rcc FDCAN_K>;
    clock-names = "hclk", "cclk";
};
```

## Editing Device Tree Files

Edit the board DTS file:

```bash
gedit /media/linux/5031ccbd-a675-4b86-b5ad-d83d0a73694d/stm32mp1/build-openstlinuxweston-stm32mp15-disco/tmp-glibc/work-shared/stm32mp15-disco/kernel-source/arch/arm/boot/dts/stm32mp157f-dk2.dts
```

If pinmux changes are needed, edit:

```bash
gedit /media/linux/5031ccbd-a675-4b86-b5ad-d83d0a73694d/stm32mp1/build-openstlinuxweston-stm32mp15-disco/tmp-glibc/work-shared/stm32mp15-disco/kernel-source/arch/arm/boot/dts/stm32mp15-pinctrl.dtsi
```

## Rebuild Kernel and DTB

Source the build environment:

```bash
cd /media/linux/5031ccbd-a675-4b86-b5ad-d83d0a73694d/stm32mp1
source layers/meta-st/scripts/envsetup.sh
```

Set locale:

```bash
export LC_ALL=en_US.UTF-8
export LANG=en_US.UTF-8
export LANGUAGE=en_US.UTF-8
```

Build the kernel:

```bash
bitbake virtual/kernel -c menuconfig
bitbake virtual/kernel -c clean
bitbake virtual/kernel -c compile -f
bitbake virtual/kernel -c deploy
```

## Deploy DTB to Board

Copy the rebuilt DTB to the board:

```bash
scp stm32mp157f-dk2.dtb root@192.168.2.198:/boot/stm32mp157f-dk2.dtb
```

## Flash SD Card Image

Write the full OpenSTLinux image to the SD card:

```bash
sudo umount /dev/sdx* 2>/dev/null
sudo dd if=FlashLayout_sdcard_stm32mp157f-dk2-optee.raw of=/dev/sdx bs=8M conv=fdatasync status=progress
sync
```

## SSH Host Key Cleanup

If the board IP changes:

```bash
ssh-keygen -f "/home/linux/.ssh/known_hosts" -R "192.168.2.214"
```

## Verify CAN

After booting the updated DTB:

```bash
dmesg | grep -i can
ip link show type can
```

## Notes

- Always modify `.dts` or `.dtsi`, then rebuild the `.dtb`.
- Do not edit `.dtb` directly.
- `python3` must be version 3.8 or newer for BitBake.
- Ensure the board IP is correct before using `scp` or `ssh`.

## License

Add your project license here.
