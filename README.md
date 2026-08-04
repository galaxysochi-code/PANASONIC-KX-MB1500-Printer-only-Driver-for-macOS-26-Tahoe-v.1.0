Panasonic KX-MB1500 on macOS
Installation Guide and Project Description

A working print driver for a discontinued multifunction printer on macOS 15 and newer, including Apple Silicon. After installation the device appears as an ordinary system printer in every application. Printing only — scanning is out of scope.
 
1. Overview
The Panasonic KX-MB1500 is discontinued. The last official driver from Panasonic targets OS X 10.10, and on macOS 11 and later it cannot run at all. This project restores printing by a different route: it runs Panasonic’s own Linux driver inside a small container and delivers the resulting data to the device through Apple’s standard USB backend.
The result is a real system print queue. You print from Word, Preview, Safari or anything else exactly as you would with a supported printer — no separate application needs to be running.
Scope: printing only, verified on macOS 26.5 with Apple Silicon. Scanning is not part of this project — section 8 explains why.
Why the vendor drivers fail
These findings come from the diagnostics built into the application, under the Device tab:
Component	State on macOS 15+
Panasonic print filter	Built for i386 and ppc only. 32-bit code stopped running in macOS 10.15, and Rosetta translates x86_64 only.
Panasonic ICA scanner module	x86_64, starts under Rosetta, then fails to load: it references Carbon symbols Apple removed.
The printer itself	Its IEEE-1284 identity string carries no CMD field — no PostScript, no PCL. Generic drivers cannot help.

Panasonic did ship a driver for Linux x86_64, and that one is fully functional. Everything here is built on top of it.
2. How it works
A print job travels through five stages:
Word / Preview
      |
macOS print queue        (PPD from this project)
      |  PDF, unmodified
CUPS backend             /usr/libexec/cups/backend/panasonic
      |  HTTP to 127.0.0.1:9631
Linux container          Panasonic mccgdi filter -> PJL/GDI stream
      |
Standard CUPS USB backend -> the printer
The container never touches USB and never sees your files. Its only job is turning a PDF into the byte stream this printer understands. Delivery to the hardware is done by Apple’s own usb backend, which already knows how to wait for a sleeping device.
Why a backend rather than a print filter
Two earlier designs failed, and the reasons are worth recording before anyone tries to “simplify” this:
•	A filter reaching the service over the network. macOS runs print filters in a sandbox that permits exactly one outbound connection — the cupsd socket.
•	A filter exchanging files through a spool directory. Writes are refused with EPERM in /private/tmp, /private/var/tmp and /Library/Printers alike: CUPS gives filters a private profile stricter than the documented system one.
•	Disabling the sandbox with “Sandboxing Off” in cups-files.conf. macOS ignores the directive entirely.
Backends are not subject to those limits by design — the ipp backend must reach the network and the usb backend must open a device. That is why the conversion lives in a backend.
 
