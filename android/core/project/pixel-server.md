---
description: "Pixel 7 Pro small footprint Linux server via Termux + Alpine - fixed IP 10.0.0.50, SSH 8022/2222, KernelSU boot"
tags:
  - "pixel"
  - "server"
  - "termux"
  - "alpine"
  - "ssh"
  - "fixed-ip"
created: "2026-08-20"
updated: "2026-08-20"
---

# Pixel 7 Pro Server (cheetah)

Small footprint Linux server on Pixel 7 Pro, kept on LineageOS via Termux + Alpine chroot. All configured via `adb` from WSL.

## Hardware Dump (2026-08-20 via adb)

- **Model:** Pixel 7 Pro `cheetah` / `google/cheetah:16/BP2A.250805.005` / Lineage `userdebug` (build 2026-02-03)
- **SoC:** Google Tensor G2 `GS201` `gs201` / `arm64-v8a` / 8-core (4x `0xd05` A55 + 2x `0xd41` A78 + 2x X1)
- **RAM:** `11711108 kB` (~12GB LPDDR5) / `MemAvailable 8087720 kB`
- **Storage:** `/data 110G` free `105G` (5% used) / UFS 3.1
- **Kernel:** `6.1.157-android14-11-g8d713f9e8e7b` / `aarch64 Toybox` / Bootloader `cloudripper-16.0-13291549`
- **Bootloader:** `orange` `flash.locked=0` `verifiedbootstate=orange` / **unlocked**
- **adb:** `33251FDH3001D5` `unauthorized -> device` + `adbd root` (`uid 0`) via KernelSU (`ksu`)
- **Termux:** `0.118.3` `/data/data/com.termux/files` / UID `10233` `u0_a233` / GIDs `[3003]` inet required for DNS
- **Battery:** `31%` USB powered `10.0.0.1` / `health 2 GOOD` / `4832000/5002000 mAh` 96.6% / `3893mV` `28.7°C` / `status 2` `Charging 1`

## Fixed IP (robust via factory MAC)

- **SSID:** `Home` `BSSID 50:88:11:c0:94:58` / `WPA_PSK` `WelcomeHome` / `5GHz 5240MHz` `11ax` `~1921Mbps`
- **Before:** `66:ad:7c:35:61:cd` randomized `MacRandomization 3 AUTO` / DHCP `10.0.0.36/37`
- **After (adb):**
  ```bash
  cmd wifi forget-network 0
  cmd wifi add-network 'Home' wpa2 WelcomeHome -r none
  cmd wifi connect-network 'Home' wpa2 WelcomeHome -r none
  # + sed /data/misc/apexdata/com.android.wifi/WifiConfigStore.xml MacRandomization 0
  svc wifi disable; svc wifi enable
  ```
- **Now:** `ac:3e:b1:b1:70:24` factory `MacRandomization 0` / `wlan0 UP` `10.0.0.50/24` `GW 10.0.0.1` `DNS 10.0.0.1` / `DNS 1.1.1.1`
- **Router:** `10.0.0.1` -> `DHCP reservation ac:3e:b1:b1:70:24 = 10.0.0.50` (was .37 sticky, now .50 via `svc wifi disable/enable` + lease 86400)
- **WifiConfigStore.xml:** `MacRandomizationSetting 0` / `RandomizedMacAddress 66:ad...` (ignored, live uses factory)

> For true fixed IP, keep Android on DHCP + router reservation - more robust than Android STATIC (WifiConfigStore STATIC via `IpAssignment STATIC` was tested via python+sed but reverted - live DHCP+MAC reservation is canonical).

## Termux + Alpine Setup (all via adb, no phone typing)

### Termux base

- `su -G 3003 10233` required (inet group 3003) - plain `su 10233` loses DNS (`socket.gaierror 7`)
- `export PREFIX=/data/data/com.termux/files/usr; PATH=$PREFIX/bin:$PATH; HOME=/data/data/com.termux/files/home`
- DNS fix: `echo "nameserver 10.0.0.1" > $PREFIX/etc/resolv.conf` (was 8.8.8.8, blocked for untrusted_app without inet)
- `pkg update` -> mirror `termux.cdn.lumito.net` ok
- `pkg install -y proot-distro openssh termux-services` -> `sshd` keys RSA/ECDSA/ED25519 generated
- `proot-distro install alpine` -> `3.24.1` `5de55e5ef9c0` 4 MiB - **proot bug**: `CANNOT LINK proot: libandroid_shmdt_fd` missing in `libandroid-shmem 0.7` (new lib has `libandroid_shmdt` without `_fd`). Bypassed via direct `chroot` as root.
- `pkg upgrade -y` -> 91 packages, libandroid-shmem 0.7 upgrade, but proot still fails - use `chroot`

### Alpine via chroot (root, no proot)

Path: `/data/data/com.termux/files/usr/var/lib/proot-distro/containers/alpine/rootfs`

