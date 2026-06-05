# Build HG680P OpenWrt 25.12 Kernel 6.12 dengan Wi-Fi Auto ON

Dokumen ini merangkum hasil inspeksi fork `0xultraos/amlogic-s9xxx-openwrt`, jalur build `ophub/amlogic-s9xxx-openwrt`, dan firmware OpenWrt yang sedang berjalan di STB HG680P.

Status kerja:

- `gh auth status` berhasil. Akun aktif: `0xultraos`.
- Repo sudah di-clone ke `amlogic-s9xxx-openwrt`.
- `origin` sudah benar menunjuk ke `https://github.com/0xultraos/amlogic-s9xxx-openwrt.git`.
- `upstream` sudah ada dan menunjuk ke `https://github.com/ophub/amlogic-s9xxx-openwrt.git`.
- Tidak ada push ke GitHub.
- Tidak ada firmware build panjang yang dijalankan.

## Ringkasan Rekomendasi

- HG680P didukung oleh database perangkat ophub.
- Board/profile build ophub yang benar adalah `s905x`.
- DTB/FDT yang benar adalah `meson-gxl-s905x-p212.dtb`.
- Platform adalah `amlogic`, family `meson-gxl`, boot config `uEnv.txt`.
- Target OpenWrt yang tersedia pada workflow repo adalah `openwrt:25.12.4`.
- Target kernel yang tersedia pada workflow repo adalah `6.12.y`; `ophub/kernel` tag `kernel_stable` saat dicek memuat seri `6.12.x` sampai `6.12.92`.
- Wi-Fi aktif pada firmware lama memakai chipset Realtek SDIO dengan ID `024C:F179`, driver kernel `8189fs`, bukan `brcmfmac` dan bukan `rtw88_8822cs`.
- Wi-Fi auto ON pada firmware lama terjadi karena `/etc/config/wireless` sudah berisi konfigurasi AP aktif tanpa `option disabled '1'`, ditambah modul `8189fs` diautoload lewat `/etc/modules.d/8189fs`.
- Risiko utama untuk OpenWrt 25.12 + kernel 6.12 adalah ketersediaan `8189fs.ko` di paket kernel `ophub/kernel`. Repo build saat ini tidak memilih paket OpenWrt standar bernama `kmod-rtl8189fs`, dan config OpenWrt resmi yang terlihat tidak menyediakan opsi tersebut untuk `openwrt_main`.

## Analisis Firmware Saat Ini via SSH

Target SSH:

```sh
ssh root@192.168.1.10
```

Firmware yang sedang berjalan:

```text
DISTRIB_ID='OpenWrt'
DISTRIB_RELEASE='24.10.4'
DISTRIB_REVISION='r28959-29397011cc'
DISTRIB_TARGET='armsr/armv8'
DISTRIB_ARCH='aarch64_generic'
DISTRIB_DESCRIPTION='OpenWrt 24.10.4 r28959-29397011cc'
```

Kernel:

```text
Linux OpenWRT 6.6.117-MirageOS+ #4 SMP PREEMPT Sat Nov 29 12:29:55 WITA 2025 aarch64 GNU/Linux
```

Board:

```json
{
  "model": "HG680P ✧ s905x",
  "board_name": "amlogic,p212",
  "target": "armsr/armv8"
}
```

Deteksi wireless dari `/etc/board.json`:

```text
phy0 path: platform/soc/d0000000.apb/d0070000.mmc/mmc_host/mmc0/mmc0:0001/mmc0:0001:1
band: 2G
max_width: 20
modes: NOHT
```

Konfigurasi `/etc/config/wireless`:

```text
config wifi-device 'radio0'
	option type 'mac80211'
	option path 'platform/soc/d0000000.apb/d0070000.mmc/mmc_host/mmc0/mmc0:0001/mmc0:0001:1'
	option band '2g'
	option channel '11'
	option txpower '20'
	option cell_density '0'
	option country 'ID'
	option htmode 'HT20'

config wifi-iface 'default_radio0'
	option device 'radio0'
	option network 'lan'
	option mode 'ap'
	option ssid 'HG680P'
	option encryption 'psk-mixed'
	option key '<redacted>'
```

