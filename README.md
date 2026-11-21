# AXI TPG + Frame Buffer Write Example — PYNQ-Z1

An example of using the **Xilinx Video Test Pattern Generator (`v_tpg`)** and
**Video Frame Buffer Write (`v_frmbuf_wr`)** IP cores on a PYNQ-Z1 (Zynq-7020).
The TPG produces a test-pattern video stream, the frame-buffer writer stores it
in DDR, and the result is exposed to Linux as a standard **V4L2** capture device
(`/dev/video0`) that can be streamed with GStreamer.

A C++ application (`tpg-test`) changes the pattern, motion speed and box size at
runtime using the on-board switches/buttons.

---

## Overview

| Item         | Value                                                     |
|--------------|-----------------------------------------------------------|
| Board        | PYNQ-Z1 (Zynq-7020, `xc7z020clg400-1`)                    |
| IP cores     | `v_tpg` @ **0x43C0_0000**, `v_frmbuf_wr` @ **0x43C1_0000** |
| GPIO         | `axi_gpio` @ **0x4120_0000** (switches/buttons)           |
| Data flow    | `v_tpg` → AXI-Stream → `v_frmbuf_wr` → DDR (HP0)          |
| Linux        | `/dev/video0` (V4L2), `/dev/v4l-subdev0`, `/dev/media0`   |
| App          | `tpg-test` (C++20)                                        |
| Toolchain    | Vivado **2025.2**, Yocto **Scarthgap** (kas)              |

## Setup environment

