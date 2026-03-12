## Ensure dependencies

```bash
pacman -S base-devel \
wget \
xmlto kmod inetutils bc libelf git cpio perl tar xz \
aarch64-linux-gnu-gcc aarch64-linux-gnu-binutils \
meson ninja
```

## Clone i.MX Linux kernel

HTTPS:

```bash
git clone https://github.com/nxp-imx/linux-imx.git
```

SSH:

```bash
git clone git@github.com:nxp-imx/linux-imx.git
```

```bash
cd linux-imx
```

## Copy IMX477 driver into kernel

The driver exists in the Raspberry Pi kernel:

* [https://github.com/raspberrypi/linux](https://github.com/raspberrypi/linux)
* [https://raw.githubusercontent.com/raspberrypi/linux/refs/heads/rpi-6.12.y/drivers/media/i2c/imx477.c](https://raw.githubusercontent.com/raspberrypi/linux/refs/heads/rpi-6.12.y/drivers/media/i2c/imx477.c)

Download it into the `linux-imx` kernel tree:

```bash
cd drivers/media/i2c
wget https://raw.githubusercontent.com/raspberrypi/linux/refs/heads/rpi-6.12.y/drivers/media/i2c/imx477.c
```

Verify that it exists:

```bash
ls | grep imx477
```

Expected output:

```
imx477.c
```

## Edit the driver

Add the following include:

```c
#include <linux/media-bus-format.h>
```

Add this guard:

```c
#ifndef MEDIA_BUS_FMT_SENSOR_DATA
#define MEDIA_BUS_FMT_SENSOR_DATA MEDIA_BUS_FMT_META_14
#endif
```

Reference:
[https://www.kernel.org/doc/html/latest/userspace-api/media/v4l/subdev-formats.html](https://www.kernel.org/doc/html/latest/userspace-api/media/v4l/subdev-formats.html)

## Edit Kconfig and Makefile

Open:

```
drivers/media/i2c/Kconfig
```

Add an entry for the IMX477 driver. The Raspberry Pi kernel entry can be reused:

[https://github.com/raspberrypi/linux/blob/rpi-6.12.y/drivers/media/i2c/Kconfig](https://github.com/raspberrypi/linux/blob/rpi-6.12.y/drivers/media/i2c/Kconfig)

```text
config VIDEO_IMX477
	tristate "Sony IMX477 sensor support"
	depends on I2C && VIDEO_DEV
	select V4L2_CCI_I2C
	select VIDEO_V4L2_SUBDEV_API
	select MEDIA_CONTROLLER
	select V4L2_FWNODE
	help
	  This is a Video4Linux2 sensor driver for the Sony
	  IMX477 camera. Also supports the Sony IMX378.

	  To compile this driver as a module, choose M here: the
	  module will be called imx477.
```

Open:

```
drivers/media/i2c/Makefile
```

Add:

```make
obj-$(CONFIG_VIDEO_IMX477) += imx477.o
```

## Configure kernel

(Optional reset to default configuration)

```bash
make ARCH=arm64 imx_v8_defconfig
```

Or maybe the generic one ? (if it is generic at all : `imx`

Or this one? : https://github.com/RobertCNelson/u-boot/blob/master/configs/imx8mp_evk_defconfig 

Open kernel configuration:

```bash
make ARCH=arm64 menuconfig
```

### Enable RKISP1 driver

Navigate to:

```
Device Drivers
  → Multimedia support
    → Media drivers
      → Media platform devices
```

Enable:

```
Rockchip Image Signal Processing v1 Unit driver
```

Set as module `<M>` or built-in `<*>`.

Related platform drivers may also appear here (e.g. NXP media devices such as i.MX ISP, PXP, CSI receiver).

Ensure the following are enabled:

```
CONFIG_MEDIA_CONTROLLER
CONFIG_V4L2_MEM2MEM_DEV (or CONFIG_V4L2_MEM2MEM_DRIVERS)
CONFIG_VIDEO_DEV
```

Enable any platform-specific drivers required for i.MX8MP camera or CSI.

### Enable the IMX477 sensor driver

Navigate to:

```
Device Drivers
  → Multimedia support
    → Media ancillary drivers
      → Camera sensor devices
```

Enable the IMX477 driver.

Save the configuration.

Verify configuration options:

```bash
grep CONFIG_VIDEO .config
```

Example expected values (somethink like that):

```
CONFIG_VIDEO_ROCKCHIP_ISP1=y
CONFIG_MEDIA_CONTROLLER=y
CONFIG_V4L2_MEM2MEM_DEV=m
CONFIG_VIDEO_DEV=y
CONFIG_VIDEO_IMX477=y
```

## Compile the kernel

Ensure build dependencies are installed (gcc/clang, make, bison, flex, openssl-dev, elfutils, etc.).

We are goiung to compile the kernel image, modules (if enabled as `<M>`), and device tree binary files (dtbs).

### Compile on the board

```bash
make -j"$(nproc)" Image modules dtbs
```
This will take forever

### Cross-compile on a PC

```bash
make -j"$(nproc)" Image modules dtbs ARCH=arm64 CROSS_COMPILE=aarch64-linux-gnu-
```
This won't take forever

If there are no errors, it should be good

If driver is included as a module (M), you can search for a `imx477.ko` file
