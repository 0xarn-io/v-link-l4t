# Project architecture and verification context

Filled 2026-10-03 from a full read of the repository at commit `aacb56f` (branch `l4t-r36`).
Facts read from the code are stated plainly. Items marked *(unverified)* come from general
Jetson/Linux knowledge or inference and have not been checked against hardware, datasheets,
or the L4T kernel sources. Read the relevant section before changing a subsystem.

## Purpose and boundaries

Videtronic **V-Link**: Linux kernel drivers and device-tree overlays that let an NVIDIA Jetson
(Jetson Linux / L4T R36) use a standard MIPI CSI-2 camera module over a GMSL2 coax link of up
to 15 m. The camera plugs into a **serializer** board (Analog Devices MAX96717); the Jetson
CSI connector takes a **deserializer** board (MAX96714). The camera keeps its own stock L4T
sensor driver; V-Link only adds the two bridges in the middle.

```
 camera (IMX219 / IMX477)  --CSI-2-->  MAX96717 ser  ==GMSL2 coax==>  MAX96714 deser  --CSI-2-->  Jetson NVCSI -> VI
 sensor driver (stock L4T)            v-link-ser.ko                   v-link-deser.ko             tegra-capture-vi
 I2C 0x10 / 0x1a                      I2C 0x42                        I2C 0x4c
```

| Path | Responsibility |
|---|---|
| `v-link-ser/v-link-ser.c` | MAX96717 serializer driver (`videtronic,v-link-ser`) |
| `v-link-deser/v-link-deser.c` | MAX96714 deserializer driver (`videtronic,v-link-deser`) |
| `v-link-*/Makefile` | Out-of-tree kbuild wrappers (identical except `obj-m`) |
| `overlays/orin_nano_super/` | Single-camera overlays for the Orin Nano / Orin NX devkit carrier (p3768), CAM0 or CAM1, IMX219 or IMX477 |
| `overlays/agx_orin/` | Six-camera IMX219 overlay for the AGX Orin devkit (p3737) via Videtronic's camera interposer |
| `scripts/boost_clocks.sh` | Pins VI/ISP/NVCSI/EMC clocks to max via BPMP debugfs (README troubleshooting for black preview) |
| `README.md` | End-user install guide |
| `.agents/`, `.grok/`, `AGENTS.md`, `CLAUDE.md` | Agent instruction bundle; not part of the product |

Out of scope / not in this repo: the sensor drivers (`sony,imx219` = NVIDIA `nv_imx219`,
`ridgerun,imx477` = NVIDIA `nv_imx477`, both shipped with L4T *(unverified)*), the base
device tree, the GMSL link configuration itself (see "Decisions").

## Existing documentation

- `README.md`: user quickstart (build, install, apply overlay, troubleshooting).
- No datasheets, schematics, or register maps in the repo. Register addresses and the
  "fixed values based on HW design" in the drivers can only be checked against the ADI
  MAX96717 / MAX96714 user guides and Videtronic's board schematics, which are not available here.
- Upstream: https://github.com/Videtronic/v-link-l4t (one branch per L4T release, e.g. `l4t-r36`).
  Sister repo for Raspberry Pi: `Videtronic/v-link-rpi`.

## Toolchain and supported environments

- Target: Jetson Linux R36 (JetPack 6). README sample shows R36.4.3, `KERNEL_VARIANT: oot`.
  R36 runs Linux 5.15 on Ubuntu 22.04, aarch64 *(unverified for this repo, but the code's
  kernel API use matches 5.15: `probe_new`, `int remove()`, 4-argument `i2c_mux_add_adapter`,
  `void` gpio `.set`; all of these changed in kernels 6.1-6.9, so a newer kernel will not build it)*.
- Language: C, Linux kernel style (but 2-space indent mixed with tabs; see ISSUES).
- Build deps: `build-essential` and the running kernel's headers at `/lib/modules/$(uname -r)/build`.
- Overlays: `dtc` (device-tree-compiler) with `-@` for symbols.
- Owner workflow: code is edited on Windows (no build tools there); builds and tests are run
  later from the owner's dual-boot Ubuntu and on the Jetson. In-session checks are desk-checks.

## Build and verification commands

Run on the Jetson (native) or an equivalent environment with the target kernel's headers.
None of these were run while writing this document.