- Vivado 2025.2
- Yocto Scarthgap + [kas](https://kas.readthedocs.io/)
- PYNQ-Z1 board
- Ubuntu 22.04 (or a similar Linux host)

## Hardware design

### Block design (`Vivado/scripts/pynq_z1_bd.tcl`)

The block design contains:

- `processing_system7` — the Zynq ARM Cortex-A9 PS (HP0 used for the frame
  buffer DMA).
- `v_tpg_0` — Video Test Pattern Generator (base address `0x43C0_0000`).
- `v_frmbuf_wr_0` — Video Frame Buffer Write (base address `0x43C1_0000`), which
  writes the incoming AXI-Stream video to DDR.
- `axi_gpio_0` — reads the on-board switches/buttons for the runtime controls
  (base address `0x4120_0000`).
- `axi_data_fifo` — FIFO between `v_frmbuf_wr` and the HP0 DDR port.

Data flow:

```
v_tpg ──(AXI-Stream video)──▶ v_frmbuf_wr ──(m_axi_mm_video)──▶ DDR (HP0)
```

### Address map

| IP            | Base address |
|---------------|--------------|
| axi_gpio_0    | `0x4120_0000`|
| v_tpg_0       | `0x43C0_0000`|
| v_frmbuf_wr_0 | `0x43C1_0000`|

## Linux / device tree & kernel

`system-user.dtsi` wires the pipeline up as a V4L2 capture device:

- `&tpg_top_v_tpg_0` — enables the TPG and declares its video output port
  (`xlnx,video-format`, `xlnx,video-width`).
- `video_cap_tpg` — a `xlnx,video` node that captures from the frame-buffer
  writer via DMA (`dmas = <&tpg_top_v_frmbuf_wr_0 1>`).

The kernel config (`linux-xlnx/xlnx-video.cfg`) enables the Xilinx video
subsystem drivers:

```
CONFIG_VIDEO_XILINX=y
CONFIG_XILINX_FRMBUF=y
CONFIG_VIDEO_XILINX_TPG=y
CONFIG_VIDEO_V4L2=y
```

This results in the following device nodes:

| Node               | Purpose                   |
|--------------------|---------------------------|
| `/dev/video0`      | V4L2 capture device       |
| `/dev/v4l-subdev0` | TPG sub-device (controls) |
| `/dev/media0`      | Media controller          |

## Application (`apps/tpg-test`)

A C++20 app (`tpg-test` binary) using V4L2 + libgpiod:

- `TpgDriver/` — configures the TPG through V4L2 controls (resolution, pixel
  format, test pattern, foreground pattern, motion speed, box size).
- `Gpio/` — libgpiod wrapper for the switches/buttons.
- `Utilities/` — RAII file-handle helper.
- `main.cpp` — reads the buttons/switches and applies changes at runtime.

Runtime controls:

| Control | Action                   |
|---------|--------------------------|
| SW1     | Increase motion speed    |
| SW2     | Decrease motion speed    |
| BTN1    | Cycle background pattern |
| BTN2    | Cycle foreground pattern |
| BTN3    | Increase box size        |
| BTN4    | Decrease box size        |

The binary is installed to `/home/root/tpg-test`.

## Build

### 1. Hardware — Vivado

```bash
source <Vivado_install>/settings64.sh
cd Vivado
make
```

`make` runs the Tcl scripts to create the project, build the bitstream, and
export `pynq_z1_top.xsa` (plus `pynq_z1_top.bit`) at the `Vivado/` root.

### 2. Linux image — Yocto (kas)

Prepare the host for Yocto:

- [Preparing the build host](https://docs.yoctoproject.org/singleindex.html#preparing-the-build-host)
- [System requirements (Scarthgap)](https://docs.yoctoproject.org/scarthgap/ref-manual/system-requirements.html#system-requirements)

Install `kas`:

```bash
pip install kas
```

Build:

```bash
cd yocto
make
```

`make` copies `../Vivado/pynq_z1_top.xsa` into the HDF recipe and runs
`kas build build.yaml` (machine `zynq-generic-7z020`, target `image-pynq-z1`).
The image includes GStreamer + V4L2 tools + the `tpg-driver` app. The SD-card
image is the `.wic` file under
`yocto/build/tmp-glibc/deploy/images/zynq-generic-7z020/`.

### 3. Flash & run

1. Flash the `.wic` image to a microSD card (e.g. with Balena Etcher).
2. Insert the card, connect the USB UART (115200 8N1), and power the board.
3. Log in as `root` and run the test app:

```bash
/home/root/tpg-test
```

Then stream the video to a host PC (see below).

## V4L2 & GStreamer usage

### Inspect the pipeline

```bash
media-ctl -p -d /dev/media0
ls /dev/ | grep -E 'media|video|v4l|xlnx'
v4l2-ctl -d /dev/v4l-subdev0 --list-ctrls
v4l2-ctl -d /dev/v4l-subdev0 --list-ctrls-menus
v4l2-ctl -d /dev/video0 --list-formats
```

### Configure a format (example: RGB 640×480)

```bash
media-ctl -d /dev/media0 -V '"43c00000.v_tpg":0[fmt:RGB888_1X24/640x480 field:none]'
v4l2-ctl -d /dev/video0 --set-fmt-video width=640,height=480,pixelformat=RGB3
v4l2-ctl -d /dev/video0 --get-fmt-video
```

### Choose the test pattern

```bash
v4l2-ctl -d /dev/v4l-subdev0 --set-ctrl test_pattern=11
v4l2-ctl -d /dev/v4l-subdev0 --set-ctrl test_pattern_foreground_pattern=1
```

### Stream to a host PC over UDP

**Sender (on the board) — RGB 640×480:**

```bash
gst-launch-1.0 -v v4l2src device=/dev/video0 io-mode=dmabuf do-timestamp=true \
  ! video/x-raw,format=RGB,width=640,height=480,framerate=60/1 \
  ! queue max-size-buffers=0 ! rtpvrawpay pt=96 mtu=1400 \
  ! udpsink host=192.168.1.20 port=5004 sync=false buffer-size=524288
```

**Receiver (on the host):**

```bash
gst-launch-1.0 udpsrc port=5004 \
  caps="application/x-rtp, media=(string)video, clock-rate=(int)90000, encoding-name=(string)RAW, sampling=(string)RGB, depth=(string)8, width=(string)640, height=(string)480, colorimetry=(string)BT601-5, payload=(int)96" \
  ! rtpvrawdepay ! videoconvert ! queue ! fpsdisplaysink text-overlay=true sync=true
```

**Sender — RGB 1280×720:**

```bash
media-ctl -d /dev/media0 -V '"43c00000.v_tpg":0[fmt:RGB888_1X24/1280x720 field:none]'
v4l2-ctl -d /dev/video0 --set-fmt-video width=1280,height=720,pixelformat=RGB3
v4l2-ctl -d /dev/v4l-subdev0 --set-ctrl test_pattern=16

gst-launch-1.0 -v v4l2src device=/dev/video0 io-mode=dmabuf do-timestamp=true \
  ! video/x-raw,format=RGB,width=1280,height=720,framerate=60/1 \
  ! queue max-size-buffers=0 ! rtpvrawpay pt=96 mtu=1400 \
  ! udpsink host=192.168.1.20 port=5004 sync=false buffer-size=524288
```

**Sender — YUV 4:2:2 (UYVY) 640×480:**

```bash
gst-launch-1.0 -v v4l2src device=/dev/video0 io-mode=dmabuf do-timestamp=true \
  ! video/x-raw,format=UYVY,width=640,height=480,framerate=60/1 \
  ! queue max-size-buffers=32 leaky=2 ! rtpvrawpay pt=96 mtu=1500 \
  ! udpsink host=192.168.1.20 port=5004 sync=true
```

**Receiver — YUV 4:2:2 (UYVY):**

```bash
gst-launch-1.0 udpsrc port=5004 \
  caps="application/x-rtp,media=(string)video,clock-rate=(int)90000,encoding-name=(string)RAW,sampling=(string)YCbCr-4:2:2,depth=(string)8,width=(string)640,height=(string)480,layout=(string)interleaved,payload=(int)96" \
  ! rtpjitterbuffer latency=100 ! rtpvrawdepay ! videoconvert \
  ! fpsdisplaysink text-overlay=true sync=true
```

> Replace `192.168.1.20` with the host PC's IP address.

## Repository layout

```
.
├── Vivado/                  # Hardware: block design, VHDL top, constraints
│   ├── scripts/             #   create_project.tcl, pynq_z1_bd.tcl, build_bitstream.tcl
│   └── src/                 #   pynq_z1_top.vhd, pynq_z1.xdc
├── yocto/                   # Linux image: build.yaml + meta-user layer
│   └── meta-user/
│       ├── recipes-bsp/     #   device-tree, u-boot
│       ├── recipes-kernel/  #   kernel config (xlnx-video.cfg)
│       └── recipes-apps/    #   tpg-driver
└── apps/
    └── tpg-test/            # C++20 V4L2 TPG controller
```

## References

- [Video Test Pattern Generator — Product Guide (PG103)](https://docs.amd.com/r/en-US/pg103-v-tpg)
- [Video Frame Buffer Read/Write — Product Guide (PG278)](https://docs.amd.com/r/en-US/pg278-v-frmbuf)
- [PYNQ-Z1 Reference Manual](https://digilent.com/reference/programmable-logic/pynq-z1/reference-manual)
- [Xilinx V4L2 Video Pipeline](https://xilinx-wiki.atlassian.net/wiki/spaces/A/pages/18842379/V4L2)
- [GStreamer](https://gstreamer.freedesktop.org/)