Catatan penting: tidak ada `option disabled '1'`, sehingga radio dan AP aktif setelah service wireless berjalan.

Status interface:

```text
phy#0
	Interface phy0-ap0
		ssid HG680P
		type AP
```

`iwinfo`:

```text
phy0-ap0  ESSID: "HG680P"
Mode: Master
Channel: 11 (2.462 GHz)
HT Mode: NOHT
Encryption: mixed WPA/WPA2 PSK (CCMP)
Hardware: 024C:F179 0000:0000 [Generic MAC80211]
```

Driver/chipset:

```text
rtl8189fs mmc0:0001:1 phy0-ap0: renamed from wlan0
```

`modinfo 8189fs`:

```text
filename: /lib/modules/6.6.117-MirageOS+/kernel/drivers/net/wireless/realtek/rtl8189fs/8189fs.ko
version: v5.7.9_35795.20191128
description: Realtek Wireless Lan Driver
alias: sdio:c*v024CdF179*
depends: cfg80211
intree: Y
```

Autoload module:

```text
/etc/modules.d/8189fs: 8189fs
```

Modul Wi-Fi terkait yang ter-load:

```text
8189fs
cfg80211
mac80211
rtw88_8822cs
rtw88_8822c
rtw88_sdio
rtw88_core
brcmfmac
brcmutil
ath10k_usb
ath10k_sdio
ath10k_core
mt7601u
```

Modul yang benar-benar mengikat perangkat internal HG680P adalah `8189fs`. Modul lain ada di firmware tetapi tidak menjadi driver Wi-Fi internal yang aktif.

Paket Wi-Fi terkait yang terpasang:

```text
hostapd-common
iw
iwinfo
kmod-cfg80211
kmod-mac80211
libiwinfo-data
libiwinfo-lua
libiwinfo20230701
rpcd-mod-iwinfo
wifi-scripts
wireless-regdb
wpad-basic-mbedtls
```

`opkg search` tidak menemukan paket pemilik `8189fs.ko`. Ini mengindikasikan `8189fs.ko` kemungkinan berasal dari paket kernel/custom kernel ophub/MirageOS, bukan paket OpenWrt standar yang bisa dipilih langsung dari `.config`.

## Cara Wi-Fi Auto ON Bekerja Saat Ini

Mekanismenya:

1. Kernel mendeteksi SDIO card pada `mmc0:0001`.
2. Modul `8189fs` diload otomatis dari `/etc/modules.d/8189fs`.
3. Driver membuat interface awal `wlan0`, lalu OpenWrt/mac80211 menamainya menjadi `phy0-ap0`.
4. `/etc/config/wireless` sudah ada dan berisi AP aktif:
   - `mode 'ap'`
   - `network 'lan'`
   - `ssid 'HG680P'`
   - `encryption 'psk-mixed'`
   - tidak ada `disabled`.
5. Init OpenWrt menjalankan `/sbin/wifi config` jika `/etc/config/wireless` belum ada, dan service network/wireless mengaktifkan konfigurasi yang ada.

Tidak ditemukan mekanisme khusus di:

- `/etc/uci-defaults/`
- `/etc/hotplug.d/` selain default `ieee80211/10-wifi-detect`
- `/etc/rc.local` untuk Wi-Fi

`/etc/rc.local` hanya menjalankan:

```sh
bash /etc/custom_service/start_service.sh
sleep 5 && /usr/bin/k6hgled-r
sleep 5 && /usr/bin/k6hgledon-r
exit 0
```

## Analisis Repository ophub

Database perangkat:

```text
make-openwrt/openwrt-files/common-files/etc/model_database.conf
```

Baris HG680P:

```text
105:HG680P:s905x:meson-gxl-s905x-p212.dtb:u-boot-p212.bin:NA:NA:2+8G,100Mb-Nic:stable/all:amlogic:meson-gxl:uEnv.txt:rapdodge:s905x:yes
```

Makna field penting:

- Model: `HG680P`
- SOC: `s905x`
- FDTFILE: `meson-gxl-s905x-p212.dtb`
- U-Boot overload: `u-boot-p212.bin`
- Kernel tags: `stable/all`
- Platform: `amlogic`
- Family: `meson-gxl`
- Boot config: `uEnv.txt`
- Board: `s905x`
- Build: `yes`

Jalur build packaging:

```text
remake
```

Urutan overlay rootfs:

1. `make-openwrt/openwrt-files/common-files`
2. `make-openwrt/openwrt-files/platform-files/amlogic`
3. `make-openwrt/openwrt-files/different-files/s905x`

Karena itu, lokasi aman untuk override khusus HG680P/s905x adalah:

```text
make-openwrt/openwrt-files/different-files/s905x/rootfs/
```

## OpenWrt 25.12 dan Kernel 6.12

Workflow ImageBuilder:

```text
.github/workflows/build-openwrt-using-imagebuilder.yml
```

Default workflow:

```text
releases_branch: openwrt:25.12.4
openwrt_kernel: 6.12.y_6.18.y
```

Untuk target yang diminta, gunakan:

```text
releases_branch: openwrt:25.12.4
openwrt_board: s905x
openwrt_kernel: 6.12.y
auto_kernel: true
kernel_repo: ophub/kernel
kernel_usage: stable
```

Ketersediaan yang sudah dicek:

- OpenWrt ImageBuilder `25.12.4` untuk `armsr/armv8` tersedia.
- Direktori resmi OpenWrt `25.12.4/targets/armsr/armv8/` memuat:
  - `openwrt-imagebuilder-25.12.4-armsr-armv8.Linux-x86_64.tar.zst`
  - `openwrt-25.12.4-armsr-armv8-rootfs.tar.gz`
  - `openwrt-25.12.4-armsr-armv8.manifest`
  - `profiles.json`
- `ophub/kernel` tag `kernel_stable` memuat seri `6.12.x`; saat dicek, versi tertinggi yang terlihat adalah `6.12.92.tar.gz`.

Catatan: `remake` dengan `auto_kernel=true` akan mencari versi terbaru dalam seri `6.12.y`, bukan memaksa angka tertentu. Jika ingin pin tepat ke satu versi, gunakan `-a false -k 6.12.xx`.

## Hasil Riset Kernel 6.12 untuk Wi-Fi HG680P

Archive kernel `ophub/kernel` sudah dicek langsung:

```text
kernel_stable/6.12.92.tar.gz
kernel_stable/6.12.81.tar.gz
```

Isi `modules-6.12.92-ophub.tar.gz` dan `modules-6.12.81-ophub.tar.gz` tidak mengandung:

```text
8189fs.ko
rtl8189fs
CONFIG_RTL8189FS
```

Modul Realtek yang ada pada `6.12.92-ophub` antara lain:

```text
rtw88_8822cs.ko
rtw88_sdio.ko
rtw88_core.ko
rtw88_8723cs.ko
rtl8xxxu.ko
rtlwifi.ko
```

Modul tersebut bukan driver yang dipakai Wi-Fi internal HG680P ini. Firmware yang bekerja sudah membuktikan perangkat internal memakai:

```text
sdio:c*v024CdF179*
8189fs.ko
rtl8189fs
```

Kesimpulan:

- Kernel `6.12.y` bawaan `ophub/kernel` saat ini belum support Wi-Fi internal HG680P.
- Build firmware `openwrt:25.12.4 + kernel_stable/6.12.y` tanpa patch kernel akan boot, tetapi Wi-Fi internal HG680P sangat mungkin tidak muncul.
- Yang harus dibuat lebih dulu adalah kernel `6.12.y` custom yang membawa `8189fs.ko`.

