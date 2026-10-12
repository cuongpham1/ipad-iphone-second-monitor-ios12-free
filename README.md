# Turn an iPad or iPhone stuck on iOS 12 into a second display for your Mac (macOS 26), over Lightning or Wi-Fi — free, self-hosted

**Built and running for real** on an iPad Air (Model A1475, iOS 12.5.8), over a Lightning cable and over Wi-Fi. H.264 video, touch, two-finger scroll, and a real mouse cursor all work.

Free alternative to Sidecar (iOS 13+), Duet Display, and Luna Display for a device that cannot run [OpenDisplay](https://github.com/peetzweg/opendisplay)'s own client (that client needs iPadOS 17+). The Mac app is unmodified OpenDisplay (GPL-3.0): it creates the virtual display, captures it, encodes H.264, and sends it over USB (`usbmuxd`) or Wi-Fi (Bonjour). No `iproxy`. The `iOS/` app is a clean-room iOS 12 client of that same protocol (MIT).

Same Wi-Fi is enough. Plug in a Lightning cable for better speed and quality: the Mac prefers USB (lower, steadier latency) and falls back to Wi-Fi when the cable is pulled.

iPhone uses the same project. Pick it as the Xcode destination. The screen is smaller, so an iPad is the useful case. Minimum iOS is **12.0** (`iOS/project.yml` → `deploymentTarget`), tested on iOS 12.5.8.

`iOS/LegacyPadDisplay.xcodeproj` is ready to open. `iOS/project.yml` is the XcodeGen source of truth (`xcodegen generate` after you edit it). The client lives in `iOS/App/` (`VideoReceiver.swift` listens on port 9000, decodes H.264, and sends touch, scroll, and cursor messages). `Mac/` is a git submodule of upstream OpenDisplay, unmodified, so `git submodule update --remote` picks up their releases.

## Step 0 — Clone with submodules

```bash
git clone --recurse-submodules https://github.com/cuongpham1/ipad-iphone-second-monitor-ios12-free.git
cd ipad-iphone-second-monitor-ios12-free
```

If you already cloned without `--recurse-submodules`:

```bash
git submodule update --init --recursive
```

## Step 1 — Get the Mac app (OpenDisplay, unmodified)

- **Prebuilt (recommended):** download `OpenDisplay.dmg` from the [latest release](https://github.com/peetzweg/opendisplay/releases/latest) and drag `OpenDisplay.app` into `/Applications` before launching. macOS ties Screen Recording and Accessibility to that path. Launching from the disk image, then moving the app, means granting both permissions again, and the volume unmounts on reboot.
- **From source:** open `Mac/` in Xcode and run it. See [OpenDisplay's build instructions](https://github.com/peetzweg/opendisplay#readme).

On first launch, grant **Screen Recording** and **Accessibility** (System Settings → Privacy & Security). Fully quit the app (Cmd+Q) and relaunch it. Flipping the toggle is not enough; the app reads the new permissions on restart.

## Step 2 — Install the iPad app

**Xcode 26 and later cannot build for iOS 12.** Their SDK only targets iOS 15+ ("the range of supported deployment target versions is 15.0 to 27.0.x"), and Xcode 16 will not launch on macOS 27. Installing the prebuilt `.ipa` works on any macOS.

### Option A — Prebuilt `.ipa` + Sideloadly (recommended)

1. Download `LegacyPadDisplay.ipa` from the [latest release](https://github.com/cuongpham1/ipad-iphone-second-monitor-ios12-free/releases/latest). GitHub Actions builds it unsigned with Xcode 16 ([workflow](.github/workflows/build-ipa.yml)).
2. Install [Sideloadly](https://sideloadly.io) on the Mac.
3. Plug the iPad in with a Lightning cable, drop the `.ipa` into Sideloadly, enter your Apple ID, and click Start. Sideloadly signs it with your free Apple ID and installs it.
4. On the iPad, open **Settings → General → Profiles & Device Management** and trust your Apple ID.

Tested on an iPad Air (iOS 12.5.8) with Xcode 27 / macOS 27 installed on the Mac.

If Sideloadly fails with `LOCKDOWN_E_INVALID_HOST_ID`, the iPad no longer trusts this Mac (common after a macOS upgrade). Unplug, on the iPad run **Settings → General → Reset → Reset Location & Privacy**, plug back in, tap **Trust**, and click Start again.

AltServer also works on iOS 12.2+. With `ideviceinstaller` (`brew install ideviceinstaller`) you can install an `.ipa` that is already signed.

The app should show a fullscreen black screen with "Listening on :9000" at the bottom. It is waiting for the Mac.

### Option B — Build from source (Xcode 16.x only)

You need Xcode 16.4 or older: on macOS 15 Sequoia or earlier, or in a Sequoia VM (UTM). A VM has no USB passthrough, so build an ad-hoc `.ipa` there and install it from the host with Sideloadly or `ideviceinstaller` ([issue #1](https://github.com/cuongpham1/ipad-iphone-second-monitor-ios12-free/issues/1), [issue #2](https://github.com/cuongpham1/ipad-iphone-second-monitor-ios12-free/issues/2)). In a VM, copy the repo onto the VM's own disk: the `.xcodeproj` package looks empty through a virtiofs shared folder. The committed project needs Xcode 16 (`objectVersion` 77); Xcode 15 cannot open it.

```bash
open iOS/LegacyPadDisplay.xcodeproj
```

1. Select target **LegacyPadDisplay** → **Signing & Capabilities** → set Team to your personal Apple ID.
2. If the iPad is not a Run destination ("Failed to prepare the device for development"), copy the iOS 12.5 DeviceSupport profile from [apptim/iPhoneOSDeviceSupport](https://github.com/apptim/iPhoneOSDeviceSupport), then quit Xcode fully and reconnect the iPad:

   ```bash
   curl -L -o /tmp/12.5.zip https://raw.githubusercontent.com/apptim/iPhoneOSDeviceSupport/master/12.5.zip
   unzip -o /tmp/12.5.zip -d /tmp/ds125
   cp -R "/tmp/ds125/12.5" ~/Library/Developer/Xcode/iOS\ DeviceSupport/
   ```

   A plain `12.5` folder covers all 12.x. [filsv/iOSDeviceSupport](https://github.com/filsv/iOSDeviceSupport) has other versions.
3. Select the iPad in the toolbar and hit Run, then trust your certificate on the iPad as in Option A.

### Free Apple ID — the app expires after 7 days

This is an Apple limitation, not something this project can fix. After 7
days, plug the cable back in and reinstall: run Sideloadly again (Option A)
or hit Run in Xcode (Option B). A paid Apple Developer Program membership
($99/year) signs for a full year.

## Step 3 — Connect

Leave both apps running (Mac permissions granted and the app relaunched; iPad app open).

- **Cable:** plug the iPad into the Mac. OpenDisplay finds it through `usbmuxd` on port 9000, same as its own client.
- **Wi-Fi:** same network, no cable. The Mac browses for `_opensidecar._tcp` and connects to port 9000. Allow **Local Network** on the iPad the first time.

The iPad should switch from black to the macOS desktop, with a mouse cursor. A new display appears under System Settings → Displays. Drag a window onto it.

## If it won't connect

- **No new display under System Settings → Displays:** Screen Recording or Accessibility is missing, or the Mac app was not fully quit and relaunched after you granted them (Step 1).
- The Mac app's console (Xcode → View → Debug Area → Console, scheme OpenSidecarMac) shows whether it sees the device over `usbmuxd`.
- A black frame for a second after you switch back to the iPad app is the reconnect. The listener stops in the background, and `ensureListening()` starts it again when the app returns.
- An `updateRequired` message means a newer OpenDisplay changed the protocol. `VideoReceiver.swift` omits the `pv` field, so the Mac treats this client as protocol 1, which current OpenDisplay releases still accept.

## What works / what's missing

Works: video, touch to click and drag, two-finger scroll, a mouse cursor (position and shape), and reconnect when the app returns to the foreground (a short flicker).

Missing: an external keyboard, automatic rotation to match the virtual display, latency measurement.

## License

- `iOS/` (this client): MIT, see [LICENSE](LICENSE).
- `Mac/`: submodule of `peetzweg/opendisplay`, GPL-3.0, copyright held by its authors. Unmodified, not vendored into this repo.

*Keywords: iPad iOS 12 second monitor Mac, old iPad external display,
Sidecar alternative iOS 12, Duet Display free alternative, Luna Display
free alternative, OpenDisplay iOS 12 client, Lightning USB second screen,
legacy iPad second monitor, LegacyPadDisplay.*
