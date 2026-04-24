# Flashing OpenWrt on a Zyxel EX5601-T0 (ACDZ Firmware, Locked U-Boot)

This guide documents the process of installing OpenWrt on a Zyxel EX5601-T0 running stock firmware `V5.70(ACDZ.4)C0` with a locked u-boot (eFuse blown, Secure Boot enforced). This was a secondhand unit from Grain (UK).

Credit to the [OpenWrt forum thread](https://forum.openwrt.org/t/adding-openwrt-support-for-zyxel-ex5601-t0/155914) community, and particularly to **@dando** whose [guide](https://github.com/ndandanov/zyxel-ex5601-t0-openwrt/blob/main/guide/guide.md) was the key reference for this process.

---

## Hardware Required

- Zyxel EX5601-T0
- Linux laptop with a built-in or USB ethernet adapter
- USB-to-TTL serial adapter (CH340N or similar, **3.3V logic level**)
- 3x female-to-female Dupont jumper wires
- Ethernet cable

---

## Identifying Your Firmware

Connect an ethernet cable from your laptop to a yellow LAN port on the router. You may need to force the link speed if autonegotiation fails:

```bash
sudo ethtool -s enp0s31f6 speed 100 duplex full autoneg off
sudo ip addr add 192.168.1.215/24 dev enp0s31f6
```

Navigate to `http://192.168.1.1` in a browser and log in with the credentials on the router's sticker. Check **Maintenance → Firmware Information** for your firmware version.

This guide applies if you see:
- Firmware: `V5.70(ACDZ.x)C0`
- The following lines in the serial boot output (indicating locked u-boot):

```
BP: 2400 0209 [0000]
```

Instead of:
```
BP: 2400 0041 [0000]
```

---

## Serial Console Setup

### Finding the UART Header

The UART header is labelled **J1** on the PCB, located near the bottom edge of the board (near the sticker labelled `250215`). It is a 5-pin black header with pins already populated — no soldering required.

Pinout (determined by multimeter continuity and voltage testing):

| Pin | Function |
|-----|----------|
| 1 | VCC (3.3V) — **do not connect** |
| 2 | TX (router transmits) |
| 3 | RX (router receives) |
| 4 | Not connected |
| 5 | GND |

### Connecting the Serial Adapter

| Router J1 | → | Adapter |
|-----------|---|---------|
| Pin 2 (TX) | → | RXD |
| Pin 3 (RX) | → | TXD |
| Pin 5 (GND) | → | GND |
| Pin 1 (VCC) | → | **leave unconnected** |

### Opening a Serial Terminal

```bash
sudo usermod -a -G dialout $USER
# log out and back in
screen /dev/ttyUSB0 115200
```

Use `Ctrl+A C` to create new screen windows, `Ctrl+A N` to switch between them.

To log boot output to a file:
```bash
screen -L -Logfile /tmp/router_boot.log /dev/ttyUSB0 115200
```

---

## Factory Reset (if secondhand)

If the admin password has been changed, perform a factory reset:

1. Power on the router
2. Insert a pin into the reset pinhole on the back
3. Hold for 10-15 seconds until LEDs flash
4. Release and wait for reboot
5. Log in with the credentials printed on the sticker

---

## Gaining Root SSH Access

### Step 1 — Log in via SSH to ZySH

Enable SSH in the router web UI under **Maintenance → Remote Management**, then:

```bash
ssh admin@192.168.1.1
```

### Step 2 — Get the MAC Address

At the `ZySH>` prompt:

```
sys atsh
```

Note the value of `First MAC Address` (e.g. `30BD1380DD40`).

### Step 3 — Set the Engineering Flag

```
sys atwz 30BD1380DD40 0 1
```

Replace `30BD1380DD40` with your MAC address (no colons).

### Step 4 — Get the Root Password

```
sys atck
```

The output contains the `supervisor password` — this is your root password. **Save it somewhere safe.**

### Step 5 — Log in as Root

```bash
ssh root@192.168.1.1
```

---

## Accessing the MT7986 U-Boot Shell

With the serial console connected and watching:

1. Power cycle the router
2. Watch for `Hit any key to stop autoboot` — spam the spacebar immediately
3. You should land at `ZHAL>`
4. Type `atgu` and immediately spam the spacebar again
5. You may land back at `ZHAL>` — type `atgu` again, then spam spacebar
6. Repeat until you see `MT7986>`

---

## Setting Up the Laptop

### Install Tools

```bash
sudo apt install tftpd-hpa gcc wget screen
```

### Configure Ethernet

```bash
sudo ethtool -s enp0s31f6 speed 100 duplex full autoneg off
sudo ip addr add 192.168.1.215/24 dev enp0s31f6
```

### Download OpenWrt Files

```bash
mkdir -p ~/openwrt && cd ~/openwrt

BASE=https://downloads.openwrt.org/releases/23.05.5/targets/mediatek/filogic
KMODS=https://downloads.openwrt.org/releases/23.05.5/targets/mediatek/filogic/kmods/5.15.167-1-03ba5b5fee47f2232a088e3cd9832aec

wget $BASE/openwrt-23.05.5-mediatek-filogic-zyxel_ex5601-t0-ubootmod-bl31-uboot.fip
wget $BASE/openwrt-23.05.5-mediatek-filogic-zyxel_ex5601-t0-ubootmod-preloader.bin
wget $BASE/openwrt-23.05.5-mediatek-filogic-zyxel_ex5601-t0-ubootmod-initramfs-recovery.itb
wget $BASE/openwrt-23.05.5-mediatek-filogic-zyxel_ex5601-t0-ubootmod-squashfs-sysupgrade.itb
wget $KMODS/kmod-mtd-rw_5.15.167+git-20160214-2_aarch64_cortex-a53.ipk
```

### Set Up TFTP Server

```bash
sudo mkdir -p /srv/tftp
sudo cp openwrt-23.05.5-mediatek-filogic-zyxel_ex5601-t0-ubootmod-initramfs-recovery.itb /srv/tftp/openwrt.itb
sudo chown -R tftp:tftp /srv/tftp
sudo systemctl start tftpd-hpa
```

### Set Up HTTP Server (for file transfers to router)

```bash
cd ~/openwrt
python3 -m http.server 8081 &
```

---

## Booting OpenWrt from RAM via TFTP

At the `MT7986>` prompt:

```
setenv serverip 192.168.1.215
tftpboot openwrt.itb
bootm 0x46000000
```

Wait for OpenWrt to boot, then SSH in:

```bash
ssh root@192.168.1.1
# no password required at this stage
```

---

## Permanently Flashing OpenWrt

All commands below are run on the router over SSH.

### Step 1 — Transfer Files from Laptop

```
cd /tmp
wget http://192.168.1.215:8081/kmod-mtd-rw_5.15.167+git-20160214-2_aarch64_cortex-a53.ipk
wget http://192.168.1.215:8081/openwrt-23.05.5-mediatek-filogic-zyxel_ex5601-t0-ubootmod-preloader.bin
wget http://192.168.1.215:8081/openwrt-23.05.5-mediatek-filogic-zyxel_ex5601-t0-ubootmod-bl31-uboot.fip
wget http://192.168.1.215:8081/openwrt-23.05.5-mediatek-filogic-zyxel_ex5601-t0-ubootmod-initramfs-recovery.itb
wget http://192.168.1.215:8081/openwrt-23.05.5-mediatek-filogic-zyxel_ex5601-t0-ubootmod-squashfs-sysupgrade.itb
```

### Step 2 — Install and Load the MTD Module

```
opkg install /tmp/kmod-mtd-rw_5.15.167+git-20160214-2_aarch64_cortex-a53.ipk
insmod mtd-rw.ko i_want_a_brick=1
```

### Step 3 — Flash Preloader and FIP

```
mtd write openwrt-23.05.5-mediatek-filogic-zyxel_ex5601-t0-ubootmod-preloader.bin bl2
mtd write openwrt-23.05.5-mediatek-filogic-zyxel_ex5601-t0-ubootmod-bl31-uboot.fip fip
```

### Step 4 — Set Up UBI Partition

```
ubidetach -p /dev/mtd5; ubiformat /dev/mtd5 -y; ubiattach -p /dev/mtd5
ubimkvol /dev/ubi0 -n 0 -N ubootenv -s 128KiB
ubimkvol /dev/ubi0 -n 1 -N ubootenv2 -s 128KiB
ubimkvol /dev/ubi0 -n 2 -N recovery -s 10MiB
ubiupdatevol /dev/ubi0_2 /tmp/openwrt-23.05.5-mediatek-filogic-zyxel_ex5601-t0-ubootmod-initramfs-recovery.itb
```

### Step 5 — Sysupgrade

```
sysupgrade -n /tmp/openwrt-23.05.5-mediatek-filogic-zyxel_ex5601-t0-ubootmod-squashfs-sysupgrade.itb
```

The router will reboot into permanently installed OpenWrt.

---

## Verify

```bash
ssh root@192.168.1.1
cat /etc/openwrt_release
```

---

## Upgrading to a Newer OpenWrt Version

When upgrading in future, always use the **Zyxel EX5601-T0 (OpenWrt U-Boot layout)** image variant (`ubootmod`). Never use the stock image after ubootmod has been applied.

---

## Notes

- `mtk-uartboot` does **not** work on this device due to locked u-boot (eFuse blown)
- The `zyeng` Multiboot protocol approach was attempted but the router did not respond to multicast packets — root access was obtained via ZySH (`sys atck`) instead
- Always keep the root/supervisor password from `sys atck` stored safely — it may be needed for recovery
- The serial console remains useful as a fallback if SSH becomes unavailable

---

## References

- [OpenWrt forum thread — Adding OpenWrt support for Zyxel EX5601-T0](https://forum.openwrt.org/t/adding-openwrt-support-for-zyxel-ex5601-t0/155914)
- [dando's guide for locked u-boot](https://github.com/ndandanov/zyxel-ex5601-t0-openwrt/blob/main/guide/guide.md)
- [OpenWrt Wiki — ZyXEL T-56](https://openwrt.org/toh/zyxel/t-56)
- [bmork's zyxel-hacks (zyeng)](https://github.com/bmork/zyxel-hacks)
