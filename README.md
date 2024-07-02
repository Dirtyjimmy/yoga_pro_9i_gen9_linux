# Running Linux on the Lenovo Yoga Pro 9i Gen 9 (16IMH9)

Here are some notes for running Linux on this laptop. This guide was primarily designed for the Gen 9 with the MiniLED screen, but it may also be helpful for the Gen 8 or similar laptops from Lenovo. Currently, KDE Plasma is recommended (refer to the Screen section) for the optimal experience.

This guide was done on the Fedora 40 based distribution [Aurora](https://getaurora.dev/) using Wayland.

## Feature Compatibility Table

| Feature                 | Status                          | Notes                                                                                                  |
| ----------------------- | ------------------------------- | ------------------------------------------------------------------------------------------------------ |
| Screen                  | Incomplete, workaround with KDE | Requires KDE Plasma for correct colors                                                                 |
| Speakers                | Needs configuring               | Subwoofer is disabled by default due to a bug                                                          |
| Nvidia GPU              | Needs configuring               | General configuration required for nvidia optimus                                                      |
| Windows Hello IR Camera | Needs configuring               | Detected out of the box; configuring [howdy](https://github.com/boltgolt/howdy) can be a bit difficult |
| Battery                 | Works                           | Good battery life; better with refresh rate set to 60Hz                                                |
| Keyboard                | Works                           | Includes backlight adjustment in GNOME and KDE                                                         |
| Camera                  | Works                           |                                                                                                        |
| Touchpad                | Works                           |                                                                                                        |
| Intel GPU               | Works, with some instability    | with MESA version >24.1.1                                                                              |
| Microphone              | Works\*                         | Add Speech processor via EasyEffects for higher volume                                                 |
| Ports                   | Works                           |                                                                                                        |
| Wifi/Bluetooth          | Works                           |                                                                                                        |
| Proximity Sensor        | Not supported                   |

## Screen

This laptop features a wide gamut display. The display controller used in this laptop (more specifically the TCON) supports sRGB, but defaults to bt.2020.
Under linux, the TCON is never switched to sRGB, resulting in overblown colors.
Until we have a working driver (if ever have one), we can either:

- (Only works under KDE Plasma for now!) Use [this](https://github.com/maximmaxim345/yoga_pro_9i_gen9_linux/raw/main/LEN160_3_2K_cal-linux.icc) icc profile which transform everything on screen to bt.2020. Save it somewhere safe, and select it under `Display & Monitor > Color Profile`.
- Or use a wayland compositor that supports color management with native bt.2020 support. This does not exist yet, but I briefly tried [manually compiling this](https://gitlab.gnome.org/GNOME/mutter/-/merge_requests/3433) with custom patches to enable bt.2020 on all displays even if HDR is off.

The screen supports VRR from 48-165Hz. I have Adaptive Sync set to "Always," which results in slightly better battery life.

As I know, HDR is currently disabled for internal screens on Intel Arc GPUs due to a bug (maybe also related to the TCON). This should change in the future.

## Speakers

Speakers require some configuration. These instructions are based on [this issue](https://github.com/karypid/YogaPro-16IMH9/issues/2) and [this discussion](https://bugzilla.kernel.org/show_bug.cgi?id=217449)

### Configuration Steps

1. Create a systemd service file by copying the following command into a terminal:

   ```bash
   sudo tee /etc/systemd/system/yoga-16imh9-speakers.service <<EOF
   [Unit]
   Description=Turn on speakers using i2c configuration

   [Service]
   User=root
   Type=oneshot
   ExecStart=/bin/sh -c "/usr/local/bin/2pa-byps.sh 2 | logger"

   [Install]
   WantedBy=multi-user.target
   EOF
   ```

2. Install or ensure the `i2c-tools` package is installed

   Depending on your Linux distribution, use one of the following commands:

   For Debian/Ubuntu based distributions:

   ```bash
   sudo apt update
   sudo apt install i2c-tools
   ```

   For Fedora:

   ```bash
   sudo dnf install i2c-tools
   ```

   For Fedora Atomic Desktop:

   ```bash
   sudo rpm-ostree install i2c-tools
   ```

   For Arch Linux:

   ```bash
   sudo pacman -Sy i2c-tools
   ```

3. Create the script `/usr/local/bin/2pa-byps.sh`, copy the following command into a terminal:

   ```bash
   sudo tee /usr/local/bin/2pa-byps.sh <<'EOF'
   #!/bin/bash

   clear
   function clear_stdin() {
       old_tty_settings=$(stty -g)
       stty -icanon min 0 time 0
       while read -r none; do :; done
       stty "$old_tty_settings"
   }

   if [ $# -ne 1 ]; then
       echo "Kindly specify the i2c bus number. The default i2c bus number is 3."
       echo "Command as following:"
       echo "$0 i2c-bus-number"
       i2c_bus=3
   else
       i2c_bus=$1
   fi

   echo "i2c bus is $i2c_bus"
   i2c_addr=(0x3f 0x38)

   count=0
   for value in "${i2c_addr[@]}"; do
       val=$((count % 2))
       i2cset -f -y "$i2c_bus" "$value" 0x00 0x00
       i2cset -f -y "$i2c_bus" "$value" 0x7f 0x00
       i2cset -f -y "$i2c_bus" "$value" 0x01 0x01
       i2cset -f -y "$i2c_bus" "$value" 0x0e 0xc4
       i2cset -f -y "$i2c_bus" "$value" 0x0f 0x40
       i2cset -f -y "$i2c_bus" "$value" 0x5c 0xd9
       i2cset -f -y "$i2c_bus" "$value" 0x60 0x10
       if [ $val -eq 0 ]; then
           i2cset -f -y "$i2c_bus" "$value" 0x0a 0x1e
       else
           i2cset -f -y "$i2c_bus" "$value" 0x0a 0x2e
       fi
       i2cset -f -y "$i2c_bus" "$value" 0x0d 0x01
       i2cset -f -y "$i2c_bus" "$value" 0x16 0x40
       i2cset -f -y "$i2c_bus" "$value" 0x00 0x01
       i2cset -f -y "$i2c_bus" "$value" 0x17 0xc8
       i2cset -f -y "$i2c_bus" "$value" 0x00 0x04
       i2cset -f -y "$i2c_bus" "$value" 0x30 0x00
       i2cset -f -y "$i2c_bus" "$value" 0x31 0x00
       i2cset -f -y "$i2c_bus" "$value" 0x32 0x00
       i2cset -f -y "$i2c_bus" "$value" 0x33 0x01

       i2cset -f -y "$i2c_bus" "$value" 0x00 0x08
       i2cset -f -y "$i2c_bus" "$value" 0x18 0x00
       i2cset -f -y "$i2c_bus" "$value" 0x19 0x00
       i2cset -f -y "$i2c_bus" "$value" 0x1a 0x00
       i2cset -f -y "$i2c_bus" "$value" 0x1b 0x00
       i2cset -f -y "$i2c_bus" "$value" 0x28 0x40
       i2cset -f -y "$i2c_bus" "$value" 0x29 0x00
       i2cset -f -y "$i2c_bus" "$value" 0x2a 0x00
       i2cset -f -y "$i2c_bus" "$value" 0x2b 0x00

       i2cset -f -y "$i2c_bus" "$value" 0x00 0x0a
       i2cset -f -y "$i2c_bus" "$value" 0x48 0x00
       i2cset -f -y "$i2c_bus" "$value" 0x49 0x00
       i2cset -f -y "$i2c_bus" "$value" 0x4a 0x00
       i2cset -f -y "$i2c_bus" "$value" 0x4b 0x00
       i2cset -f -y "$i2c_bus" "$value" 0x58 0x40
       i2cset -f -y "$i2c_bus" "$value" 0x59 0x00
       i2cset -f -y "$i2c_bus" "$value" 0x5a 0x00
       i2cset -f -y "$i2c_bus" "$value" 0x5b 0x00

       i2cset -f -y "$i2c_bus" "$value" 0x00 0x00
       i2cset -f -y "$i2c_bus" "$value" 0x02 0x00
       count=$((count + 1))
   done
   clear_stdin
   EOF
   ```

4. Make the script executable:

   ```bash
   sudo chmod +x /usr/local/bin/2pa-byps.sh
   ```

5. Enable and start the service:

   ```bash

   sudo systemctl daemon-reload
   sudo systemctl enable yoga-16imh9-speakers.service
   sudo systemctl start yoga-16imh9-speakers.service
   ```

For better audio quality, you can also use [this EasyEffects profile](https://github.com/maximmaxim345/yoga_pro_9i_gen9_linux/raw/main/Yoga%20Pro%209i%20gen%209%20v2.json). If you have problems with too much vibrations use [this profile](https://github.com/maximmaxim345/yoga_pro_9i_gen9_linux/raw/main/Yoga%20Pro%209i%20gen%209%20v2%20less%20bass.json) instead.

If the file opens in your browser, right-click the link and select "Save link as..." to download it. You can get EasyEffects from [Flathub](https://flathub.org/apps/com.github.wwmm.easyeffects). Select the json file under `Presets > Import a preset` button, when in the `Output` tab, and load it.

<details>
    <summary>Details about the profile</summary>

    Measured in two accustically different rooms while sitting on a desk using a calibrated Sonarworks SoundID microphone.
    Made using REW.

    Measures flat from 400Hz with a 12dB/oct falloff (18dB/oct for the less bass version)

</details>

## Nvidia GPU

Works. Both KDE and GNOME tested (with wayland).

Enable lower power consumption by running this command:

```bash
echo "options nvidia NVreg_DynamicPowerManagement=0x02" | sudo tee /etc/modprobe.d/nvidia.conf
```

Fix suspend issues by running:

```bash
sudo systemctl enable nvidia-hibernate.service nvidia-persistenced.service nvidia-resume.service nvidia-suspend.service
```

## Windows Hello IR Camera

The IR camera is detected out of the box. Configuring [howdy](https://github.com/boltgolt/howdy) is rather complicated. See [here](https://copr.fedorainfracloud.org/coprs/principis/howdy-beta/) or [here](https://wiki.archlinux.org/title/Howdy) for setup instructions. I have had success running the beta version. You will also need [linux-enable-ir-emitter](https://github.com/EmixamPP/linux-enable-ir-emitter) for the IR emitter to work.

## Battery

I have good battery life, more or less equal to Windows. If you need more battery, set the refresh rate to 60Hz. This results in:

| Activity               | Battery Life |
| ---------------------- | ------------ |
| Idle                   | 10h          |
| Light use              | 8h           |
| Programming            | 6h 30m       |
| YouTube video playback | 6h 30m       |

## Keyboard

Everything works out of the box, including backlight adjustment in GNOME and KDE.

## Camera

Works out of the box.

## Touchpad

Works out of the box with good palm rejection.

## Intel GPU

Works with MESA version 24.1.1 and later.
Some hangs may occur from time to time (tested on Plasma), [this is a known issue in mesa](https://gitlab.freedesktop.org/drm/i915/kernel/-/issues/10395).

This is issue easily reproducible:

1. Start KDE Plasma with Wayland and the [kzones](https://github.com/gerritdevriese/kzones) kwin script enabled.
2. Connect/disconnect an external monitor. Or alternatively change the Scale of the laptop screen.
3. Try to move a window.
4. Observe the indefinite hang. If it does not hang, try again from step 2.

## Microphone

A bit quiet by default, add a `Speech Processor` via EasyEffects to boost the volume.
This laptop has a quad mic array, further investigation is needed for noise cancellation.

## Ports

Works out of the box.

## Thunderbolt

Tested on the [DELL WD19TB](https://www.dell.com/support/home/en-us/product-support/product/dell-wd19tb-dock/overview) and works out of the box (network, ports etc).  
However, eventhough the laptop appears to be charging, the battery status on PopOS does not say so.

## Wifi/Bluetooth

Works out of the box.

## Proximity Sensor

This laptop also has a proximity sensor, which can be used to lock the screen when you walk away from it. This is not yet supported on Linux.