| Step | Command (working dir) | Effect |
|---|---|---|
| Build | `make` (`v-link-ser/`, then `v-link-deser/`) | Produces `v-link-ser.ko` / `v-link-deser.ko`; `KERNEL_DIR=...` overrides the header path |
| Install | `sudo make install` (each dir) | `modules_install` into `/lib/modules/$(uname -r)/updates` *(default kbuild path, unverified)* and runs `depmod -a`. Modules autoload from the DT `compatible` alias |
| Clean | `make clean` | kbuild clean |
| Uninstall | `make uninstall` | Calls kbuild target `modules_remove`, which is not believed to exist (ISSUES V-008) |
| Build overlay | `dtc -I dts -O dtb -@ X.dts -o X.dtbo` | Rebuild the checked-in `.dtbo` |
| Check overlay is seen | `sudo /opt/nvidia/jetson-io/config-by-hardware.py -l` | Lists the `overlay-name` under its header |
| Apply overlay | `sudo /opt/nvidia/jetson-io/config-by-hardware.py -n 2="<overlay-name>"` then reboot | Changes boot config on live hardware |

Smoke tests after boot *(suggested, not project-defined)*: `dmesg | grep -i v-link` should
show "Probed succesfully" (sic) for both drivers and "GPIOCHIP probed" for the serializer;
`lsmod | grep v_link`; `ls /dev/video*`; then a capture, e.g.
`gst-launch-1.0 nvarguscamerasrc sensor-id=0 ! fakesink` or `v4l2-ctl --stream-mmap`.

There are no unit tests, CI, or linters in the project. `scripts/checkpatch.pl` from the kernel
tree would be the natural lint *(not set up)*.

## Hardware topology and task/program layout

### Driver behaviour (both are I2C clients, 16-bit register / 8-bit value regmap, max reg 0x1f00)

Probe sequence (both drivers): allocate → regmap → parse DT → create a 1-channel I2C mux with
a no-op `select` (child adapter has no DT node; nothing in the overlays uses it, so its purpose
is unclear) → program the chip → register a V4L2 subdev (2 pads: 0 = sink, 1 = source,
function `MEDIA_ENT_F_VID_IF_BRIDGE`, `link_validate` = default) via `v4l2_async_register_subdev`.

Streaming is controlled only by `s_stream`; there are no pad/format ops, power management, or
link-lock checks. The bridges rely on NVIDIA's VI driver calling `s_stream` on every subdev in
the graph *(unverified)*.

**Serializer (MAX96717), `v-link-ser.c`**
- DT: `bus-width` (1-4, required) = CSI lanes from the sensor. Also `gpio-controller`, `#gpio-cells = <2>`.
- Setup: `MIPI_RX1` (0x331) bits 5:4 = lanes-1; `MIPI_RX5` (0x335) PHY2 polarity = 0x7;
  `MIPI_RX4` (0x334) PHY1 polarity = 0 (code passes `0x03 >> 3`, see ISSUES V-004); lane map
  `MIPI_RX3`/`MIPI_RX2` = identity (computed value is always 0xE4 for 1-4 lanes).
- GPIO chip: 1 line (MFP0, register `0x2be`) used as the camera reset. Forwarding (RX/TX) is
  disabled at probe. Overlays use it as `reset-gpios = <&serializer_x 0 1>` (active low).
- `s_stream(1/0)`: set/clear `FRONTOP0` (0x308) bit 5 `START_PORT_B`.
- Empty V4L2 control handler; `open` internal op only logs.

**Deserializer (MAX96714), `v-link-deser.c`**
- DT: `bus-width` (1-4, required) = CSI lanes to the Jetson; `link-freq` (required) as a
  **decimal string** in Hz, 50-1250 MHz in 50 MHz steps.
- Setup: write 0xC4 to 0x332 (leaves `MIPI_STDBY_N` bits 5:4 = 0, i.e. CSI output in standby);
  `BACKTOP25` (0x320) bits 4:0 = `link_freq*2 / 100 MHz` (per-lane data rate in 100 Mbps
  units: 800 MHz → 16 → 1.6 Gbps); `MIPI_LANE_CNT` (0x44a) bits 7:6 = lanes-1;
  `MIPI_POLARITY` (0x335) = 0x3B; lane map (0x333) intended identity (see ISSUES V-001).