Config kernel ophub:

```text
kernel-config/release/stable/config-6.12
```

Config ini sudah punya prasyarat:

```text
CONFIG_CFG80211=m
CONFIG_MAC80211=m
CONFIG_MMC=y
CONFIG_MMC_MESON_GX=y
CONFIG_MMC_MESON_MX_SDIO=y
```

Tetapi belum punya:

```text
CONFIG_RTL8189FS=m
```

Sebagai pembanding, config kernel lama ophub masih punya opsi ini:

```text
kernel-config/release/stable/config-5.10: CONFIG_RTL8189FS=m
kernel-config/release/stable/config-5.15: CONFIG_RTL8189FS=m
```

Sumber driver yang paling relevan:

```text
https://github.com/jwrdegoede/rtl8189ES_linux/tree/rtl8189fs
```

Alasan memilih sumber ini:

- Dipakai oleh instruksi HG680P komunitas.
- Dipakai oleh repo out-of-tree `alive4ever/rtl8189fs-armbian-current-meson64`.
- Repo tersebut secara eksplisit menyebut bahwa sejak kernel `6.12.22-current`, `8189fs` tidak lagi ikut paket kernel dan perlu dibangun out-of-tree.
- Branch `rtl8189fs` sudah memiliki commit fix build untuk kernel baru sampai `6.17`, sehingga lebih mungkin kompatibel dengan `6.12.y` dibanding source lama.

## Cara Membuat Kernel 6.12 Support Wi-Fi HG680P

Ada dua jalur teknis.

### Jalur A: Kernel Custom Terintegrasi

Ini jalur yang paling cocok untuk firmware baru karena modul `8189fs.ko` ikut masuk ke archive kernel dan otomatis dipaketkan oleh `remake`.

Langkah konseptual:

1. Fork atau gunakan repo kernel custom berbasis:

```text
https://github.com/ophub/kernel
```

2. Tambahkan patch driver `rtl8189fs` untuk source `linux-6.12.y` di:

```text
kernel-patch/beta/linux-6.12.y/
```

3. Patch harus menambahkan driver ke kernel tree, misalnya:

```text
drivers/net/wireless/realtek/rtl8189fs/
drivers/net/wireless/realtek/Kconfig
drivers/net/wireless/realtek/Makefile
```

4. Tambahkan opsi config:

```text
CONFIG_RTL8189FS=m
```

ke:

```text
kernel-config/release/stable/config-6.12
```

5. Build kernel memakai workflow:

```text
Compile mainline beta kernel
```

Input yang disarankan:

```text
kernel_source   = unifreq
kernel_version  = 6.12.y
kernel_auto     = true
kernel_package  = all
kernel_config   = kernel-config/release/stable
kernel_patch    = kernel-patch/beta
auto_patch      = true
kernel_toolchain = gcc
```

Output akan di-upload ke release:

```text
kernel_beta
```

6. Setelah kernel custom selesai, verifikasi asset release kernel custom:

```sh
gh release download kernel_beta --repo <user-or-org>/<kernel-repo> --pattern '6.12.*.tar.gz' --dir /tmp/kernel-test
tar -tzf /tmp/kernel-test/6.12.*.tar.gz | grep modules
tar -xzf /tmp/kernel-test/6.12.*.tar.gz -C /tmp/kernel-test
tar -tzf /tmp/kernel-test/6.12.*/modules-6.12.*.tar.gz | grep 8189fs
```

Yang harus ada:

```text
kernel/drivers/net/wireless/realtek/rtl8189fs/8189fs.ko
```

7. Build OpenWrt HG680P dengan repo kernel custom:

```text
openwrt_board  = s905x
openwrt_kernel = 6.12.y
kernel_repo    = <user-or-org>/<kernel-repo>
kernel_usage   = beta
```