```bash
chroot /.../alpine/rootfs /sbin/apk add openssh nginx python3 bash shadow  # + linux-pam 6 pkgs
/usr/sbin/chpasswd <<< "root:alpine123"  # via /usr/sbin/chpasswd (shadow)
busybox sed -i 's/^#Port.*/Port 2222/' /etc/ssh/sshd_config
busybox sed -i 's/^#PermitRootLogin.*/PermitRootLogin yes/' /etc/ssh/sshd_config
busybox sed -i 's/^#PasswordAuthentication.*/PasswordAuthentication yes/' /etc/ssh/sshd_config
chown root:root /var/empty; chmod 711 /var/empty
mkdir -p /run/sshd
ssh-keygen -A
/usr/sbin/sshd -p 2222  # LISTEN 0.0.0.0:2222
```

### Termux sshd

```bash
printf "termux123\ntermux123\n" | passwd  # set
su -G 3003 10233 -c 'PREFIX=...; sshd'  # LISTEN 0.0.0.0:8022
ssh-keygen -t ed25519 -f /tmp/termux_test -N '' -C wsl-to-pixel
cat /tmp/termux_test.pub > ~/.ssh/authorized_keys (600)
cat /tmp/termux_test.pub > /.../alpine/rootfs/root/.ssh/authorized_keys (600)
```

Tested:
```
ssh -i /tmp/termux_test -p 8022 u0_a233@10.0.0.50  -> hello from Termux / uid=10233
ssh -i /tmp/termux_test -p 2222 root@10.0.0.50  -> (needs PermitRootLogin fix, use Termux+su chroot)
ssh -p 8022 u0_a233@10.0.0.50 "su -c 'chroot .../alpine/rootfs cat /etc/alpine-release'" -> 3.24.1 (via /system/bin/su)
```

### Robust passive (boot survives)

- Whitelist: `cmd deviceidle whitelist +com.termux`; `cmd appops set com.termux RUN_IN_BACKGROUND allow`; `am set-standby-bucket com.termux active`
- Wake lock: `termux-wake-lock` + Settings -> Battery Unrestricted + Developer Stay awake when charging
- **Termux:Boot** script (if app installed): `~/.termux/boot/00-robust.sh` (179B):
  ```sh
  #!/data/data/com.termux/files/usr/bin/sh
  termux-wake-lock
  sshd
  su -c "chroot /data/data/com.termux/.../alpine/rootfs /usr/sbin/sshd -p 2222"
  ```
- **KernelSU** service (always runs, Termux:Boot not required): `/data/adb/service.d/99-robust-pixel.sh` + `/data/adb/ksu/service.d/` (549B):
  ```sh
  #!/system/bin/sh
  sleep 15
  su -G 3003 10233 -c 'PREFIX=...; termux-wake-lock; sshd'
  chroot /.../alpine/rootfs /usr/sbin/sshd -p 2222
  ```
  Created via `adb shell` as root, `chmod 755`.
- Verified: `ss -tulpn | grep -E '8022|2222'` both LISTEN, `ps | grep sshd` 11485+11525

## Access

- **WLAN:** `10.0.0.50` (factory MAC reservation)
- **Termux:** `ssh -i /tmp/termux_test -o StrictHostKeyChecking=no -p 8022 u0_a233@10.0.0.50` or `ssh -p 8022 u0_a233@10.0.0.50` pass `termux123`
- **Alpine:** `ssh -i /tmp/termux_test -p 2222 root@10.0.0.50` pass `alpine123` or `ssh -p 8022` then `/system/bin/su -c "chroot .../alpine/rootfs /bin/sh"`
- **ADB:** `33251FDH3001D5` `Pixel 7 Pro` `2-2 18d1:4ee7` via `C:\Users\n0ne\scoop\shims\adb.exe` or `~/android/platform-tools/adb` (WSL via `usbipd` not needed - `Not shared` but adb bridges)
- **Key:** `/tmp/termux_test` (WSl) `ed25519 AAAAC3... wsl-to-pixel` - keep, also in Termux `~/.ssh/authorized_keys`

## Gotchas hit

- `lsusb` 1 on WSL, no `/dev/bus/usb`, use `powershell usbipd list` + `Get-PnpDevice USB\VID`
- Charge-only cable vs data cable -> no `18d1:4ee7` in `usbipd`
- `unauthorized` -> tap Allow RSA on phone
- `proot` `libandroid_shmdt_fd` missing after `termux upgrade` -> use `chroot` as root (KernelSU) instead
- `su 10233` loses `inet` (3003) -> DNS `No address` -> use `su -G 3003 10233`
- `WifiConfigStore.xml` edits overwritten if wifi not disabled -> `svc wifi disable` before `sed`/`python`, `svc wifi enable` after
- `python3` for edit is at `/data/data/com.termux/files/usr/bin/python3` (700, need `chmod 755` for root)

## Next uses

- `apk add tailscale` inside Alpine for `100.x` fixed outside Home
- `nginx -t` OK, `python3 --version` 3.14.7
- Attach USB-OTG Ethernet for wired + power, keep `Protect battery 80%`