- Exposes `V4L2_CID_LINK_FREQ` (single-item int menu = `link-freq`).
- `s_stream(1/0)`: set/clear 0x332 bits 5:4 (take the CSI PHYs out of / into standby).

### Overlay topology

**Orin Nano / NX devkit (p3768), `orin_nano_super/`** — one camera per overlay. The cam0 and cam1
overlays both define VI `num-channels = <1>` / `port@0`, so they are alternatives, not combinable.

| Overlay | I2C path | CSI | Sensor | Notes |
|---|---|---|---|---|
| `…-imx219-vlink-cam0` | `cam_i2cmux` (gpio mux, AON GPIO CC.3) `i2c@0` | `serial_b`, port-index 1, 2 lanes | IMX219 @0x10, 5 modes up to 3280x2464 | `lane_polarity = "6"` |
| `…-imx219-vlink-cam1` | `cam_i2cmux` `i2c@1` | `serial_c`, port-index 2, 2 lanes | IMX219 | no lane_polarity |
| `…-imx477-vlink-cam0` | `i2c@0` | `serial_b`, port 1 | IMX477 @0x1a, 3840x2160@30, 1920x1080@60 | `lane_polarity = "6"` |
| `…-imx477-vlink-cam1` | `i2c@1` | `serial_c`, port 2 | IMX477 | no lane_polarity |

All four: ser @0x42 `bus-width=2`, deser @0x4c `bus-width=2`, `link-freq="800000000"`.

**AGX Orin devkit (p3737), `agx_orin/…-overlay-6-imx219-vlink`** — six IMX219 cameras. On
`i2c@3180000`: a PCA9555 GPIO expander @0x20 (declared, not referenced) and a PCA9548 mux
@0x70; mux channels 0-5 each hold ser @0x42, deser @0x4c (`link-freq="500000000"`, changed from
450 MHz in `1d4c85b`), IMX219 @0x10 (`video0`-`video5`). CSI ports used: A, B, C, D, E, G
(nvcsi port-index 0,1,2,3,4,6; there is no F). The deser nodes declare `gpio-controller` but the
driver registers no gpiochip.

Graph per camera (both platforms): sensor endpoint → ser `port@0` → ser `port@1` → deser `port@0`
→ deser `port@1` → nvcsi channel `port@0` → nvcsi `port@1` → VI port. `tegra-camera-platform`
`drivernode0` points at the sensor's sysfs DT path.

## Error model and approved safe states

This is a camera/video path, not a machine-safety function; no safety requirements are recorded.
Failure handling in code: register access failures are logged (`dev_err`) and abort probe or
`s_stream`; DT validation errors return `-EINVAL`. There is no link-lock detection, retry, or
recovery, and the I2C mux adapter is not cleaned up on probe failure (ISSUES V-002).
`boost_clocks.sh` writes BPMP debugfs as root; it locks clocks at max until reboot.

## Decisions and known issues

- **GMSL link configuration is not done by the drivers.** Link rate, I2C pass-through, and the
  ser/deser I2C addresses (0x42 / 0x4c) are assumed to come from the boards' pin straps
  *(inference: no driver code touches them)*. The sensor at 0x10/0x1a is reached through the
  link's I2C pass-through on the same bus segment as the deser.
- **Bridges are separate subdevs** in the VI graph so the stock NVIDIA sensor drivers and
  camera stack (Argus) work unchanged; only the overlay needs editing per sensor.
- **Camera reset comes from the serializer's MFP0** exposed as a gpiochip, so stock sensor
  drivers can use `reset-gpios`. Probe order is left to deferred probing.
- **Lane polarity values are fixed in code** ("based on HW design"), so they are tied to the
  current V-Link boards, not configurable from DT.
- **`link-freq` is a non-standard string property** instead of the standard V4L2
  `link-frequencies` u64 array on the endpoint.
- Checked-in `.dtbo` files are build outputs of the `.dts`; nothing checks they stay in sync,
  and the AGX one is already stale (ISSUES V-012). Rebuild the `.dtbo` whenever a `.dts` changes.

Defects and doubts are logged in `.agents/ISSUES.md` (V-001 … V-011).

## Future direction

Owner-approved plans belong in `.agents/ROADMAP.md`; do not treat ideas as authorized scope.
