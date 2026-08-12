# Panasonic KX-MB1500 on macOS 15+

Printing **and scanning** for the Panasonic KX-MB1500 multifunction printer on
modern macOS, including Apple Silicon. The device is discontinued: the last
official Panasonic driver targets OS X 10.10 and does not work on macOS 11 or
later.

This project gives you a **real system printer**. After installation
“Panasonic KX-MB1500” shows up in the normal print dialog of any
application — Word, Preview, Safari, anything.

Scanning works too, through a USB tunnel — but only inside the bundled
“Panasonic MFP” app, not in Image Capture. See [Scanning](#scanning).

> **Scope.** Verified on macOS 26.5 (Apple Silicon) against real hardware:
> printing from Word, and multi-page scanning collected into a single PDF.

[Русская версия](README.ru.md)

---

## Why the vendor drivers fail

Verified on macOS 26.5; the app ships the same diagnostics under **Device**:

| Component | State |
|---|---|
| Panasonic print filter (macOS) | Built for `i386`/`ppc` only. 32-bit code stopped running in macOS 10.15, and Rosetta translates `x86_64` only |
| Panasonic ICA scanner module | `x86_64`, starts under Rosetta, then fails to load: it references `_kICANotification*` from Carbon, an API Apple removed |
| The device itself | Its IEEE-1284 `device-id` has no `CMD` field — no PostScript, no PCL. It only speaks Panasonic’s host-based format, so generic drivers cannot help either |

Panasonic did, however, ship a driver for **Linux x86_64**, and that one is
perfectly functional. Everything here is built on top of it.

---

## How it works

```
Word / Preview
      ↓
macOS print queue   (PPD from this repository)
      ↓  PDF, unmodified
CUPS backend  /usr/libexec/cups/backend/panasonic
      ↓  HTTP to 127.0.0.1:9631
Linux container  →  Panasonic’s own filter (mccgdi)  →  PJL/GDI stream
      ↓
Standard CUPS USB backend  →  the printer
```

The container has no access to USB and no access to your files. It only turns a
PDF into the byte stream the printer understands. Delivery to the device is done
by Apple’s own `usb` backend.

Scanning runs the same trick in reverse — the Linux driver stays in a container,
and USB is brought to it over a socket:

```
Panasonic MFP app
      ↓
scanimage (container)  →  Panasonic’s own libsane backend
      ↓  libusb-0.1 calls, intercepted by a shim
TCP 9632  →  usbbridge (macOS)  →  IOKit  →  the scanner
```

Two details make this work, and both cost a while to find:

* The Panasonic backend carries **its own copy of `sanei_usb`** (from
  sane-backends 1.0.19) that goes through libusb-0.1. Loaded as a module by
  SANE’s `dll` backend, its `sanei_usb_*` calls get resolved against the system
  `libsane.so.1` instead, which uses libusb-1.0 and `/dev/bus/usb` — neither of
  which exists in the container. So the backend is substituted **for
  `libsane.so.1` itself**; that is legitimate, since its `SONAME` is exactly
  `libsane.so.1`.
* Its libusb headers define `LIBUSB_PATH_MAX` as **4097**, not 512 as in
  libusb-0.1.12. Get that wrong and the backend reads the device list past the
  end of the shim’s structures and silently finds nothing.

The bridge holds the device only for the duration of one operation and closes it
right after, so it does not block printing.

### Why a backend and not a filter

Two earlier designs failed, which is worth knowing before “simplifying” this:

1. **A filter talking over the network.** macOS runs print filters in a sandbox
   that permits exactly one outbound connection — the `cupsd` socket
   (`/System/Library/Sandbox/Profiles/com.apple.printtool.daemon.sb`).
2. **A filter exchanging files.** Writes are refused with `EPERM` in
   `/private/tmp`, `/private/var/tmp` and `/Library/Printers` alike: CUPS builds
   filters a private profile that is stricter than the system one.
3. **`Sandboxing Off`** in `cups-files.conf` is ignored by macOS.

Backends are not subject to those limits by design — `ipp` must reach the
network, `usb` must open a device. So the conversion lives in a backend.

---

## Installation

Short version below; for a step-by-step walkthrough with troubleshooting, see
**[docs/INSTALL.md](docs/INSTALL.md)**.

### Requirements

* macOS 15 or newer, Apple Silicon or Intel;
* [Homebrew](https://brew.sh);
* colima and docker:

```bash
brew install colima docker
```

### From the prebuilt package

Download `Panasonic MFP <version>.pkg` from Releases and run it. The package is
not signed, so the first time right-click it and choose **Open**, or allow it
under *System Settings → Privacy & Security*.

The installer creates the print queue and enables a small background helper. On
the app’s first launch the Panasonic driver is downloaded from the vendor’s
official support site and the container is built — this needs internet access
and takes a few minutes.

### From source

```bash
git clone https://github.com/<your-account>/PanasonicMFP.git
cd PanasonicMFP
./packaging/build-installer.sh
open build/*.pkg
```

---

## Usage

Just print. “Panasonic KX-MB1500” is available in every application’s print
dialog, and the app does **not** need to be running: a background helper keeps
the conversion service alive.

The “Panasonic MFP” app is there for the rest:

* **scanning** — see [Scanning](#scanning); this is the only place it works;
* hardware diagnostics — what USB, CUPS and ImageCapture actually see, and the
  state of the stock Panasonic drivers;
* controlling the conversion service, with its status in the menu bar;

The interface follows your system language: English and Russian are included.

### Managing the service by hand

```bash
/Library/Printers/PanasonicMFP/panasonic-mfp-service status
/Library/Printers/PanasonicMFP/panasonic-mfp-service start
/Library/Printers/PanasonicMFP/panasonic-mfp-service logs
```

---

## Scanning

Scanning happens **inside the “Panasonic MFP” app**, on the Scanning tab. The
scanner will *not* appear in Image Capture, Preview or any other macOS program:
Panasonic’s ICA module cannot load — it depends on a Carbon API Apple removed —
and this project deliberately bypasses ImageCapture entirely rather than trying
to repair it.

What the app does:

* flatbed scanning at 75–1200 dpi, line art / grayscale / color;
* pages accumulate into one document — scan, put the next sheet, scan again;
* export to PDF, searchable PDF (OCR via Vision, Russian and English),
  multi-page TIFF, or JPEG/PNG per page;
* a destination folder you pick once; saving afterwards takes no dialog.

The KX-MB1500 has no document feeder, so “batch scanning” means pressing
**Scan page** once per sheet. They are merged when you save.

On the first scan the app downloads Panasonic’s Linux scanner driver
(`panamfs-scan-1.3.1-x86_64`, ~1.2 MB), verifies its SHA-256 and builds the
container — a few minutes, once.

To check the whole path against real hardware without the interface:

```bash
/Applications/Panasonic\ MFP.app/Contents/MacOS/PanasonicMFP --scan-test 2
```

It scans two pages, merges them into a PDF and prints where it landed.

## What gets installed where

| Path | Purpose |
|---|---|
| `/Applications/Panasonic MFP.app` | The application |
| `/usr/libexec/cups/backend/panasonic` | CUPS backend, runs as root |
| `/Library/Printers/PanasonicMFP/` | Helper scripts, PPD, container definitions, `usbbridge` |
| `~/Library/Application Support/PanasonicMFP/` | Downloaded drivers, service token, scanner exchange folder |
| `~/Library/LaunchAgents/com.vt.panasonic-mfp.supervisor.plist` | Keeps the service alive |

### Uninstall

```bash
lpadmin -x Panasonic_KX_MB1500
launchctl bootout gui/$UID/com.vt.panasonic-mfp.supervisor
rm -f ~/Library/LaunchAgents/com.vt.panasonic-mfp.supervisor.plist
/Library/Printers/PanasonicMFP/panasonic-mfp-service stop
sudo rm -rf /Library/Printers/PanasonicMFP /usr/libexec/cups/backend/panasonic
sudo rm -rf "/Applications/Panasonic MFP.app"
rm -rf ~/Library/Application\ Support/PanasonicMFP
docker rmi panasonic-mfp-converter panasonic-mfp-scanner
```

---

## Known limitations

All of these were established by measurement on real hardware, not guessed at.

**Several files sent as one job print only the first document.** Select three
files in Finder, hit Print, and one sheet comes out. CUPS starts a backend
*once per job*, not once per document, and expects it to stay alive and accept
the rest; ours exits after the first, so CUPS waits on a dead process. Print
files one at a time, or add them in the application — it submits each file as
its own job and waits for the previous one to leave the queue.

**"Done" means the data reached the printer, not that the sheet came out.** The
printer interface is unidirectional: the device never reports back. The
application marks a file finished when its job leaves the CUPS queue, which
happens once the bytes are delivered. Expect the tray to lag behind the
interface by a few seconds.

**Print density has exactly two controls.** `TonerSave` and `Resolution` change
the byte stream; `MediaType` does not — `Plain`, `ThickPaper` and `ThinPaper`
produce byte-identical output. The PPD declares no `print-quality`, so CUPS
silently drops that option and logs `Bad resolution`. If output is pale with
both controls at maximum, the cause is the toner-save setting in the device's
own menu, the cartridge, or the drum — there is nothing left to turn in
software.

**The device drops off the USB bus when it sleeps.** Presence is checked
through IOKit; the first job after a long idle may need the device woken up.

**Scanning works only inside the application.** The vendor's ICA module does
not load on current macOS, so the scanner never appears in Image Capture,
Preview, or anything else that uses ImageCapture.

---

## Security notes

It is worth knowing what runs with elevated privileges:

* **the backend runs as root** for every job — that is how all CUPS backends
  work. It only talks to `127.0.0.1`, handles temporary files and hands the
  result to Apple’s `usb` backend;
* **the service listens on `127.0.0.1` only** and requires a token stored in the
  user’s home directory with mode `0600`;
* **Panasonic’s closed-source drivers run inside containers**, with no access to
  your files and no direct access to USB;
* **the USB bridge runs as your user, not root**, listens on `127.0.0.1:9632`
  only, and is started for a single scan and stopped right after. It exposes one
  device — the Panasonic scanner interface — and nothing else on the bus.

---

## Licensing and legal

The code in this repository is your own work; pick a license before publishing
(MIT is a reasonable default).

**The Panasonic driver is neither included nor redistributed here.** It is the
vendor’s proprietary software: the package downloads it from Panasonic’s
official support site on first launch and verifies its SHA-256 checksum. The
printer description (`macos/PanasonicKXMB1500.ppd`) was written from scratch
against the PPD specification and contains no vendor files.

This project is not affiliated with or endorsed by Panasonic Corporation.