Jika menjalankan `remake` lokal:

```sh
sudo ./remake -b s905x -k 6.12.y -a true -r <user-or-org>/<kernel-repo> -u beta
```

Catatan: `remake` mengubah `-u beta` menjadi tag release `kernel_beta`.

### Jalur B: Modul Out-of-Tree

Ini cocok untuk eksperimen pada sistem berjalan, tetapi kurang ideal untuk first boot firmware baru.

Referensi:

```text
https://github.com/alive4ever/rtl8189fs-armbian-current-meson64
```

Konsepnya:

1. Build `8189fs.ko` memakai header kernel yang sama persis dengan kernel target.
2. Salin modul ke:

```text
/lib/modules/<kernel>/kernel/drivers/net/wireless/realtek/rtl8189fs/8189fs.ko
```

3. Jalankan:

```sh
depmod -a
modprobe 8189fs
```

4. Tambahkan:

```text
/etc/modules.d/8189fs
```

Kekurangan jalur ini:

- Modul harus cocok dengan `vermagic` kernel.
- Tidak nyaman untuk first boot image.
- Jika OpenWrt image langsung boot tanpa modul, Wi-Fi tetap belum ON.
- Untuk firmware final, jalur A lebih bersih.

## Rekomendasi Final untuk Target Ini

Urutan yang paling aman:

1. Buat kernel custom `6.12.y` dulu sampai terbukti `8189fs.ko` ada.
2. Setelah kernel custom valid, baru buat overlay OpenWrt untuk Wi-Fi auto ON.
3. Build firmware `openwrt:25.12.4` board `s905x` memakai kernel custom tersebut.
4. Test boot dari USB/TF terlebih dahulu sebelum install/update eMMC.

Jangan memulai build firmware OpenWrt sebelum kernel `8189fs.ko` terverifikasi, karena build firmware tanpa driver ini hampir pasti tidak memenuhi target Wi-Fi auto ON.

## File yang Perlu Diubah

Belum ada file build yang diubah selain dokumen ini. Untuk port Wi-Fi auto ON secara minimal dan board-specific, file yang direkomendasikan adalah:

```text
make-openwrt/openwrt-files/different-files/s905x/rootfs/etc/modules.d/8189fs
make-openwrt/openwrt-files/different-files/s905x/rootfs/etc/config/wireless
```

Isi `etc/modules.d/8189fs`:

```text
8189fs
```

Contoh `etc/config/wireless`:

```text
config wifi-device 'radio0'
	option type 'mac80211'
	option path 'platform/soc/d0000000.apb/d0070000.mmc/mmc_host/mmc0/mmc0:0001/mmc0:0001:1'
	option band '2g'
	option channel '11'
	option txpower '20'
	option cell_density '0'
	option country 'ID'
	option htmode 'HT20'

config wifi-iface 'default_radio0'
	option device 'radio0'
	option network 'lan'
	option mode 'ap'
	option ssid 'HG680P'
	option encryption 'psk2'
	option key '<set-your-own-password>'
```

Rekomendasi keamanan:

- Jangan commit password Wi-Fi pribadi ke repo publik.
- Gunakan placeholder di repo, atau buat file overlay lokal yang tidak dipush.
- Jika ingin perilaku persis seperti firmware lama, encryption lama adalah `psk-mixed`, tetapi untuk build baru lebih aman memakai `psk2` kecuali perangkat klien lama membutuhkan WPA1.

## Paket/Modul yang Perlu Ada

Wajib:

```text
kmod-cfg80211
kmod-mac80211
wpad-basic-mbedtls atau wpad-basic
iw
iwinfo
wireless-regdb
wifi-scripts
8189fs.ko
```

Firmware lama tidak menunjukkan file firmware khusus untuk `rtl8189fs`; driver `8189fs` biasanya memakai firmware/parameter bawaan driver. Yang benar-benar wajib diverifikasi pada kernel baru adalah keberadaan:

