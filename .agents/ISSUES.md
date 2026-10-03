# Issues and audit notes

Add observed issues with an identifier, date, affected version/commit, evidence, impact,
owner/status, and verification of any fix. Keep observations separate from hypotheses.
Do not silently rewrite previous findings.

## Open issues

All entries below: found 2026-10-03 by desk review (reading the code, not building or running it) of commit `aacb56f`,
branch `l4t-r36`. Status: open, owner unassigned, no fix applied. "Observation" means read
directly from the code; "Hypothesis" needs a datasheet, schematic, or hardware test to confirm.

### V-001 Deserializer lane map uses an uninitialized variable — high
- Observation: `v_link_deser_setup()` (`v-link-deser/v-link-deser.c:118`) declares
  `nlanes` and uses it as the loop bound at line 152, but never assigns it. The serializer
  version assigns `nlanes = priv->bus_width` (`v-link-ser.c:220`).
- Impact: undefined behaviour. If the stack garbage is 0-4 the written map is still identity
  (0xE4) because unused lanes are also filled in order, which probably explains why it works.
  A large value would shift past the width of `val` (32-bit) and `lanes_used` (64-bit), which is
  UB, and could loop for a very long time in probe.
  The compiler should warn (`-Wuninitialized`).
- Fix direction: `nlanes = priv->bus_width;`, matching the serializer.
- 2026-10-03: that fix is applied in the working tree on branch `pi-test` (uncommitted, not
  compiled or run). Still open on `l4t-r36`.

### V-002 I2C mux adapter leaked on probe failure — medium
- Observation: both probes call `*_i2c_mux_init()` (non-devm) before chip setup and V4L2
  registration; on failure they return without `i2c_mux_del_adapters()`.
- Impact: a stray I2C adapter remains after a failed probe; a retried probe adds another.

### V-003 V4L2 registration error path — low
- Observation: in `*_v4l2_register()`, a failed `v4l2_async_register_subdev()` jumps to a
  label that calls `v4l2_async_unregister_subdev()` on a subdev that was never registered,
  and `media_entity_cleanup()` is never called on error.

### V-004 Serializer PHY1 polarity value is always 0 — hypothesis
- Observation: `v-link-ser.c:239-241` sets `val = 0x03` then writes `FIELD_PREP(mask, val >> 3)`,
  which is 0. The `0x03` has no effect.
- Hypothesis: either PHY1 polarity really should be 0 (and the code is just confusing) or a
  different value was intended. Check against the board schematic / MAX96717 guide.

### V-005 `FIELD_PREP` used without including `<linux/bitfield.h>` — low, unverified
- Observation: both drivers use `FIELD_PREP` but do not include `linux/bitfield.h`; it must
  arrive through another header. Fragile across kernel versions; not confirmed to fail.
- 2026-10-03: include added on branch `pi-test` (uncommitted, not compiled).

### V-006 Misleading log messages — low
- `*_read()` logs "write 0x%04x failed" (both drivers).
- Deser `parse_dt` logs "Missing pix_clk_hz property" for the `link-freq` property.
- Deser probe logs "Unable to setup serializer".
- "Probed succesfully" (typo) in both; changing it affects anyone grepping dmesg for it.

### V-007 AGX sixth camera port-index inconsistency — hypothesis
- Observation: in `overlays/agx_orin/…-6-imx219-vlink.dts`, VI `port@5` endpoint and the
  sensor `imx219_g` endpoint use `port-index = <5>`, but the nvcsi `channel@5` input uses
  `port-index = <6>` with `tegra_sinterface = "serial_g"`. The other five cameras are consistent.
- Hypothesis: either intentional (CSI port G is index 6) and the VI/sensor values are ignored,
  or it breaks camera 6. Needs a capture test on camera 6.

### V-008 `make uninstall` calls a target that probably does not exist — low
- Observation: both Makefiles run `$(MAKE) -C $(KERNEL_DIR) M=... modules_remove`.
  Kbuild is not known to have a `modules_remove` target *(unverified; run it in Ubuntu to confirm)*.

### V-009 README clock-boost instructions use a wrong path — low
- Observation: README runs `chmod +x scripts/boost_clocks.sh`, then `sudo su` and
  `./boost_clocks.sh` (from the repo root that file is at `scripts/boost_clocks.sh`).

### V-010 Unused DT declarations — low
- Observation: AGX deser nodes declare `gpio-controller` / `#gpio-cells`, but the deser driver
  registers no gpiochip; the AGX PCA9555 @0x20 is declared but not referenced. A future
  `<&deserializer_x …>` GPIO reference would defer forever.
- The driver-created 1-channel I2C mux child adapters have no DT node and are not used by any
  overlay; their purpose is unknown.

### V-011 Cosmetic overlay/code inconsistencies — low
- IMX219 `mode3` comments say 60 fps, values are 30 fps (Orin Nano overlays).
- `num_csi_lanes = <4>` in the single-camera, 2-lane Orin Nano overlays.
- Drivers mix 2-space and tab indentation; not kernel style for `checkpatch.pl`.

### V-012 AGX `.dtbo` is stale: ships 450 MHz, `.dts` says 500 MHz — medium
- Observation: commit `1d4c85b` changed all six `link-freq` values in the AGX `.dts` from
  `"450000000"` to `"500000000"` but did not touch the `.dtbo`. A byte search of
  `overlays/agx_orin/…-6-imx219-vlink.dtbo` finds `450000000` six times and `500000000` zero
  times. The four Orin Nano `.dtbo` files each contain `800000000` once, matching their `.dts`.
- Impact: users following the README copy the prebuilt `.dtbo` and get the old 450 MHz link
  frequency, not the intended 500 MHz.
- Fix direction (in Ubuntu): rebuild with `dtc -I dts -O dtb -@ X.dts -o X.dtbo`. Ideally
  compare every `.dtbo` with a fresh build (`dtc -I dtb -O dts`) to catch other drift.

### V-013 Sensor-mode CSI settings are not applied to NVCSI with V-Link in the path � hypothesis on impact
- Found 2026-10-03 by desk review of NVIDIA nvidia-oot source (OE4T/linux-nv-oot `l4t-r36.4.4`
  and `l4t-r38.4`; R39.2 source not read).
- Observation: `tegra_channel_connect_sensor()` sets the CSI channel's sensor data only when the
  sensor endpoint's remote parent is the NVCSI channel node. In every V-Link overlay that parent
  is the serializer, so `csi5_stream_set_config()` takes its other branch: `lane_polarity` and
  `cil_settletime` are read as string properties from the nvcsi `channel@N` node (absent in all
  overlays, so 0) and the clock rate is a fixed default, not derived from `pix_clk_hz`.
- Impact: `lane_polarity = "6"` in the CAM0 sensor modes has no effect, and `serdes_pix_clk_hz`
  would not reach NVCSI. Hypothesis: one of CAM0/CAM1 may see inverted lanes unless the
  deserializer board compensates; not tested.

## Integrity limits

This file is an editable working log, not a tamper-evident audit system. A checksum stored
in the same editable file cannot establish who changed it or prevent a matching rewrite.
Use repository history for ordinary review. Where independent audit assurance is required,
use an externally trusted, access-controlled record according to project policy.

Never place credentials, private customer data, or unnecessary raw logs here.
