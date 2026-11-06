# Jio 5G SA on a Canadian Galaxy S23 (SM-S911W)

This is a record of one successful repair on **7 November 2026**, not a guarantee for other models or firmware. It documents the observed settings, the single modem-policy change that was needed, and the checks that established a real 5G data connection.

## Device and starting state

| Item | Observed value |
| --- | --- |
| Phone | Samsung Galaxy S23, SM-S911W (Canadian variant) |
| CSC | XAC |
| Android | 17 |
| AP/CP build | S911WVLU8GZIF |
| Jio network | PLMN 405-862 |

The Jio SIM connected to 5G in another phone at the same location. On the S23, Android allowed NR in its network-type bitmask (`GPRS|EDGE|UMTS|HSDPA|HSUPA|HSPA|LTE|HSPA+|GSM|LTE_CA|NR`), power saving was off, and the phone was registered on Jio LTE. Before the fix, Samsung's `*#0011#` screen showed `Serving PLMN(405-862)-LTE`, LTE band 5, 10 MHz, RSRP -84 dBm, and no NR band or NR signal values. Android reported `LTE,Unknown`. Those facts narrowed the problem but did not by themselves identify a carrier rejection.

## 1. Establish ADB and USB access

1. Install Samsung's Android USB driver and Google's platform tools on the PC if they are not already available. Enable **USB debugging** in Developer options, connect the unlocked phone with a data-capable cable, and accept its USB debugging authorization prompt.
2. Check `adb devices -l`; it should show `device`, not `unauthorized` or an empty list. Confirm `adb shell getprop ro.product.model` returns `SM-S911W` before changing anything.
3. In the Samsung Phone dialer, `*#0808#` opens **USB Settings**. The initial mode used for SamFw's MTP read was **MTP + ADB**. The diagnostic mode was **DM + MODEM + ADB**. Changing the USB mode briefly disconnects and reconnects ADB; COM numbers can change.

On this PC, both Samsung diagnostic serial ports appeared in Windows with status **OK** after switching to DM + MODEM + ADB. The phone was returned to MTP + ADB before starting SamFw's MTP stage.

## 2. Enable Samsung's secret-code access

SamFw Tool **6.2** was obtained from the developer's [official release page](https://samfw.com/blog/samfwtool/). Its function **Samsung → MTP → Enable Secret code for Verizon** was run against the **SAMSUNG Mobile USB Modem** port, not a diagnostic serial port. Despite the name, the function was used on this XAC Canadian phone; it did not change the CSC.

One attempt selected a stale modem COM port and failed because that port did not exist. A later attempt selected the active modem port and completed these stages:

```text
Reading info mode MTP ... OK
Enable DIAG ... OK
Waiting for DIAG ... OK
Enabling secret code ... OK
Disable DIAG ... OK
```

If SamFw stops at **Waiting for DIAG**, switch the phone from MTP + ADB to **DM + MODEM + ADB** with `*#0808#`, then wait for Windows to detect the diagnostic serial ports. On this run SamFw proceeded through that stage and reported OK. The modem port number changed during the sequence, so the current device label mattered more than a remembered COM number.

`*#0011#` opened the basic network-status screen. The advanced dialer codes `*#73#` and `*#27663368378#` did not reliably open the advanced settings on this Android 17 build, even after SamFw succeeded. Direct ADB broadcasts of Samsung's secret code were rejected by Android's permission check. We did not bypass that permission check.

## 3. Open advanced ServiceMode

The working route was the developer-published [Samsung Toolkit v3.0.5](https://github.com/raywdude/samsung-toolkit/releases/tag/v3.0.5), with the [thedjchi Shizuku v13.7.0 fork](https://github.com/thedjchi/Shizuku/releases/tag/v13.7.0-thedjchi) for Android 17. Both APKs were installed on the phone. The owner accepted Samsung Toolkit's on-screen disclaimer and allowed its Shizuku access request.

Shizuku was started once over the already-authorized USB ADB connection using the exact **View command** shown by the Shizuku app. Its startup output reported `starting server` and a server PID. Shizuku gives approved apps ADB-level system-API access; the owner explicitly authorized this step. It needs to be started again after a phone reboot if Toolkit is needed later.

In Samsung Toolkit, **Service Menu - Unlock 4G & 5G bands** opened Samsung ServiceMode. The first launch displayed basic information. On a subsequent launch, **MAIN MENU2** appeared. The Toolkit's **GCF Protocol Menu** also opened a separate GCF configuration screen; no GCF option was changed.

## 4. Find and correct the actual block

From **MAIN MENU2**, the observed path was:

```text
UE SETTING & INFO
  → SETTING
  → PROTOCOL
  → NR5G
  → Stack Selection
  → STACK 1
```

ServiceMode confirmed **`STACK Selected : SUB 1`**, matching the active Jio SIM. Then, using ServiceMode's **More options → Back** to return to NR5G configuration:

```text
NR5G → SA/NSA Mode Control
```

The menu showed **`ACTIVE STACK = SUB 1`** and **`SA DISABLE (*)`**. This was the concrete block: Jio's mobile 5G service uses standalone NR, while SA was disabled on the active SIM stack.

We selected **`SA/NSA ENABLE`**. ServiceMode returned:

```text
SA/NSA ENABLED!!
Please reboot the UE
```

The phone was rebooted once to apply the change. **No NR5G pruning removal, NR-whitelist removal, SA permit-list removal, band-mask edit, GCF certification mode, or NR-only forcing was performed on this S23.** Those were unnecessary once the specific SA-disable setting was found.

## 5. Verify a real 5G connection

After reboot, Android reported `gsm.network.type = NR_SA,Unknown` on Jio PLMN `405862`, and the operator name changed to **Jio True5G**. Live telephony diagnostics showed voice and data in service with radio technology `NR_SA` and a connected data state (`2`). The physical-channel record identified **NR band 78**. This is stronger evidence than a 5G status-bar icon alone.

Wi-Fi was briefly turned off for an internet connectivity test. Three ICMP probes to `1.1.1.1` succeeded with **0% packet loss** and about **64 ms average round-trip time**. Wi-Fi was restored to its original on state. This was a connectivity check, not a download-speed measurement.

## What remains untested

Sustained throughput, incoming and outgoing calls, SMS, emergency calling, and persistence after future firmware updates were not tested. This result was observed on one SM-S911W with build `S911WVLU8GZIF`; other builds can expose different menus or policy defaults. If a future update resets the policy, inspect the active stack and SA/NSA mode again before changing anything.

No firmware was flashed, CSC was changed, bootloader was unlocked, phone was rooted, or IMEI/EFS/NV data was edited. The phone was returned to **MTP + ADB** USB mode and its home screen after diagnosis. The two helper apps remained installed; Shizuku's temporary server stopped on reboot.

This guide intentionally omits phone number, IMEIs, serial number, unique device IDs, and account information.