```text
/lib/modules/<kernel>/kernel/drivers/net/wireless/realtek/rtl8189fs/8189fs.ko
```

Jika `8189fs.ko` tidak ada dalam kernel `6.12.x` dari `ophub/kernel`, Wi-Fi internal HG680P tidak akan auto ON walaupun `/etc/config/wireless` sudah benar.

## Command Build yang Direkomendasikan

Build lokal penuh tidak direkomendasikan dijalankan di Termux. Jalur yang paling sesuai dengan repo ini adalah GitHub Actions ImageBuilder.

Workflow:

```text
.github/workflows/build-openwrt-using-imagebuilder.yml
```

Input workflow:

```text
releases_branch = openwrt:25.12.4
openwrt_board   = s905x
openwrt_kernel  = 6.12.y
auto_kernel     = true
kernel_repo     = ophub/kernel
kernel_usage    = stable
openwrt_ip      = 192.168.1.1
```

Command lokal untuk packaging jika rootfs sudah ada:

```sh
sudo ./remake -b s905x -k 6.12.y -a true
```

Command lokal untuk ImageBuilder rootfs, jika dijalankan di environment Linux yang sesuai:

```sh
sudo ./config/imagebuilder/imagebuilder.sh openwrt:25.12.4
sudo ./remake -b s905x -k 6.12.y -a true
```

Catatan: `remake` membutuhkan operasi mount/loop device dan biasanya memerlukan Linux host/runner, bukan Termux biasa.

## Risiko dan Batasan

1. `8189fs.ko` belum diverifikasi ada di kernel `6.12.x`.
   - Firmware saat ini memakai kernel custom `6.6.117-MirageOS+` yang memiliki `8189fs.ko`.
   - Repo OpenWrt config tidak menunjukkan paket standar `kmod-rtl8189fs`.
   - Jika `ophub/kernel` `6.12.x` tidak memasukkan driver ini, perlu patch/build kernel atau memilih kernel yang sudah membawa `8189fs`.

2. `wifi status` pada firmware lama mengeluarkan warning:

```text
WARNING: Variable 'wlan\r' does not exist or is not an array/object
```

   AP tetap aktif, tetapi warning ini mengindikasikan ada data board/wireless JSON yang kurang rapi. Jangan jadikan output `wifi status` satu-satunya indikator sukses; cek juga `iw dev` dan `iwinfo`.

3. HG680P memakai DTB generic P212:

```text
meson-gxl-s905x-p212.dtb
```

   Ini cocok dengan board saat ini (`amlogic,p212`), tetapi variasi hardware HG680P bisa berbeda. Driver yang sudah diverifikasi di unit ini adalah `8189fs` dengan SDIO ID `024C:F179`.

4. Menyalin `/etc/config/wireless` akan membuat Wi-Fi aktif pada first boot, tetapi SSID/password akan menjadi default firmware.
   - Untuk repo publik, jangan simpan password pribadi.
   - Untuk firmware pribadi, boleh membuat overlay lokal sebelum build.

## Rekomendasi Langkah Berikutnya

1. Verifikasi isi kernel `6.12.x` dari `ophub/kernel` apakah memuat `8189fs.ko`.
2. Jika ada, buat overlay board-specific:
   - `make-openwrt/openwrt-files/different-files/s905x/rootfs/etc/modules.d/8189fs`
   - `make-openwrt/openwrt-files/different-files/s905x/rootfs/etc/config/wireless`
3. Jalankan build via GitHub Actions dengan:
   - `openwrt:25.12.4`
   - `s905x`
   - `6.12.y`
4. Setelah flash/test boot, validasi:

```sh
uname -a
ubus call system board
lsmod | grep 8189fs
dmesg | grep -Ei '8189|wlan|sdio|cfg80211'
iw dev
iwinfo
uci show wireless
```

5. Jangan push perubahan overlay Wi-Fi yang berisi password pribadi.
