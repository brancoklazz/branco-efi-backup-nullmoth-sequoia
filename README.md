# Hackintosh EFI — B850M + Ryzen 9 7900 + RTX 5070

Backup EFI OpenCore untuk build hackintosh harian. Sequoia 15.8.2, satu-satunya
GPU yang dipakai RTX 5070 lewat driver NullMoth.

**Backup diambil:** 11 November 2026

---

## Hardware

| Komponen | Model |
|---|---|
| Motherboard | ASRock B850M Steel Legend WiFi (AM5, chipset B850) |
| CPU | AMD Ryzen 9 7900 — 12 core / 24 thread |
| GPU | NVIDIA GeForce RTX 5070, 12 GB, `10DE:2F04` |
| RAM | 64 GB DDR5 |
| Ethernet | Realtek RTL8125 2.5 GbE, `10EC:8125` |
| WiFi / BT | Broadcom BCM94360, chipset `BCM_4350C2` |
| Audio | Realtek onboard (AppleALC) |
| Monitor | Lenovo R34w-30, 3440×1440 @ 180 Hz |

iGPU Ryzen (`1002:164E`) tidak dipakai — di-disable, semua display lewat RTX 5070.

## Software

| | |
|---|---|
| macOS | 15.8.2 Sequoia, build 24H232, Darwin 24.6.0 |
| OpenCore | DBG-109 (build 2026-10-31) |
| Driver GPU | NullMoth 2.9, firmware NVIDIA 620.70.06 |
| SMBIOS | MacPro7,1 |

## boot-args

```
keepsyms=1 debug=0x100 msgbuf=1048576 amfi_get_out_of_my_way=0x1 -amfipassbeta
ipc_control_port_options=0 gfxrst=4 nvfb=1 nvaccel=1 nvfbheads=4 -nvkmsnosmooth
nvfbhidpi=1 amfi=0x80
```

`gfxrst=4` (GPU reset), `nvfb=1 nvaccel=1 nvfbheads=4` (framebuffer + akselerasi +
4 head), `nvfbhidpi=1` (HiDPI framebuffer), `-nvkmsnosmooth` (matikan smoothing KMS
— ini yang menghilangkan flicker di 3440×1440 @180).

## Yang jalan

- Metal + Quartz compositing penuh, satu monitor 3440×1440 @180 Hz
- Hardware video decode VP9 / AV1
- Audio onboard, jack line-out + digital out
- Ethernet 2.5 GbE
- WiFi + Bluetooth (BCM94360)
- USB: semua port fisik terpetakan lewat `UTBMap.kext`

## Yang tidak jalan

- **NVENC / hardware encode** — tidak ada di Turing ke atas untuk macOS. Encode video pakai CPU.
- **H.264 / HEVC hardware decode RTX 50** — masih eksperimental, hasilnya video hijau di Safari. Sengaja tidak diaktifkan.
- **iGPU** — tidak dipakai, tidak ada output.
- **Rotation display** — tidak didukung driver.
- **AirDrop / iPhone Mirroring** — tidak jalan di BCM94360 dengan stack legacy.
- **Sleep** — dimatikan sengaja (`pmset sleep 0`), lihat catatan di bawah.

## Struktur

```
EFI/
├── BOOT/
│   └── BOOTx64.efi
└── OC/
    ├── ACPI/           SSDT-EC, SSDT-PLUG-ALT, SSDT-USBX
    ├── Drivers/        HfsPlus, OpenCanopy, OpenRuntime, ResetNvramEntry
    ├── Kexts/          19 kext (lihat tabel bawah)
    ├── Resources/      tema OpenCanopy
    ├── Tools/          NullMothSafe.efi
    ├── OpenCore.efi
    └── config.plist    ← config aktif
```

### Kext

