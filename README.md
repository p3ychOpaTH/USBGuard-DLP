# USBGuard-DLP

**Real-time endpoint Data Loss Prevention for USB mass storage devices.**

USBGuard-DLP silently enforces USB device policies, instantly blocks unauthorized drives, and forensically shadow-copies all file activity on allowed devices — all from a single, tamper-resistant Windows service.

---

[![License](https://img.shields.io/badge/license-GPLv3-blue.svg)](LICENSE)
[![Python](https://img.shields.io/badge/python-3.8%2B-blue)](https://www.python.org/)
[![Platform](https://img.shields.io/badge/platform-Windows%2010%2F11%20%7C%20Server-lightgrey)]()
[![Status](https://img.shields.io/badge/status-stable-brightgreen)]()

<p align="center">
  <img src="https://via.placeholder.com/600x200/1a1a2e/ffffff?text=USBGuard-DLP+Logo" alt="USBGuard-DLP logo" />
</p>

---

## Features

- **Real-time detection** — WMI event monitoring plus a polling fallback catches devices the moment they connect.
- **Instant blocking** — Unauthorized devices are disabled via SetupAPI before their filesystems ever mount.
- **Device allowlisting** — Permit drives by serial number, vendor ID, product ID, or device class. Mix and match as needed.
- **Forensic shadow copy** — Every file created or modified on an allowed drive is silently mirrored to a secure vault, preserving original directory structure and timestamps.
- **Tamper-resistant** — Runs as a protected Windows service with hardened file ACLs. Only SYSTEM and built-in Administrators can touch the binaries or config.
- **Encrypted configuration** — Sensitive sections of the config file support AES-256 encryption, with keys stored in the registry or TPM.
- **Zero network exposure** — No cloud dependency, no phoning home, no outbound connections at all. Policy is local, signed, and works offline.
- **Dual logging** — Writes simultaneously to the Windows Event Log and a rotating on-disk debug file.

---

## Requirements

| Component | Minimum |
|-----------|---------|
| OS | Windows 10 / Windows 11 / Windows Server 2016 or later |
| Python | 3.8 or higher |
| Privileges | Local Administrator |
| Python packages | `pywin32`, `wmi`, `cryptography` |

Install the required packages with:

```bash
pip install -r requirements.txt

A requirements.txt is included in the repository.

---

Quick start

```bash
# Install the Windows service
python endpoint_usb_dlp.py install

# Start it immediately
python endpoint_usb_dlp.py start

# Optional: run in console debug mode (Ctrl+C to exit)
python endpoint_usb_dlp.py debug
```

After the first run, the default configuration file and shadow vault are created automatically at:

```
C:\ProgramData\USBGuard\
```

---

Configuration

Open C:\ProgramData\USBGuard\config.ini after the first run. Here's a realistic example:

```ini
[General]
polling_interval_sec = 2.0

[Policy]
# Set to True if you want to allow any USB drive by default (not recommended)
allow_unknown_devices = False

[Shadow]
shadow_enabled = True
shadow_copy_folder = C:\ProgramData\USBGuard\ShadowVault
# Files larger than this (in MB) will be skipped during shadow copy
shadow_file_max_size_mb = 50

[Logging]
event_log_id_base = 1000

[Allowlist_Devices]
# Format: serial_number = friendly_name (optional, helps with auditing)
1234567890ABCDEF = CorpApprovedSandisk
ABCDEF1234567890 = JaneEngineeringDrive

[Allowlist_Vendors]
# Comma-separated vendor IDs (hex, case-insensitive)
vendor_ids = 0781,0951

[Allowlist_Products]
# Comma-separated product IDs
product_ids = 557D,1666

[Allowlist_Classes]
# Device classes to permit (DiskDrive is usually needed)
classes = DiskDrive
```

A fully commented example file is kept at config.ini.example in the repository. Copy it to config.ini and edit to fit your environment.

---

How it works — the short version

1. A USB mass storage device is inserted.
2. The monitor thread detects it via WMI event or poll cycle.
3. Its serial, VID, PID, and class are compared against the allowlists.
4. Not allowed → the device is disabled on the spot and any associated drive letters are unmounted.
5. Allowed → the shadow copy engine starts silently mirroring every new or modified file to the vault.
6. When the device is removed, the shadow thread stops cleanly. The forensic snapshot stays behind.

Everything is logged locally. Nothing leaves the endpoint.

---

Security considerations

· This tool protects against physical exfiltration via USB only. It does not replace network DLP, email filters, or full-disk encryption. Layer it with those for defense in depth.
· The shadow vault is not encrypted by default. Secure it with EFS, BitLocker, or strict NTFS permissions. Treat it as sensitive forensic evidence.
· The config.ini file contains hardware identifiers. The included .gitignore prevents it from being committed to version control. Keep it safe.
· Test in a staging environment before deploying to production. Disabling the wrong device class can block legitimate hardware.
· A local administrator who is determined and skilled can disable the service. This tool raises the bar; it does not eliminate the insider threat entirely.

---

Uninstallation

```bash
# Stop the service
python endpoint_usb_dlp.py stop

# Remove it from the Windows service list
python endpoint_usb_dlp.py remove
```

To remove leftover data:

```powershell
Remove-Item -Recurse -Force "C:\ProgramData\USBGuard"
Remove-ItemProperty -Path "HKLM:\SOFTWARE\USBGuard" -Name "CfgKey"
```

---

FAQ

Does this work on Windows Server Core?
Yes. There's no GUI dependency. The service and debug mode both run on Core editions.

What happens if someone pulls the drive out mid-copy?
The shadow copy thread detects the drive is gone on the next poll cycle and terminates gracefully. Already-copied files remain in the vault.

Can I block specific file types on allowed drives?
Not in the current version. This release enforces device-level policy. Content-level filtering (e.g., "block .docx on USB") is on the roadmap.

Does the agent need internet access?
No. It is fully offline. No license check, no telemetry, no cloud component.

What if a device has no serial number?
The script falls back to the PNP Device ID as a unique identifier. This is less elegant but functional.

Can I push the config via GPO or SCCM?
Yes. The config file is a plain INI file at a known path. Deploy it with your existing configuration management tools.

---

License

This project is licensed under the GNU General Public License v3.0.
