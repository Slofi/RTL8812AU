# RTL8812AU WiFi driver for the Cyberdeck — patches only

> Created 2026-10-06 (S442, Claude) when the two 743 MB build trees `Projects/rtl8812au/` and
> `Projects/rtl8812au-patched/` were deleted (the owner's call). Everything below was verified live that day.

> ⚠️ **Superseded 2026-10-08:** the default driver on this box is now **rtw88 (`rtw_8812au`)** — see the companion repo
> [Slofi/RTL8812AU-new](https://github.com/Slofi/RTL8812AU-new). `88XXau` below stays installed (blacklisted for auto-load) and is
> swapped in automatically only for monitor mode. This repo is the backup of the old driver's patch + DKMS config.

## What actually runs on CD
- Module **`88XXau`**, loaded automatically from
  `/lib/modules/6.18.44-current-rockchip64/updates/dkms/88XXau.ko`.
- DKMS package **`realtek-rtl88xxau/5.6.4.2~20230501`**, source **`/usr/src/realtek-rtl88xxau-5.6.4.2~20230501/`**.
  That source was placed by hand (NOT from an apt package — `dpkg -S` finds nothing, no apt hold needed): its
  `Makefile` is dated 2026-05-03 (= the LIVE patch), `dkms.conf` 2026-08-30. DKMS rebuilds from it on kernel updates
  (needs `linux-headers-current-rockchip64`).
- Nothing on CD referenced the deleted Projects trees.
- ⚠️ `/usr/src` on CD is **not backed up** — **this folder is the backup of the patch.**

## The patches (both against upstream `aircrack-ng/rtl8812au`, branch `v5.6.4.2`, commit `7344855`, 2026-03-17 — still upstream HEAD on 2026-10-06)
| File | What | Status |
|---|---|---|
| `rtl8812au-kernel6.15-compat-LIVE-2026-05-03.patch` | from `rtl8812au-patched/`: `Makefile` `EXTRA_CFLAGS`→`ccflags-y` + `$(M)` include paths · `include/osdep_service_linux.h` `from_timer`/`del_timer_sync` compat macros (KERNEL_VERSION ≥ 6.15) · `os_dep/linux/ioctl_cfg80211.c` `radio_idx` param in cfg80211_ops callbacks | **THE ONE RUNNING.** Verified: fresh upstream clone @ `7344855` + this patch = the 3 DKMS source files on CD, **byte-identical** (sha256 `32b60e60…` / `05054d7e…` / `351b0878…`) |
| `rtl8812au-kernel6.15-compat-ALT-2026-05-23.patch` | from `rtl8812au/`: same 3 files, a later different approach (direct renames `timer_container_of`/`timer_delete_sync`, `$(src)`/`$(srctree)` paths, `-Wno-error`) | **NOT used.** Kept because it is the only copy |

Both patches were proven to `git apply` cleanly to a fresh upstream clone and to reproduce their trees byte-for-byte.

## Files here
- the two `.patch` files (above) · **`dkms.conf`** — byte-copy of CD's `/usr/src/realtek-rtl88xxau-5.6.4.2~20230501/dkms.conf`
  (sha256 `3afad813…`, 2026-10-06): `MAKE=… CC=gcc-14` pin + `AUTOINSTALL="yes"`. It is the second hand-made piece; `/usr/src`
  on CD is not backed up.

## Rebuilding from scratch (e.g. CD reinstalled) — the S412 procedure (2026-08-30, `cyberdeck.md`)
```bash
sudo apt install -y linux-headers-current-rockchip64 gcc-14     # BOTH needed — the Armbian kernel is built with gcc-14,
                                                                # CD's default gcc-13 rejects its flags (breaks ANY out-of-tree module)
git clone --branch v5.6.4.2 https://github.com/aircrack-ng/rtl8812au.git
cd rtl8812au && git checkout 7344855
git apply /home/slofi/Projects/cyberdeck/rtl8812au/rtl8812au-kernel6.15-compat-LIVE-2026-05-03.patch
sudo cp -a . /usr/src/realtek-rtl88xxau-5.6.4.2~20230501
sudo cp /home/slofi/Projects/cyberdeck/rtl8812au/dkms.conf /usr/src/realtek-rtl88xxau-5.6.4.2~20230501/dkms.conf
sudo dkms add -m realtek-rtl88xxau -v 5.6.4.2~20230501
sudo dkms install -m realtek-rtl88xxau -v 5.6.4.2~20230501
modinfo -n 88XXau                                               # must end in …/updates/dkms/88XXau.ko
```
*(These commands are RECONSTRUCTED from S412's prose record — "installed headers + gcc-14, copied the patched tree to /usr/src/…, pinned CC=gcc-14 in dkms.conf, registered it with DKMS" — and were NOT re-run in S442. What S442 verified: the patch reproduces the running source byte-for-byte, and `dkms.conf` here is a byte-copy of the live one.)*

**Before ANY kernel upgrade (card pre-flight):** `dkms status` lists the module · `linux-headers` installed · the gcc in
`cat /proc/version` matches the `CC=` pin — **bump the pin if the kernel moves to gcc-15+**.