| Kext | Versi | Fungsi |
|---|---|---|
| Lilu | 1.7.3 | patcher inti, wajib untuk semua |
| VirtualSMC | 1.3.9 | emulasi SMC |
| AMDRyzenCPUPowerManagement | 0.7.2 | power management Ryzen |
| SMCAMDProcessor | 1 | sensor CPU |
| SMCSuperIO | 1.3.8 | sensor fan/suhu (NCT6796D-S) |
| AppleMCEReporterDisabler | 1.2 | matikan MCE reporter yang bikin panic |
| IOSkywalkFamily | 1.0 | stack network legacy |
| IO80211FamilyLegacy | 1200.12.2b1 | driver WiFi BCM94360 |
| AMFIPass | 1.4.1 | lewati AMFI tanpa matikan SIP penuh |
| NMHDA | 1.0.0 | HDMI/DP audio di driver NullMoth |
| NVMeFix | 1.1.4 | power management NVMe |
| RestrictEvents | 1.1.7 | sembunyikan banner CPU tidak dikenal |
| RTL812xLucy | 1.1.302 | Ethernet RTL8125 |
| USBToolBox | 1.2.0 | base class untuk map USB |
| UTBMap | 1.2 | map 19 port dari USBToolBox Windows |
| WhateverGreen | 1.7.2 | framebuffer patch |
| AppleALC | 1.9.9 | audio onboard |
| BlueToolFixup | 2.7.3 | Bluetooth stack |

## Setelan picker & security

| Key | Nilai | Alasan |
|---|---|---|
| `ScanPolicy` | `984323` (`0xF0503`) | izinkan scan ESP + APFS, device dibatasi SATA/SAS/SCSI/NVMe |
| `SecureBootModel` | `Disabled` | tidak pakai Apple Secure Boot |
| `Vault` | `Optional` | tidak pakai vault signing |
| `DmgLoading` | `Signed` | recovery hanya dari image bertanda tangan |
| `ShowPicker` | `true` | picker selalu muncul |
| `Timeout` | `5` | |
| `HideAuxiliary` | `true` | entry tambahan (Windows, tool) sembunyi — tekan **Space** |
| `PickerMode` | `External` | OpenCanopy (grafis) |
| `UpdateSMBIOSMode` | `Custom` | SMBIOS dikirim apa adanya |

`ScanPolicy 984323` itu `0xF0103 | 0x400`. Bit `0x400` = izin scan ESP. Tanpa bit
itu, Windows tidak akan pernah muncul di picker meski `.contentVisibility` sudah benar.

## Jalan pulang kalau driver bikin macOS tidak mau start

Dua lapis:

1. **Dari picker** — restart, tekan **Space**, pilih:
   - `1401: Start once without NVIDIA driver` — sekali boot tanpa driver
   - `1401: Remove NVIDIA driver` — cabut permanen

   Keduanya dijalankan oleh `EFI/OC/Tools/NullMothSafe.efi`.

2. **Dari Recovery** (kalau picker pun tidak bisa):

   ```
   nvram "7C436110-AB2A-4BBB-A880-FE41995C9F82:nullmoth-remove=1"
   reboot
   ```

   Setelah masuk macOS: `sudo /Library/NullMoth/uninstall.sh`, lalu `nvram -d`
   untuk flag-nya.

   **Catatan:** config ini punya `Booter.Quirks.DisableVariableWrite = true`
   (dipasang 1401 untuk stabilitas AM5), jadi penulisan NVRAM dari macOS bisa
   ditolak. Jalur picker lebih bisa diandalkan.

## Cara restore

```bash
diskutil mount disk2s3                 # mount ESP
cp -R EFI/ /Volumes/EFI/               # timpa seluruhnya
```

Jangan pakai Finder drag-and-drop — atribut file `.efi` perlu utuh.

Setelah restore, cek:

```bash
plutil -lint /Volumes/EFI/EFI/OC/config.plist
```

## Catatan

- **Ini bukan EFI universal.** Spesifik untuk board, CPU, dan GPU di atas. Dipakai
  di hardware lain akan gagal boot.
- **File `config.plist*` lain di folder `OC/`** adalah snapshot historis waktu
  tuning (`before-*`, `bak-*`). Yang dipakai hanya `config.plist`. Sisanya arsip.
- `EFI/Boot/.contentVisibility = Disabled` dipasang di ESP **Windows** (disk lain),
  bukan di sini — mencegah entry "EFI" nyasar. Kalau EFI ini dipakai di mesin lain,
  file itu harus dibuat manual di ESP Windows.
- Driver NullMoth 2.x adalah rilis terakhir yang mendukung Sequoia dan Tahoe.
  macOS 27 ke atas tidak jalan di x86_64.
- **Jangan Sleep / Shutdown** saat driver aktif. Pakai **Restart**.