3. Requirements
•	macOS 15 or newer, Apple Silicon or Intel.
•	Homebrew (https://brew.sh).
•	colima and docker, which provide the Linux container.
•	An internet connection for the first launch only.
•	The printer connected by USB and switched on.
Install the container tooling first:
brew install colima docker
4. Installation
From the prebuilt package
1.	Download the .pkg file from the Releases page of the repository.
2.	Right-click the file and choose Open — the package is not signed by an Apple Developer ID, so a plain double-click is refused. Alternatively allow it under System Settings → Privacy & Security.
3.	Follow the installer. It will ask for your administrator password: the CUPS backend has to be installed as root, which is how every print backend works.
4.	Launch the Panasonic MFP application from the Applications folder.
5.	Wait for the first-run setup to finish. The application downloads the Panasonic Linux driver from the vendor’s support site, verifies its SHA-256 checksum and builds the container. This takes a few minutes and happens only once.
6.	Print a test page from any application to confirm.
From source
git clone https://github.com/<account>/PanasonicMFP.git
cd PanasonicMFP
./packaging/build-installer.sh
open "build/Panasonic MFP 1.0.pkg"
What the installer does
•	Copies the application to the Applications folder.
•	Installs the CUPS backend and the helper scripts.
•	Creates the print queue named Panasonic KX-MB1500.
•	Enables a small background agent that keeps the conversion service alive, so printing works even when the application is closed.
 
5. Using the printer
Print normally. The queue appears in every print dialog, and the standard options — paper size, resolution, paper type, toner saving — are described by the printer definition shipped with this project.
The first job after a period of inactivity may take longer: the virtual machine has to start. Subsequent jobs take a few seconds.
6. The application
Printing does not require the application. It exists for three other purposes:
•	Hardware diagnostics — what USB, CUPS and ImageCapture actually see, and the exact state of the stock Panasonic drivers. The report can be copied to the clipboard.
•	Service control — the status is shown in the menu bar, with start, stop and restart.
The interface follows the system language; English and Russian are included.
7. Managing the service by hand
/Library/Printers/PanasonicMFP/panasonic-mfp-service status
/Library/Printers/PanasonicMFP/panasonic-mfp-service start
/Library/Printers/PanasonicMFP/panasonic-mfp-service restart
/Library/Printers/PanasonicMFP/panasonic-mfp-service logs
8. Scanning
Scanning is deliberately not part of this project.
macOS detects the scanner, but opening a session returns “Failed to open a connection to the device”. The cause is the same Carbon dependency described in section 1: Panasonic’s ICA module cannot be loaded at all. The Device tab still reports whether macOS sees the scanner, which helps when diagnosing the hardware.
The approach that rescued printing does not transfer directly. A Linux scanner driver exists, but scanning needs real USB access, and a container on macOS has none. A plausible route is a shim that tunnels the container’s libusb calls to a macOS helper driving the device through IOKit. That is a separate project and it is not solved here.
 
9. Troubleshooting
Symptom	What to do
Jobs sit in the queue and nothing prints	Check the menu bar icon. If the service is stopped, start it there or run panasonic-mfp-service start.
“Service is not running” in the job status	colima is probably not up. Run colima start, then panasonic-mfp-service start.
The printer is missing from the Device tab	The device drops off the USB bus in deep power saving. Wake it with any button and press Refresh.
First launch fails to download the driver	Check the internet connection and run panasonic-mfp-service fetch-driver to retry.
The package will not open	It is unsigned. Right-click it and choose Open, or allow it in System Settings → Privacy & Security.
10. What gets installed where
Path	Purpose
/Applications/Panasonic MFP.app	The application
/usr/libexec/cups/backend/panasonic	CUPS backend, runs as root
/Library/Printers/PanasonicMFP/	Helper scripts, printer definition, container definition
~/Library/Application Support/PanasonicMFP/	Downloaded driver and service token
~/Library/LaunchAgents/	Background agent that keeps the service alive
Uninstalling
lpadmin -x Panasonic_KX_MB1500
launchctl bootout gui/$UID/com.vt.panasonic-mfp.supervisor
rm -f ~/Library/LaunchAgents/com.vt.panasonic-mfp.supervisor.plist
/Library/Printers/PanasonicMFP/panasonic-mfp-service stop
sudo rm -rf /Library/Printers/PanasonicMFP
sudo rm -f /usr/libexec/cups/backend/panasonic
sudo rm -rf "/Applications/Panasonic MFP.app"
rm -rf ~/Library/Application\ Support/PanasonicMFP
docker rmi panasonic-mfp-converter
11. Security notes
It is worth knowing exactly what runs with elevated privileges:
•	The backend runs as root for every print job. That is how all CUPS backends work. It talks only to 127.0.0.1, handles temporary files and hands the result to Apple’s usb backend.
•	The conversion service listens on 127.0.0.1 only and requires a token stored in the user’s home directory with mode 0600.
•	Panasonic’s closed-source driver runs inside the container, with no access to your files and no access to USB.
12. Licensing
The code of this project is released under the MIT license.
The Panasonic driver is neither included nor redistributed. It is the vendor’s proprietary software: the installer downloads it from Panasonic’s official support site on first launch and verifies its SHA-256 checksum. The printer definition file was written from scratch against the PPD specification and contains no vendor files.
This project is not affiliated with or endorsed by Panasonic Corporation.
<img width="451" height="243" alt="image" src="https://github.com/user-attachments/assets/f8171f73-72be-41c9-a748-ec67f97b0aa0" />
