# Decoding an E-marker into Speed class and Power rating

Resolves [#3](https://github.com/ricklkiwi/usbc-tester/issues/3) on the map [#1](https://github.com/ricklkiwi/usbc-tester/issues/1).
Researched 2026-10-11.

## Answer in brief

- macOS prints each VDO as **raw bytes in wire order**. Read each 4-byte blob as a **little-endian `UInt32`**.
- **Speed class** comes from bits `2:0` of the Passive Cable VDO or Active Cable VDO1 ("USB Highest Speed").
  - `000` USB 2.0 · `001` Gen1 (5 Gb/s) · `010` Gen2 (10/20 Gb/s) · `011` USB4 Gen3 (40 Gb/s) · `100` USB4 Gen4 (80 Gb/s).
  - The Intel `0x8087` Thunderbolt cable mode can add TB3 20/40 Gb/s.
- **Power rating** comes from three fields:
  - bits `6:5`, VBUS current: `01` = 3 A, `10` = 5 A;
  - bits `10:9`, maximum VBUS voltage;
  - bit `17`, EPR Mode Capable, which is PD r3.1 and later.
  - The result is 60 W (3 A), 100 W (5 A, SPR), or 240 W (5 A, EPR-capable, 50 V field).
- **Unmarked cable** (C-to-C): treat the Power rating as 3 A, meaning 60 W at 20 V. The spec only allows a USB 2.0 cable to be unmarked, so treat it as **USB 2.0**. Label both values as assumed.
- **The machine's example cable** is a Passive Cable, Type-C to Type-C, with 20–30 ns latency (about 3 m). Its E-marker needs VCONN and claims 20 V, 5 A, USB 2.0 only, and no EPR. **Claimed capability:** Speed class USB 2.0, Power rating 100 W. VID, XID and PID are all 0, so the E-marker is not registered with USB-IF.

## Sources and trust

| Tag | Source | Role |
|---|---|---|
| [PD] | USB-IF *USB Power Delivery Specification* r3.0 v2.0, r3.1, r3.2: §6.4.4.3.1 Discover Identity, Tables 6-33…6-44 (ID Header, Cert Stat, Product, Passive Cable VDO, Active Cable VDO1/2) | Primary. Not machine-fetchable here (behind a zip on usb.org). Field text is quoted through the two headers below, which copy the spec tables verbatim. |
| [EC] | Chromium EC `include/usb_pd_vdo.h`: https://github.com/coreboot/chrome-ec/blob/main/include/usb_pd_vdo.h | Verbatim copy of the PD r3.x tables, including `100b = USB4 Gen4` and the PD 3.2 Enter_USB DO. Fetched 2026-10-11. |
| [LX] | Linux `include/linux/usb/pd_vdo.h`: https://github.com/torvalds/linux/blob/master/include/linux/usb/pd_vdo.h | Cross-check for every VDO layout. Fetched 2026-10-11. |
| [TBT] | Linux `include/linux/usb/typec_tbt.h`: https://github.com/torvalds/linux/blob/master/include/linux/usb/typec_tbt.h | TBT3 cable Discover Mode VDO (Intel SVID 0x8087). Written by Intel (Heikki Krogerus). The Intel TBT3 alt-mode spec itself is not public. |
| [TC] | USB-IF *USB Type-C Cable and Connector Specification* r2.x §3 (cable types) and §4.9 (Electronically Marked Cables) | Primary for Unmarked-cable rules. Corroborated by TI SSZTBJ8 (https://www.ti.com/lit/pdf/ssztbj8). |
| [IFX] | Infineon KBA "Enable EPR mode for EMCA application": https://community.infineon.com/t5/Knowledge-Base-Articles/Enable-extended-power-range-EPR-mode-for-EMCA-application/ta-p/288728 | Shows that "EPR mode capable" is a Cable VDO field. |
| [TD] | tomorrow56 teardown of a Daiso "USB3" cable: https://note.com/tomorrow56/n/n1a43a5001aa8 | Real-world fake E-marker (VID, XID and PID all 0, claims inconsistent). Secondary or anecdotal. |

**Confidence notes.**
- **Bit 17 = EPR Mode Capable:** both open-source headers predate EPR and still mark bit 17 Reserved. The field's existence is confirmed by [IFX] and [TD], which quotes the spec wording "Cable is not EPR Mode Capable". Its position, B17, comes from PD r3.1 Table 6-41 and is not checked against a second machine-readable source. **Medium-high confidence.**
- **Active VDO2 bit 1 (USB4 Asymmetric, PD 3.2):** medium confidence.
- Everything else matches across [EC] and [LX].

---

## 1. Byte order of the macOS IORegistry blobs (stated explicitly)

macOS shows `Metadata` → `VDOs` as `CFData` 4-byte blobs, for example `<00000018>`. Each blob holds the data object's bytes **in the order they came off the CC wire**. USB PD sends every 32-bit data object **least-significant byte first** [PD §6.2]. So:

> **Decode rule: `UInt32(littleEndian:)` over each 4-byte blob.**
> `<00000018>` → **`0x18000000`** · `<40680800>` → **`0x00086840`**.

Evidence that this is right, not just the endianness of Apple silicon:

| Interpretation | ID Header → Product Type (bits 29:27) | Cable VDO → current (6:5), VDO ver (23:21) | Consistent with macOS's own `"Product Type Description"="Passive Cable"`? |
|---|---|---|---|
| **Little-endian** (`0x18000000`, `0x00086840`) | `011` = **Passive Cable** | `10` = 5 A, ver `000` = 1.0 (only legal passive value) | **Yes** |
| Big-endian (`0x00000018`, `0x40680800`) | `000` = Not a cable plug; VID = 0x0018 | `00` = *Reserved, Shall Not be used*; ver `011` (illegal for passive) | No |

Swift:
```swift
let raw: UInt32 = data.withUnsafeBytes { $0.loadUnaligned(as: UInt32.self) }
let vdo = UInt32(littleEndian: raw)   // explicit; correct on any host
```

VDO array order in the Discover Identity ACK [PD Table 6-32; LX VDO_INDEX_*]:
- macOS drops the VDM Header, so its array starts at **[0] ID Header**.
- Then come **[1] Cert Stat**, **[2] Product VDO** and **[3] Cable VDO** (Passive Cable VDO, or Active Cable VDO1).
- An active cable adds **[4] Active Cable VDO2** (PD r3.0+).
- A PD r2.0 E-marker sends 4 VDOs with the older r2.0 Cable VDO layout (§4).

## 2. ID Header VDO [PD Table 6-33; LX; EC]

| Bits | Field | Values |
|---|---|---|
| 31 | USB Communications Capable as USB Host | 0 for cables |
| 30 | USB Communications Capable as USB Device | 0 for cables |
| 29:27 | **Product Type (UFP / SOP′ Cable Plug)** | On SOP′: `000` not a Cable Plug/VPD · `011` **Passive Cable** · `100` **Active Cable** · `110` VCONN-Powered USB Device (VPD) · others reserved |
| 26 | Modal Operation Supported | 1 means the Cable has SVIDs/modes (e.g. TBT3 `0x8087`), so do a SOP′ Discover SVIDs |
| 25:23 | Product Type (DFP) | SOP only; `000` on a Cable |
| 22:21 | Connector Type (PD r3.1+) | `00` = not set (PD r3.0 or earlier field was reserved) · `10` receptacle · `11` plug. A cable plug should report `11` under r3.1+ |
| 20:16 | Reserved | 0 |
| 15:0 | **USB Vendor ID** | USB-IF-assigned VID of the cable maker. **0x0000 is not an assigned VID** (§9) |

## 3. Cert Stat VDO and Product VDO [PD Tables 6-34, 6-35; LX]

| VDO | Bits | Field |
|---|---|---|
| Cert Stat | 31:0 | **XID**: USB-IF-assigned compliance ID from certification. 0 means not certified or not registered. |
| Product | 31:16 | USB Product ID (from the VID owner) |
| Product | 15:0 | bcdDevice |

## 4. Passive Cable VDO [PD r3.x Table 6-41; EC; LX]

| Bits | Field | Encoding |
|---|---|---|
| 31:28 | HW Version | vendor-assigned |
| 27:24 | FW Version | vendor-assigned |
| 23:21 | **VDO Version** | `000` = **v1.0** (the only legal value for Passive in PD r3.0 v2+/r3.1/r3.2; others reserved). In PD r3.0 v1.x, bits 23:20 were reserved, so they still read as 000. |
| 20 | Reserved | 0 |
| 19:18 | Plug type | `10` USB Type-C · `11` Captive · (`00`/`01` were Type-A/B in PD r2.0; reserved in r3.x) |
| 17 | **EPR Mode Capable** (PD r3.1+) | `0` not EPR · `1` EPR (240 W) capable. Reserved (0) in PD r3.0. |
| 16:13 | Cable Latency | `0001` <10 ns (~1 m) · `0010` 10–20 ns (~2 m) · `0011` 20–30 ns (~3 m) · `0100` ~4 m · `0101` ~5 m · `0110` ~6 m · `0111` ~7 m · `1000` >70 ns (>~7 m) · others reserved |
| 12:11 | Cable Termination | `00` VCONN not required · `01` VCONN required · `1x` reserved |
| 10:9 | **Maximum VBUS Voltage** | `00` **20 V** · `01` 30 V · `10` 40 V · `11` **50 V**. PD r3.1+ marks 30 V and 40 V as deprecated; EPR cables report 50 V. |
| 8:7 | Reserved | 0 |
| 6:5 | **VBUS Current Handling** | `00` reserved · `01` **3 A** · `10` **5 A** · `11` reserved |
| 4:3 | Reserved | 0 |
| 2:0 | **USB Highest Speed** | see §6 |

**PD r2.0 Cable VDO** (for old E-markers that answer with SVDM version 1.0) [LX]:
- bits 10:7 were SSTX/SSRX directionality;
- bit 4 was "VBUS through cable" and bit 3 was "SOP″ controller present";
- bits 2:0 were "USB SS signalling": `000` USB 2.0, `001` USB 3.1 Gen1, `010` USB 3.1 Gen2.

Current is at the same bits 6:5, and the speed codes 0–2 mean the same, so one decoder covers both. Use the SVDM version in the VDM header, or bit 17 being 0, only for the reserved-bit sanity checks.

## 5. Active Cable VDO1 and VDO2 [PD r3.x Tables 6-42/6-43, 6-44; EC; LX]

**VDO1** has the same layout as the Passive Cable VDO, except:

| Bits | Field | Encoding |
|---|---|---|
| 23:21 | VDO Version | `011` = **v1.3** (PD r3.0 v2+/r3.1/r3.2). Other values are reserved; older PD r3.0 active cables may show `000`. |
| 17 | EPR Mode Capable (PD r3.1+) | as Passive |
| 16:13 | Cable Latency | `0001`–`0111` as Passive · `1000` 1000 ns (~100 m) · `1001` 2000 ns (~200 m) · `1010` 3000 ns (~300 m) |
| 12:11 | Cable Termination | `10` one end active, VCONN required · `11` both ends active, VCONN required |
| 8 | SBU Supported | `0` supported · `1` not supported |
| 7 | SBU Type | `0` passive · `1` active |
| 6:5 | VBUS Current | **only valid if bit 4 = 1**: `00` USB Type-C default current · `01` 3 A · `10` 5 A |
| 4 | **VBUS Through Cable** | `0` no VBUS (e.g. some optical cables): **Power rating = none** · `1` yes |
| 3 | SOP″ Controller Present | |
| 2:0 | USB Highest Speed | §6 |

**VDO2** (only on active cables). Note the **inverted** "Supported" bits, where 0 means supported:

| Bits | Field | Encoding |
|---|---|---|
| 31:24 | Max operating temperature (°C) | |
| 23:16 | Shutdown temperature (°C) | |
| 14:12 | U3/CLd power | `000` >10 mW … `110` <50 µW |
| 11 | U3→U0 transition | `0` direct · `1` through U3S |
| 10 | Physical connection | `0` copper · `1` optical |
| 9 | Active element | `0` re-driver · `1` re-timer |
| 8 | **USB4 Supported** | **`0` = yes**, `1` = no |
| 7:6 | USB 2.0 hub hops consumed | |
| 5 | **USB 2.0 Supported** | **`0` = yes**, `1` = no |
| 4 | **USB 3.2 Supported** | **`0` = yes**, `1` = no |
| 3 | USB lanes | `0` one lane · `1` two lanes |
| 2 | Optically isolated | `0` no · `1` yes |
| 1 | USB4 Asymmetric Mode Supported (PD r3.2) | reserved before r3.2 |
| 0 | USB Gen | `0` Gen1 · `1` Gen2 or higher (VDO1 bits 2:0 give the exact gen) |

## 6. USB Highest Speed → Speed class [PD r3.x; EC `enum usb_rev30_ss`]

| Bits 2:0 | Spec meaning | Speed class (max, two-lane where applicable) |
|---|---|---|
| `000` | USB 2.0 only, no SuperSpeed | **USB 2.0** (480 Mb/s) |
| `001` | USB 3.2 / USB4 Gen1 | **5 Gb/s** (USB 3.2 Gen1; Gen1x2 = 10 Gb/s) |
| `010` | USB 3.2 / USB4 Gen2 | **10 Gb/s per lane**: USB 3.2 Gen2 10 Gb/s, Gen2x2 20 Gb/s, USB4 20 Gb/s |
| `011` | USB4 Gen3 | **USB4 40 Gb/s** |
| `100` | USB4 Gen4 (PD r3.2) | **USB4 80 Gb/s** (120 Gb/s asymmetric if VDO2 bit 1 = 1) |
| `101`–`111` | Reserved | **Invalid**: treat as a misreporting E-marker |

The same 3-bit code is reused in the Enter_USB Data Object (bits 23:21) that a host sends to enter USB4 [EC, PD r3.2 Table 6-50]. That gives one place where the Claimed speed directly drives the Observed link.

Active cables: AND the speed code with the VDO2 support flags (USB4 Supported, USB 3.2 Supported, USB 2.0 Supported). Some active or optical cables carry **no USB 2.0** (VDO2 bit 5 = 1), so a USB-2-only device will not enumerate through them.

## 7. VBUS current, voltage, EPR → Power rating

Rule (for a Cable with VBUS, i.e. Passive, or Active with VDO1 bit 4 = 1):

```
I = {01: 3 A, 10: 5 A, else: invalid}        (Active 00 = Type-C default → treat as 3 A, assumed)
if EPR(bit17)==1 and I==5 A and Vmax==50 V:  Power rating = 240 W  (EPR, 48 V × 5 A)
else:                                        Power rating = 20 V × I  → 60 W or 100 W (SPR)
```

- **SPR tops out at 20 V** in PD. A Max VBUS of 30 V or 40 V (deprecated legacy codes) does not give a higher SPR rating.
- **EPR (PD r3.1+)** needs all three: the cable is EPR Mode Capable, 5 A, and 50 V max. A source only enters EPR after confirming this through the cable's SOP′ Discover Identity [PD r3.1 §6.4.10; IFX].
- **Inconsistent combinations are tells of a bad E-marker**, e.g. 50 V with bit 17 = 0 [TD], or EPR = 1 with 3 A.
- EPR fixed voltages are 28 V (140 W), 36 V (180 W) and 48 V (240 W). The cable rating is still "240 W".

## 8. Thunderbolt 3 / 4 / 5 cables [TBT; PD §6.4.4.3.2–3]

How a Thunderbolt cable identifies itself:
- ID Header bit 26 (Modal) = 1.
- **SOP′ Discover SVIDs** includes **`0x8087` (Intel)**.
- SOP′ Discover Modes on 0x8087 returns the **TBT3 Cable Discover Mode VDO**:

| Bits | Field | Encoding |
|---|---|---|
| 0 | TBT mode | identifies the TBT3 mode |
| 18:16 | **Cable speed** | `001` USB 3.x Gen1 cable · `010` 10 Gb/s (USB 3.x Gen2 passive) · `011` **10 and 20 Gb/s** per lane, i.e. a **TB3 40 Gb/s** link |
| 20:19 | Rounded/gen support | `00` TBT Gen3 non-rounded only · `01` Gen3 and Gen4 rounded and non-rounded |
| 21 | Optical | |
| 22 | Re-timer | |
| 23 | Link training (unidirectional LSRX) | |
| 25 | Active (1) / Passive (0) | |

How this maps to the Speed class:
- **TB3 passive 40 Gb/s cables** (≤0.8 m) usually report USB Highest Speed `010` (Gen2) in the PD Cable VDO. Their **TBT cable speed `011`** is the only sign that they carry 40 Gb/s in TBT3 mode.
- So the Speed class is **max(USB speed from Cable VDO, TBT speed from 0x8087 mode)**, with TBT `011` meaning "Thunderbolt 3, 40 Gb/s". Without reading the 0x8087 mode, a TB3 40 Gb/s cable looks like a 10/20 Gb/s cable.
- **Thunderbolt 4** cables:
  - no TB4-specific bit exists in any PD VDO;
  - TB4 is an Intel certification of a **USB4 Gen3** cable (`011`, 40 Gb/s) that also exposes the 0x8087 mode with speed `011`;
  - "TB4" can only be inferred, or tied to a certified XID.
- **Thunderbolt 5** cables are, in the same way, **USB4 Gen4** cables (`100`, 80 Gb/s, PD r3.2). There is no separate TB5 field in the public sources.
- **Report the Speed class as the USB4 / TB3 rate, never as "TB4" or "TB5", unless certification is checked.**

## 9. Unmarked cable defaults [TC §3, §4.9]

An **Unmarked cable** is one where SOP′ Discover Identity gets no GoodCRC or response. The rules are:

- **Current:** a cable without an E-marker must be treated as **3 A** max, which is 60 W at 20 V. Any C-to-C cable rated above 3 A (5 A) **shall** be electronically marked. A PD source shall not offer more than 3 A without reading a 5 A E-marker.
- **Speed:** every **USB Full-Featured Type-C cable** (SuperSpeed or USB4 pairs) **shall** be electronically marked. So the only *compliant* Unmarked C-to-C cable is a **USB 2.0 Type-C cable**, and its default Claimed capability is **USB 2.0, 60 W (assumed)**.
- **Caveat:** non-compliant unmarked cables with SuperSpeed wiring exist. Hosts may still train a 5/10 Gb/s link over them, because USB 3.x link training does not check the E-marker. USB4/TBT hosts, however, need a USB4/TBT E-marker before entering USB4/TBT. So an Unmarked cable's **Observed** speed can exceed its assumed USB 2.0 claim.
  - Report this as "Observed above unmarked default", not as a pass or fail.
  - Legacy Type-A-to-C cables never carry an E-marker; that is allowed (TC Legacy Cable section) and does not apply to Loopback on C ports.

## 10. Vendor ID meaning and fake or misreporting E-marker tells

The ID Header VID is the **cable maker's** USB-IF VID. It may be the E-marker chip vendor's VID only when they license it.
- `0x0000` is not assigned.
- An XID of 0 means no USB-IF certification record.

Tells, roughly from strongest to weakest:
1. **VID = 0 and XID = 0 and PID = 0.** The E-marker was never provisioned. The chip's default config was shipped, so its claims are generic. [TD] shows such a cable claiming 10 Gb/s and 50 V/5 A on a 5 Gb/s product.
2. **Reserved or illegal codes:** current `00`/`11` on a passive cable, speed `101`–`111`, latency `0000`, plug type `00`/`01` under PD r3, or the wrong VDO version (Passive ≠ `000`, Active ≠ `011`).
3. **Internal inconsistencies:**
   - Max VBUS 50 V with EPR = 0, or EPR = 1 without 5 A or 50 V;
   - an Active Product Type with only 4 VDOs;
   - VDO1 speed ≥ USB4 while VDO2 says USB4 not supported;
   - Passive Gen3/Gen4 with latency above ~1 m. USB-IF certifies passive USB4 Gen3 only to about 0.8 m, and passive USB 3.2 Gen2 to about 1 m [TC Table 5-1 per EC comment].
4. **Claimed capability above Observed capability** on Loopback, e.g. claimed Gen2 but only USB 2.0 trains. This is the app's main mismatch flag.
5. Nothing in the E-marker proves the copper can carry 5 A. Proving the Power rating under load is out of scope (#1).

## 11. Worked example: this machine's cable

IORegistry: `VDOs = (<00000018>, <00000000>, <00000000>, <40680800>)`, `"Product Type Description"="Passive Cable"`, Vendor ID 0.

| VDO | Bytes → LE `UInt32` | Decode |
|---|---|---|
| ID Header | `00 00 00 18` → `0x18000000` = `0001 1000 0000 …` | Host 0, Device 0, **Product Type 011 = Passive Cable**, Modal 0 (no SVIDs, so not a TBT cable), DFP 000, Connector Type 00 (not set, PD r3.0-style), **VID 0x0000** |
| Cert Stat | `0x00000000` | **XID 0**: not certified |
| Product | `0x00000000` | PID 0x0000, bcdDevice 0x0000 |
| Passive Cable VDO | `40 68 08 00` → `0x00086840` = `0000 0000 0000 1000 0110 1000 0100 0000` | see below |

`0x00086840` field by field:

| Bits | Value | Meaning |
|---|---|---|
| 31:28 HW | 0 | |
| 27:24 FW | 0 | |
| 23:21 VDO ver | `000` | v1.0 (valid for Passive) |
| 20 | 0 | |
| 19:18 plug | `10` | USB Type-C to Type-C |
| 17 EPR | 0 | Not EPR capable |
| 16:13 latency | `0011` | 20–30 ns, ~3 m (hint: it's ~2–3 m long) |
| 12:11 term | `01` | VCONN required |
| 10:9 Vmax | `00` | 20 V |
| 8:7 | 0 | |
| 6:5 current | `10` | **5 A** |
| 4:3 | 0 | |
| 2:0 speed | `000` | **USB 2.0 only** |

**Claimed capability: Speed class USB 2.0 (480 Mb/s); Power rating 100 W (20 V × 5 A, SPR, no EPR).**
- The E-marker is unregistered (VID, XID and PID all 0), but every field is internally consistent and uses legal codes.
- It matches the Observed USB 2 Loopback link recorded on #1.

## Open questions surfaced

- Does macOS expose the cable's **SOP′ Discover SVIDs / Discover Modes** responses (the 0x8087 TBT cable mode VDO) anywhere in the IORegistry? Without them, a TB3 40 Gb/s passive cable reads as Gen2 (§8).
- Does macOS show the **SVDM version** or PD spec revision used with SOP′? This is needed to tell a PD r2.0 Cable VDO from an r3.x one when checking reserved bits strictly.
- What does the IORegistry show for an **Unmarked cable**: no `IOPortTransportComponentCCUSBPDSOPp` node, or an empty one? This decides how "Unmarked" is detected.
