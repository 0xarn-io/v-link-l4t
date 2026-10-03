# Pi Camera v1.3 (OV5647) over V-Link: test notes

Branch `pi-test`. Everything here is untested on hardware: it was written on a machine with
no compiler, no `dtc` and no Jetson. Expect to iterate.

## What this branch adds

- `v-link-ser`, `v-link-deser`: build on kernel 5.15 (Jetson Linux R36) and 6.8 (R39), and the
  deserializer no longer uses an uninitialized lane count.
- `overlays/orin_nano_super/tegra234-p3767-camera-p3768-ov5647-vlink-cam0.dts` and `-cam1.dts`:
  OV5647 behind the serializer/deserializer on the Orin Nano devkit.

The OV5647 sensor driver is not in this repository. It comes from
https://github.com/akandr/ov5647-jetson (GPL-2.0). NVIDIA ships no driver for this sensor.

## Hardware

Camera (15-pin ribbon) -> V-Link serializer -> coax -> V-Link deserializer -> Orin Nano CAM0
(or CAM1 with the `cam1` overlay).

## Steps on the Jetson

1. Record the release, it decides which notes below apply:

       cat /etc/nv_tegra_release
       uname -r

2. Build tools and headers:

       sudo apt install build-essential device-tree-compiler git

3. Build and install the two V-Link drivers (from this branch):

       cd v-link-ser   && make && sudo make install && cd ..
       cd v-link-deser && make && sudo make install && cd ..

4. Build and install the OV5647 driver. Build only the `driver/` directory. Do NOT run that
   project's `install.sh`: it installs its own direct-attach overlay and edits `extlinux.conf`.

       git clone https://github.com/akandr/ov5647-jetson
       cd ov5647-jetson/driver && make && sudo make install

   That Makefile knows R32, R35 and JetPack 7 (R39). On R36 it will not find the headers;
   stop there and report the output of `ls /usr/src` and `ls /usr/src/linux-headers-*`.

5. Compile and install the overlay:

       cd overlays/orin_nano_super
       dtc -I dts -O dtb -@ tegra234-p3767-camera-p3768-ov5647-vlink-cam0.dts \
           -o tegra234-p3767-camera-p3768-ov5647-vlink-cam0.dtbo
       sudo cp tegra234-p3767-camera-p3768-ov5647-vlink-cam0.dtbo /boot/

6. Select it and reboot:

       sudo /opt/nvidia/jetson-io/config-by-hardware.py -l
       sudo /opt/nvidia/jetson-io/config-by-hardware.py -n 2="Camera OV5647 (CAM0) over v-link"
       sudo reboot

   If the overlay is not listed (possible on R39, where the header name may differ), report
   the `-l` output instead of forcing it.

## Checks after reboot, in order

Stop at the first one that fails and keep its output.

1. Drivers bound:

       sudo dmesg | grep -iE "v-link|ov5647|max967"

   Expected: `v-link-ser ... Probed succesfully`, `GPIOCHIP probed`,
   `v-link-deser ... Probed succesfully`, `OV5647 detected (chip id 0x5647)`.

2. Devices on the camera I2C bus (find the bus number with `i2cdetect -l`, look for the mux
   channel): `0x42` serializer, `0x4c` deserializer, `0x36` sensor show as `UU`.

3. Video node exists: `ls /dev/video*` and `v4l2-ctl -d /dev/video0 --all`.

4. Raw capture, 1080p:

       v4l2-ctl -d /dev/video0 --set-ctrl=sensor_mode=1 \
           --set-fmt-video=width=1920,height=1080,pixelformat=BG10 \
           --stream-mmap --stream-count=30 --stream-to=frames.raw

   Success is 30 frames and a `frames.raw` of 30 x 1920 x 1080 x 2 bytes = 124416000 bytes.
   On failure also save `sudo dmesg | tail -80`.

Argus (`nvarguscamerasrc`) is not expected to work: it needs a tuning file that does not exist
for this sensor.

## What each failure points to

| Symptom | Likely area |
|---|---|
| `make` errors in `v-link-*` | Kernel API difference; send the full error |
| Serializer or deserializer not probed | Link not up, wrong connector, or overlay not applied |
| Sensor chip id read fails | Camera not powered: enable line (serializer GPIO) or ribbon |
| Everything bound, capture times out | CSI settings: try the fallbacks below |

Apply only one of the `cam0` / `cam1` overlays at a time; they define the same nodes.

If the sensor's chip-id read fails once (link not locked yet), the driver does not retry.
Rebind it by hand (replace `9-0036` with the name shown in `ls /sys/bus/i2c/devices`):

    echo 9-0036 | sudo tee /sys/bus/i2c/drivers/ov5647/bind

Fallbacks for capture timeouts, one at a time (rebuild the `.dtbo`, copy to `/boot`, reboot):

1. Try the other connector with the other overlay (CAM0 <-> CAM1). The two connectors differ
   in lane polarity on the Jetson side and the overlays do not compensate for it.
2. Add `lane_polarity = "6";` (a string) inside the nvcsi `channel@0` node of the overlay.
   With the serializer in the path, that node is where NVIDIA's capture code reads it; the
   `lane_polarity` inside the sensor modes is not applied.
3. Lower the deserializer `link-freq` to `"250000000"` (500 Mbps per lane, still above the
   437.5 Mbps the sensor sends).
