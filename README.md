# Turn an iPad or iPhone stuck on iOS 12 into a second display for your Mac (macOS 26), over Lightning or Wi-Fi — free, self-hosted

**Built and running for real** on an iPad Air (Model A1475, iOS 12.5.8), over a Lightning cable and over Wi-Fi. H.264 video, touch, two-finger scroll, and a real mouse cursor all work.

Free alternative to Sidecar (iOS 13+), Duet Display, and Luna Display for a device that cannot run [OpenDisplay](https://github.com/peetzweg/opendisplay)'s own client (that client needs iPadOS 17+). The Mac app is unmodified OpenDisplay (GPL-3.0): it creates the virtual display, captures it, encodes H.264, and sends it over USB (`usbmuxd`) or Wi-Fi (Bonjour). No `iproxy`. The `iOS/` app is a clean-room iOS 12 client of that same protocol (MIT).

Same Wi-Fi is enough. Plug in a Lightning cable when you want it: the Mac prefers USB (lower, steadier latency) and falls back to Wi-Fi when the cable is pulled.

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

## Step 2 — Build the iPad app (`iOS/` in this repo)

```bash
open iOS/LegacyPadDisplay.xcodeproj
```

In Xcode:

1. Select target **LegacyPadDisplay** → **Signing & Capabilities** → set Team to your personal Apple ID.
2. Copy the iOS 12.5 DeviceSupport profile (next section) if the iPad is not already a Run destination.
3. Plug the iPad in with a Lightning cable, select it in the toolbar, and hit Run.
4. On first install, on the iPad open **Settings → General → VPN & Device Management** and trust your developer certificate.

The app should show a fullscreen black screen with "Listening on :9000" at the bottom. It is waiting for the Mac.

### Copy the iOS 12.5 DeviceSupport profile

Recent Xcode builds do not ship iOS 12 device support. Without this folder the iPad does not appear as a Run destination, and Xcode reports "Failed to prepare the device for development". Copy the 12.5 profile from [apptim/iPhoneOSDeviceSupport](https://github.com/apptim/iPhoneOSDeviceSupport):

```bash
curl -L -o /tmp/12.5.zip https://raw.githubusercontent.com/apptim/iPhoneOSDeviceSupport/master/12.5.zip
unzip -o /tmp/12.5.zip -d /tmp/ds125
cp -R "/tmp/ds125/12.5" ~/Library/Developer/Xcode/iOS\ DeviceSupport/
```

A plain `12.5` folder is enough for 12.5.8 (16H88). [filsv/iOSDeviceSupport](https://github.com/filsv/iOSDeviceSupport) has other versions. Quit Xcode fully, reconnect the iPad, and reopen the project.

### Free Apple ID — the app expires after 7 days

Apple expires a free-account install after 7 days. Plug the cable back in, open Xcode, and hit Run again. A paid Apple Developer Program membership ($99/year) signs the app for a year.

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
