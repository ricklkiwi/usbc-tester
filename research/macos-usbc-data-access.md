# How macOS exposes USB-C port, PD and E-marker data to an app

Resolves [#2](https://github.com/ricklkiwi/usbc-tester/issues/2) for the map [#1](https://github.com/ricklkiwi/usbc-tester/issues/1).
Researched 2026-10-11 on a MacBook Pro M2 Max (Mac14,5), macOS 26.6.2 (25G83), with one Cable in Loopback between port 1 and port 3.

## Answer in one paragraph

Everything comes from the IORegistry. A per-port controller node (`AppleTCControllerType10` on M1/M2) has child services that describe what the port is doing: CC/PD (`IOPortTransportStateCC` → `IOPortTransportComponentCCUSBPDSOP` / `…SOPp` / `…SOPpp`) and each data transport (`IOPortTransportStateUSB2`, `…USB3`, `…CIO`, `…DisplayPort`). An unprivileged, unsandboxed Swift process reads them with **public IOKit C APIs** (`IOServiceGetMatchingServices`, `IORegistryEntryCreateCFProperties`) and needs **no root and no entitlement**; the live probe below ran as uid 501. Plug and unplug events arrive through `IOServiceAddMatchingNotification` (first-match and terminated) on the transport and SOP′ classes, plus `IOServiceAddInterestNotification` (`kIOGeneralInterest`, which carries `kIOMessageServicePropertyChange`) on the port node. The **class names and property keys are undocumented** and depend on the chip generation, so the port-reading layer must treat them as a moving target.

## Key facts

### 1. Classes and properties (live evidence, this Mac)

From `ioreg -l -w0 -r -n "Port-USB-C@3"` and `ioreg -c AppleTCControllerType10`:

| Node (class) | What it holds | Key properties seen |
| --- | --- | --- |
| `Port-USB-C@N` (`AppleTCControllerType10`, provider `AppleHPMDevice`, driver `com.apple.driver.AppleHPM`) | Port state | `PortNumber`, `ConnectionActive`, `TransportsSupported` (`CC, USB2, USB3, CIO, DisplayPort`), `TransportsProvisioned`, `TransportsActive`, `TransportsUnauthorized`, `IOAccessoryUSBConnectString` (`Host` / `Device` / `None`, describing the **partner**), `IOAccessoryUSBSuperSpeedActive`, `PlugOrientation`, `ActiveCable`, `OpticalCable`, `Plug Event Count`, `ConnectionUUID`, `AuthorizationRequired`, `UserAuthorizationPending` |
| `CC` (`IOPortTransportStateCC`) | Type-C/PD channel state | `Active`, `Metadata` = {`Vendor ID (SOP)`, `Product ID (SOP)`} |
| `SOP` (`IOPortTransportComponentCCUSBPDSOP`) | Port partner's Discover Identity | `Specification Revision`, `Vendor ID`, `Product ID`, `Metadata.VDOs` (array of 4-byte `Data`) |
| `SOP'` (`IOPortTransportComponentCCUSBPDSOPp`) | **The Cable's E-marker** (near end) | `Product Type` / `Product Type Description` (`Passive Cable`), `Metadata.VDOs`, `Metadata.VDO Count` |
| `SOP''` (`IOPortTransportComponentCCUSBPDSOPpp`) | Far-end E-marker, if any | Same shape (per WhatCable README; not present on this Cable) |
| `USB2` / `USB3` / `CIO` / `DisplayPort` (`IOPortTransportState*`) | One per data transport: **the Observed link** | `Active`, `DataRole[Description]`, `Generation[Description]`, `DataRate[Description]`, `NominalSignalingFrequenciesHz`, plus `AuthorizationStatus`, `TRM_*` |
| `Power In` / `USB-PD` (`IOPortFeaturePowerIn`, `IOPortFeaturePowerSource`) | Source PDOs when the Mac is charging | (not needed for v1 Power rating) |

**VDO byte order.** Each VDO is stored little-endian. Port 3's SOP′ `VDOs` were `<00000018> <00000000> <00000000> <40680800>`, which read as ID Header `0x18000000` (Product Type 3, passive cable), Cert Stat `0`, Product `0`, and Cable VDO `0x00086840`. Decoding that by the USB PD 3.x Passive Cable VDO layout gives speed `000` (USB 2.0 only), VBUS current `10` (5 A), and max VBUS `00` (20 V). That matches the map's first evidence.

**SOP′ only shows up on the VCONN-source side.** Port 3 had a SOP′ node and port 1 did not; only the port sourcing VCONN talks to the E-marker. In Loopback, read the E-marker from whichever end has it, not from a fixed end.

### 2. Public API status, permissions and sandbox

- The calls are public IOKit (`IOKitLib.h` in the macOS 26 SDK): `IOServiceMatching`, `IOServiceGetMatchingServices`, `IORegistryEntryCreateCFProperties`, `IOServiceAddMatchingNotification`, `IOServiceAddInterestNotification`, `IONotificationPortCreate` and `IONotificationPortSetDispatchQueue`. `kIOMessageServicePropertyChange` is defined in `IOMessage.h`.
- The `IOPort*` and `AppleTCController*` **classes are not declared in any SDK header**. The only public trace is in `IOUSBHostFamilyDefinitions.h`, where the XHCI port properties `UsbIOPort` and `UsbTransportState` hold registry paths to the "IOPort service" and the "IOPortTransportState service". So the API is public but the schema is undocumented.
- The probe ran **as uid 501, with no entitlements and no root**. It read every property and registered all eight notifications (`kr=0`). WhatCable reports the same: "No entitlements, no private APIs, no helper daemons". It also reports that the IOKit reads come back identical inside the App Sandbox. That doesn't matter for us, but it confirms there is no hidden privilege gate.
- Intel Macs publish no port-controller data (WhatCable: "every Intel Mac in the community diagnostic corpus reports its USB-C port-controller services as empty"). That is fine because v1 is Apple silicon only.

### 3. Plug and unplug notifications

- The `IOPortTransportState{USB3,CIO,USB2}` and `IOPortTransportComponentCCUSBPDSOP*` services are **created when a link comes up and terminated when it goes down**. WhatCable's `USB3TransportWatcher` says `IOPortTransportStateUSB3` services "appear dynamically when a USB 3 SuperSpeed device is connected and disappear on unplug", and on this Mac's idle ports no USB3 or CIO nodes exist at all. Matching notifications (`kIOMatchedNotification`/`kIOFirstMatchNotification` plus `kIOTerminatedNotification`) on these classes are therefore the plug and unplug signal.
- The port node itself is permanent. Register `IOServiceAddInterestNotification(…, kIOGeneralInterest, …)` on each `AppleTCControllerType10` (or `AppleHPMInterfaceType*`) to receive `kIOMessageServicePropertyChange` when `ConnectionActive`, `TransportsActive` and similar change. This is WhatCable's `AppleHPMInterfaceWatcher.registerInterest` pattern. Re-read all properties on any callback.
- **The E-marker read lags the plug.** WhatCable shows a "Reading cable details…" state for about 5 s after connect, so the app should wait for the SOP′ node to settle before giving a verdict.
- Always drain the iterator once after registering, or the notification never arms. `IOServiceAddMatchingNotification` consumes the matching dictionary's reference.

### 4. Where the Observed link speed lives

| Link | Where | Live value on this Mac |
| --- | --- | --- |
| USB 2 | `IOPortTransportStateUSB2.DataRateDescription` / `DataRate` / `NominalSignalingFrequenciesHz` | Port 1: `Active=Yes`, `480 Mbps (High Speed)`, `[240000000]`, role `Device` |
| USB 3.x | `IOPortTransportStateUSB3`: `Generation` (1 = Gen 1, 5 Gb/s; 2 = Gen 2, 10 Gb/s), `Active`, and port-level `IOAccessoryUSBSuperSpeedActive` | Not present (no USB 3 link). The class name was confirmed in the macOS 26.6.2 dyld shared cache, and key semantics come from WhatCable's `USB3Transport` |
| USB4 / Thunderbolt (CIO) | `IOPortTransportStateCIO`: `CableSpeed` (2 = 20, 3 = 40, 4 = 80 Gb/s, which WhatCable calls a "floor on cable capability" rather than the trained rate), `LinkTrainingMode`, `Generation`. The **trained lane** is on `IOThunderboltPort` lane adapters (`Adapter Type`=1) under `IOThunderboltSwitchType5`: `Current Link Speed` (code 0x8 = 10, 0x4 = 20, 0x2 = 40 Gb/s per lane), `Current Link Width`, `Target Link Speed/Width`, `Link Bandwidth`, and `Socket ID` = receptacle | Idle lanes read `Current Link Speed=8`, `Current Link Width=1` (WhatCable: idle ports look like this, so gate on CIO `Active`). system_profiler shows the per-receptacle buses as "Up to 40 Gb/s", "No device connected" |
| DisplayPort | `IOPortTransportStateDisplayPort` | n/a |

**Read both ends.** In this Loopback the device-side USB2 node (port 1) was `Active=Yes` at 480 Mb/s, while the host-side node (port 3) was `Active=No`, `DataRateDescription=None`, and port 3's `TransportsActive` was only `("CC")`. No USB device enumerated on any XHCI bus (`system_profiler SPUSBHostDataType` showed only empty buses). The Observed capability for USB 2 therefore has to be taken from whichever end reports an active transport.

### 5. Stability across macOS versions and chips

- **The port-node class changes with the chip, not the OS.** M1/M2 use `AppleTCControllerType10/11`, M3-era Macs use `AppleHPMInterfaceType10/11/12`, and A18 Pro (MacBook Neo) uses `AppleHPMInterfaceType18`. They share the superclass `IOPort`, and WhatCable also matches on `IOPort` as a catch-all, filtering on `PortTypeDescription` and a `Port-` prefix. The `IOPortTransport*` child classes are common to all of them.
- **Port numbers are not unique across port types.** MagSafe 3 (`ParentPortType=17`) and HDMI (`ParentPortType=6`, which has its own `IOPortTransportStateDisplayPort`) are also port 1, while USB-C ports are `ParentPortType=2`. Key ports by (`ParentPortType`, `ParentPortNumber`) or by registry entry ID.
- WhatCable requires macOS 14 or later, which is evidence that the schema has held from 14 through 26. Apple publishes no guarantee. The `TRM_*`/`Authorization*` keys are newer and tie into the accessory-security feature.
- WhatCable (darrylmorley/whatcable, about 8.9k stars, active on 2026-10-11) is the most thorough prior art. It is a Swift menu-bar app reading these exact classes, built from a corpus of about 771 machines. Its license is "Other" (custom), so check it before copying any code. Cable Detective (App Store, closed source) reads the same data.

### 6. Accessory security can block the Loopback data path

Apple silicon laptops gate the data of new accessories behind System Settings → Privacy & Security → "Allow accessories to connect". The options are Always ask, Ask for new accessories (the **default**), Automatically when unlocked, and Always allow. With "Don't Allow", the Mac won't recognize the accessory, but **power still flows** ([Apple Support 102282](https://support.apple.com/en-us/102282)). This Mac's transports show `TRM_ProfileDescription = "Always Allow"`, `AuthorizationStatusDescription = "Policy Authorized"`. Under the default setting a Loopback could hold its data transports in `TransportsUnauthorized` / `UserAuthorizationPending` until the owner approves. CC/PD and the E-marker read are unaffected, because `CC` showed `AuthorizationRequired = No`.

## Throwaway probe (settles "no root" and "notifications register")

`swiftc -O probe.swift -o probe && ./probe 2`, run as a normal user:

```swift
import Foundation
import IOKit

func props(_ s: io_object_t) -> [String: Any] {
    var u: Unmanaged<CFMutableDictionary>?
    guard IORegistryEntryCreateCFProperties(s, &u, kCFAllocatorDefault, 0) == KERN_SUCCESS else { return [:] }
    return u!.takeRetainedValue() as! [String: Any]
}
func each(_ cls: String, _ body: (io_object_t) -> Void) {
    var it: io_iterator_t = 0
    guard IOServiceGetMatchingServices(kIOMainPortDefault, IOServiceMatching(cls), &it) == KERN_SUCCESS else { return }
    while case let s = IOIteratorNext(it), s != 0 { body(s); IOObjectRelease(s) }
    IOObjectRelease(it)
}
func le32(_ d: Data) -> UInt32 { d.withUnsafeBytes { $0.loadUnaligned(as: UInt32.self) } } // VDOs stored little-endian

print("uid=\(getuid())")
each("AppleTCControllerType10") { s in
    let p = props(s)
    print("Port \(p["PortNumber"] ?? "?") connected=\(p["ConnectionActive"] ?? "?") active=\(p["TransportsActive"] ?? "?") connect=\(p["IOAccessoryUSBConnectString"] ?? "?")")
}
for cls in ["IOPortTransportStateUSB2", "IOPortTransportStateUSB3", "IOPortTransportStateCIO", "IOPortTransportStateDisplayPort"] {
    each(cls) { s in
        let p = props(s)
        print("\(cls) port=\(p["ParentPortNumber"] ?? "?") active=\(p["Active"] ?? "?") rate=\(p["DataRateDescription"] ?? "-") role=\(p["DataRoleDescription"] ?? "-")")
    }
}
each("IOPortTransportComponentCCUSBPDSOPp") { s in
    let p = props(s)
    let md = p["Metadata"] as? [String: Any] ?? [:]
    let vdos = (md["VDOs"] as? [Data] ?? []).map(le32)
    print("SOP' port=\(p["ParentPortNumber"] ?? "?") type=\(p["Product Type Description"] ?? "?") VDOs=\(vdos.map { String(format: "0x%08X", $0) })")
    if vdos.count >= 4 {
        let c = vdos[3]
        print("  Cable VDO: speed=\(c & 0x7) vbusCurrent=\((c >> 5) & 0x3) maxVbus=\((c >> 9) & 0x3)")
    }
}

let port = IONotificationPortCreate(kIOMainPortDefault)!
IONotificationPortSetDispatchQueue(port, DispatchQueue.main)
var iters: [io_iterator_t] = []
for cls in ["IOPortTransportComponentCCUSBPDSOPp", "IOPortTransportStateUSB2", "IOPortTransportStateUSB3", "IOPortTransportStateCIO"] {
    for (type, label) in [(kIOFirstMatchNotification, "arrived"), (kIOTerminatedNotification, "gone")] {
        var it: io_iterator_t = 0
        let ctx = Unmanaged.passRetained("\(cls) \(label)" as NSString).toOpaque()
        let kr = IOServiceAddMatchingNotification(port, type, IOServiceMatching(cls), { ctx, it in
            let tag = Unmanaged<NSString>.fromOpaque(ctx!).takeUnretainedValue()
            while case let s = IOIteratorNext(it), s != 0 { print("NOTIFY \(tag)"); IOObjectRelease(s) }
        }, ctx, &it)
        print("register \(cls) \(label): kr=\(kr)")
        while case let s = IOIteratorNext(it), s != 0 { IOObjectRelease(s) } // drain to arm
        iters.append(it)
    }
}
let secs = Double(CommandLine.arguments.dropFirst().first ?? "2") ?? 2
DispatchQueue.main.asyncAfter(deadline: .now() + secs) { exit(0) }
dispatchMain()
```

Output:

```
uid=501
Port 3 connected=1 active=(CC) connect=Device
Port 2 connected=0 active=() connect=None
Port 1 connected=1 active=(CC, USB2) connect=Host
IOPortTransportStateUSB2 port=3 active=0 rate=None role=Host
IOPortTransportStateUSB2 port=1 active=1 rate=480 Mbps (High Speed) role=Device
IOPortTransportStateDisplayPort port=1 active=0 rate=- role=-      <- this is Port-HDMI@1 (ParentPortType 6), not USB-C port 1
SOP' port=3 type=Passive Cable VDOs=["0x18000000", "0x00000000", "0x00000000", "0x00086840"]
  Cable VDO: speed=0 vbusCurrent=2 maxVbus=0      (USB 2.0, 5 A, 20 V)
register … : kr=0   (all 8 registrations succeeded)
```

The probe did not see a plug or unplug event fire, because nobody touched the Cable during the run. It shows that registration succeeds without privilege, and WhatCable's shipping watchers show that the callbacks fire.

## Not settled here (candidate questions)

1. **Will a fast Cable in Loopback bring up USB3 or CIO between two ports of the same Mac?** A Mac acting as a USB device seems to be USB 2 only (this Loopback came up as USB2 Device on one end). WhatCable's `HostToHostLink` recognises CIO between *two different* Macs, but nothing here proves a single Mac will train a CIO link to itself across two of its own USB4 controllers. This is the biggest threat to Loopback for Speed class, and it needs a hands-on test with a USB4/TB4 Cable.
2. Does the default "Ask for new accessories" setting prompt, or silently withhold data, for a Loopback self-connection? This Mac is set to "Always Allow", so it couldn't be observed.
3. What exact `DataRateDescription` strings and `Generation` values does `IOPortTransportStateUSB3` publish for a Gen 2x2 link, if any?

## Sources

- Live `ioreg -l -w0` / `ioreg -c AppleTCControllerType10` / `ioreg -c IOThunderboltPort` and `system_profiler SPUSBHostDataType SPThunderboltDataType` on this Mac, 2026-10-11.
- macOS 26.6.2 dyld shared cache: the strings `IOPortTransportStateUSB2`, `…USB3`, `…CIO`, `…DisplayPort` are present.
- macOS 26 SDK headers: `IOKit.framework/Headers/IOKitLib.h` (matching and interest notifications, `IORegistryEntryCreateCFProperties`), `IOMessage.h` (`kIOMessageServicePropertyChange`), and `IOKit/usb/IOUSBHostFamilyDefinitions.h` (`UsbIOPort`, `UsbTransportState`).
- Apple, [The IOKitLib API (Accessing Hardware From Applications)](https://developer.apple.com/library/content/documentation/DeviceDrivers/Conceptual/AccessingHardware/AH_IOKitLib_API/AH_IOKitLib_API.html).
- Apple Support, [Allow accessories to connect to a Mac laptop](https://support.apple.com/en-us/102282).
- WhatCable source, [github.com/darrylmorley/whatcable](https://github.com/darrylmorley/whatcable) (README "How it works"; `Watchers/HPMPortControllerClasses.swift`, `AppleHPMInterfaceWatcher.swift`, `USB3TransportWatcher.swift`, `USBPDSOPWatcher.swift`, `TRMTransportWatcher.swift`; `Core/Thunderbolt/IOThunderboltLink.swift`, `CIOCableCapability.swift`, `HostToHostLink.swift`).
- USB Power Delivery Specification Rev 3.x, Passive Cable VDO field layout (bits 2..0 USB speed, 6..5 VBUS current, 10..9 max VBUS voltage).
