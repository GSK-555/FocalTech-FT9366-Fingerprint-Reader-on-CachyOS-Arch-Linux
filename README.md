## Credits & Disclaimer

First, full credit goes to [@leopalladium](https://github.com/leopalladium) and the original [FocalTech FT9366 Arch Shim](https://github.com/leopalladium/focaltech-ft9366-arch-shim) repository, which made this setup possible.

This is simply my personal documentation of getting the fingerprint reader working on CachyOS, with the help of LLM-guided troubleshooting and configuration. I'm sharing my approach and experience so others can learn from it.

I'm not the original developer of the driver or shim, and these instructions are not guaranteed to work on every system. Use your own judgement, review commands before running them, and always keep a password-based recovery option.


# FocalTech FT9366 Fingerprint Reader on Arch Linux / CachyOS

A practical guide to enabling the FocalTech FT9366 fingerprint sensor (USB ID `2808:a658`) on Arch-based Linux distributions using a compatibility shim, a patched proprietary `libfprint` library, `fprintd`, and PAM.

This guide documents a working configuration tested on **CachyOS with KDE Plasma and Plasma Login Manager**.

> [!NOTE]
> This is a hardware-specific setup. It was tested with a FocalTech FT9366 fingerprint reader identified as `2808:a658`. Other FocalTech sensors or USB IDs may require different drivers or configuration.

## Table of Contents

- [1. Overview](#1-overview)
- [2. Tested Environment](#2-tested-environment)
- [3. How the Driver Works](#3-how-the-driver-works)
- [4. Requirements](#4-requirements)
- [5. Identify the Fingerprint Reader](#5-identify-the-fingerprint-reader)
- [6. Install the Driver and Compatibility Shim](#6-install-the-driver-and-compatibility-shim)
- [7. Configure the fprintd Service](#7-configure-the-fprintd-service)
- [8. Verify Driver Initialization](#8-verify-driver-initialization)
- [9. Enroll a Fingerprint](#9-enroll-a-fingerprint)
- [10. Test Fingerprint Verification](#10-test-fingerprint-verification)
- [11. Enable Fingerprint Authentication in KDE](#11-enable-fingerprint-authentication-in-kde)
- [12. Enable Fingerprint Authentication at Login](#12-enable-fingerprint-authentication-at-login)
- [13. Password Fallback](#13-password-fallback)
- [14. Security Recommendations: Do and Don't](#14-security-recommendations-do-and-dont)
- [15. Troubleshooting](#15-troubleshooting)
- [16. Rollback and Recovery](#16-rollback-and-recovery)
- [17. Important Notes](#17-important-notes)
- [18. References](#18-references)

---

## 1. Overview

The FocalTech FT9366 fingerprint sensor may not work with the standard `libfprint` library supplied by a modern Arch-based distribution.

The driver included in this setup uses a proprietary FocalTech-enabled `libfprint` binary. It was originally built against an older version of `libgusb`, whereas modern Arch-based distributions ship newer versions.

This creates a binary compatibility problem.

The compatibility shim provided by this project allows the older driver to work with the newer system libraries without downgrading the system-wide `libgusb` installation.

### Features achieved

- FT9366 hardware detection.
- Fingerprint enrollment.
- Fingerprint verification.
- Fingerprint authentication at the KDE Plasma lock screen.
- Fingerprint authentication at the Plasma Login Manager login screen.
- Password fallback when fingerprint authentication times out.
- Original system `libfprint` library left untouched.

**Authentication status in the tested configuration:**

| Feature | Result |
|---|---|
| Hardware detection | Working |
| Fingerprint enrollment | Working |
| Fingerprint verification | Working |
| KDE lock screen | Working |
| Plasma Login Manager | Working |
| Password fallback | Working |
| Fingerprint authentication for sudo | Intentionally disabled |

## 2. Tested Environment

The following environment was used for the working setup.

| Component | Tested configuration |
|---|---|
| Operating system | CachyOS |
| Distribution family | Arch Linux |
| Desktop environment | KDE Plasma |
| Display manager | Plasma Login Manager |
| Plasma Login Manager package | `plasma-login-manager 6.8.2-1` |
| Fingerprint hardware | FocalTech FT9366 |
| USB vendor ID | `2808` |
| USB product ID | `a658` |
| `libfprint` | `1.94.100-2` |
| `fprintd` | `1.94.5-2.1` |
| `libgusb` | `0.4.9-2.1` |
| Shell | Fish |

Package versions may change over time. The important part is that the setup uses the FocalTech-enabled library and its compatibility shim.

## 3. How the Driver Works

The setup consists of three main components.

### A. FocalTech-enabled libfprint

The repository provides a proprietary `libfprint-2.so.2.0.0` binary containing the FocalTech driver.

It includes FT9366-specific initialization and communication functions that are not present in the standard system library.

### B. Compatibility shim

The original driver expects older `libgusb` symbols.

The shim provides compatibility with the newer library by exposing the legacy symbol versions and redirecting calls to available modern functions.

### C. fprintd

`fprintd` is the fingerprint management daemon.

It communicates with the fingerprint device and provides the interface used by PAM and the fingerprint enrollment and verification commands.

### Why install under /opt?

The tested installation uses:

```text
/opt/focaltech/
├── libfprint-2.so.2.0.0
├── libfprint-2.so.2 -> libfprint-2.so.2.0.0
├── libfprint-2.so -> libfprint-2.so.2.0.0
└── focaltech-shim-test.so
```

The custom library is kept separate from the original system library:

```text
/usr/lib/libfprint-2.so.2.0.0
```

The `fprintd` service is configured to use the custom library through `LD_LIBRARY_PATH` and load the shim through `LD_PRELOAD`.

This avoids replacing the system-wide `libfprint` library.

> The tested installation used `focaltech-shim-test.so`. Check the actual filename in your checkout before copying it. Do not rename or substitute a different shim without verifying that it works with the supplied binary.

## 4. Requirements

Install the required packages:

```bash
sudo pacman -S fprintd libfprint gcc pkgconf libgusb patchelf
```

These packages provide:

- `fprintd`: fingerprint management daemon.
- `libfprint`: fingerprint framework.
- `gcc`: compiler.
- `pkgconf`: compiler and library configuration helper.
- `libgusb`: USB communication library.
- `patchelf`: binary modification utility, useful for other installation methods.

Check that the fingerprint daemon is installed:

```bash
pacman -Q fprintd
```

Check that the PAM module is available:

```bash
find /usr/lib/security -name 'pam_fprintd.so'
```

Expected output:

```text
/usr/lib/security/pam_fprintd.so
```

## 5. Identify the Fingerprint Reader

Connect the fingerprint reader and run:

```bash
lsusb -d 2808:a658
```

Expected output resembles:

```text
Bus 003 Device 004: ID 2808:a658 Realtek USB2.0 Finger Print Bridge FocalTech Fingerprint Device
```

The important part is:

```text
ID 2808:a658
```

If your device has a different USB ID, do not assume this driver will work.

## 6. Install the Driver and Compatibility Shim

Clone the repository:

```bash
git clone https://github.com/leopalladium/focaltech-ft9366-arch-shim.git
```

Enter the directory:

```bash
cd focaltech-ft9366-arch-shim
```

Check the available files:

```bash
ls -lh
```

The tested setup requires:

```text
libfprint-2.so.2.0.0
focaltech-shim-test.so
```

Create the isolated installation directory:

```bash
sudo mkdir -p /opt/focaltech
```

Copy the custom library:

```bash
sudo install -m 755 libfprint-2.so.2.0.0 /opt/focaltech/
```

Copy the shim using the filename present in your tested checkout:

```bash
sudo install -m 755 focaltech-shim-test.so /opt/focaltech/
```

Create the library symlinks:

```bash
sudo ln -sfn libfprint-2.so.2.0.0 /opt/focaltech/libfprint-2.so
sudo ln -sfn libfprint-2.so.2.0.0 /opt/focaltech/libfprint-2.so.2
```

Check the directory:

```bash
ls -l /opt/focaltech
```

### Verify the binary

The SHA256 checksum recorded for the tested library was:

```text
db95bcdf068c63a16d6fe37a55238d1dafcbe9e4d61387b2cf0bd4a631712cf4
```

Verify it:

```bash
sha256sum /opt/focaltech/libfprint-2.so.2.0.0
```

The checksum should match if you are using the same binary.

**Important:** A different checksum does not automatically mean a binary is malicious or broken. It may indicate that the repository binary has changed. Verify its source and compatibility before using it.

## 7. Configure the fprintd Service

The custom library and shim must be loaded when `fprintd` starts.

Create the systemd override directory:

```bash
sudo mkdir -p /etc/systemd/system/fprintd.service.d
```

Open the override:

```bash
sudo vim /etc/systemd/system/fprintd.service.d/override.conf
```

Add:

```ini
[Service]
Environment="G_MESSAGES_DEBUG=all"
Environment="LIBUSB_DEBUG=4"
Environment="LD_LIBRARY_PATH=/opt/focaltech"
Environment="LD_PRELOAD=/opt/focaltech/focaltech-shim-test.so"
```

Save the file.

Reload systemd:

```bash
sudo systemctl daemon-reload
```

Restart the fingerprint daemon:

```bash
sudo systemctl restart fprintd.service
```

Check its status:

```bash
systemctl status fprintd.service
```

The service should initialize without a library or symbol lookup error.

### About debugging variables

The following variables were used during the working setup:

```ini
Environment="G_MESSAGES_DEBUG=all"
Environment="LIBUSB_DEBUG=4"
```

They produce detailed diagnostic logs.

Once the installation is stable, you can remove these two lines to reduce log output.

Keep the library and shim environment variables:

```ini
Environment="LD_LIBRARY_PATH=/opt/focaltech"
Environment="LD_PRELOAD=/opt/focaltech/focaltech-shim-test.so"
```

Then reload systemd and restart the service.

## 8. Verify Driver Initialization

Check the service logs:

```bash
journalctl -u fprintd.service -b --no-pager
```

Look for FocalTech-specific initialization messages.

The working setup produced messages similar to:

```text
fpi_device_focaltech_init
Device reported probe completion
Device FocalTech Systems Co., Ltd fingerprint scan type changed to 'press'
Device FocalTech Systems Co., Ltd fingerprint enroll stages changed to 13
```

These indicate that the FocalTech driver initialized and the device was detected.

If you see an error similar to:

```text
undefined symbol: g_usb_device_get_interfaces
```

the compatibility shim is not being loaded correctly, or the custom library and shim are incompatible.

Check:

```bash
cat /etc/systemd/system/fprintd.service.d/override.conf
```

Also check that the paths in the override match the actual files in `/opt/focaltech`.

## 9. Enroll a Fingerprint

Start enrollment:

```bash
fprintd-enroll
```

To explicitly enroll the right index finger:

```bash
fprintd-enroll -f right-index-finger
```

Follow the instructions and repeatedly place your finger on the reader.

Wait until enrollment completes.

Expected result:

```text
Enroll result: enroll-completed
```

### List enrolled fingerprints

```bash
fprintd-list "$USER"
```

Example:

```text
found 1 devices
Device at /net/reactivated/Fprint/Device/0
Fingerprints for user sai on FocalTech Systems Co., Ltd fingerprint (press):
 - #0: right-index-finger
```

## 10. Test Fingerprint Verification

Run:

```bash
fprintd-verify
```

Place your enrolled finger on the reader.

Expected result:

```text
Verify result: verify-match (done)
```

If verification succeeds, the following components are working:

- USB device communication.
- FocalTech driver.
- Compatibility shim.
- Fingerprint enrollment database.
- Fingerprint matching.

Do not proceed to PAM configuration until enrollment and verification work successfully.

## 11. Enable Fingerprint Authentication in KDE

KDE uses PAM for lock-screen authentication.

On the tested system, the original configuration was located at:

```text
/usr/lib/pam.d/kde
```

The system also provided:

```text
/usr/lib/pam.d/kde-fingerprint
/usr/lib/pam.d/kde-smartcard
```

The existing `kde-fingerprint` profile was not used because it was a separate fingerprint-only configuration and did not provide the password fallback required for this setup.

### Step 1: Copy the original configuration

```bash
sudo cp /usr/lib/pam.d/kde /etc/pam.d/kde
```

Create a backup:

```bash
sudo cp /etc/pam.d/kde /etc/pam.d/kde.backup
```

### Step 2: Edit the local configuration

```bash
sudo vim /etc/pam.d/kde
```

Original authentication line:

```text
auth       include                     system-local-login
```

Change it to:

```text
auth       sufficient                  pam_fprintd.so timeout=10
auth       include                     system-local-login
```

Keep the remaining account, password, and session rules unchanged.

The beginning of the resulting file should look like:

```text
#%PAM-1.0

auth       sufficient                  pam_fprintd.so timeout=10
auth       include                     system-local-login

account    include                     system-local-login

password   include                     system-local-login

session    include                     system-local-login
```

Save the file.

### Step 3: Test the lock screen

1. Save your work.
2. Press `Meta + L`.
3. Wait for the lock screen.
4. Place your enrolled finger on the sensor.
5. If fingerprint authentication does not succeed, wait for the timeout and use your password.

The tested configuration successfully unlocked KDE using the fingerprint reader.

## 12. Enable Fingerprint Authentication at Login

The tested display manager was:

```text
Plasma Login Manager
```

Its system PAM configuration was:

```text
/usr/lib/pam.d/plasmalogin
```

The local configuration override belongs in:

```text
/etc/pam.d/plasmalogin
```

### Step 1: Copy the original PAM configuration

```bash
sudo cp /usr/lib/pam.d/plasmalogin /etc/pam.d/plasmalogin
```

### Step 2: Edit the local file

```bash
sudo vim /etc/pam.d/plasmalogin
```

Find:

```text
auth        include     system-login
```

Insert the fingerprint rule immediately above it:

```text
auth        sufficient  pam_fprintd.so timeout=10
auth        include     system-login
```

Keep the remaining original lines unchanged.

The beginning should look like:

```text
#%PAM-1.0

auth        sufficient  pam_fprintd.so timeout=10
auth        include     system-login
-auth       optional    pam_gnome_keyring.so
-auth       optional    pam_kwallet5.so
```

Save the file.

### Step 3: Test login

Save all open work before logging out.

1. Log out of KDE Plasma.
2. Wait for Plasma Login Manager.
3. Select your user.
4. Leave the password field empty.
5. Press `Enter`.
6. Wait for fingerprint authentication.
7. Scan your enrolled finger.

The tested setup successfully authenticated after pressing Enter with an empty password field.

### Login-screen behavior

One important detail:

**The fingerprint prompt may not appear immediately when the login screen first loads.**

In the tested Plasma Login Manager setup, the user had to:

1. Select the account.
2. Leave the password field empty.
3. Press Enter.

Only then did the fingerprint authentication process start.

This is a behavior of the login manager's authentication flow, not necessarily a problem with the fingerprint driver.

## 13. Password Fallback

The tested configuration uses:

```text
pam_fprintd.so timeout=10
```

The `timeout` option controls how long the fingerprint authentication module waits before returning an authentication failure.

The default timeout is 30 seconds. The tested configuration uses 10 seconds.

If fingerprint authentication fails or times out, PAM proceeds to the existing password authentication rules.

### Important PAM behavior

PAM authentication is sequential.

Fingerprint and password authentication do not run simultaneously through the standard `pam_fprintd` module.

For the graphical login and KDE lock screen, the existing password stack remains below the fingerprint rule.

A user can therefore wait for the fingerprint attempt to time out and then use the password.

Do not assume pressing Enter will cancel an active fingerprint scan. In the tested `sudo` experiment, Enter did not cancel the scan; waiting for the timeout did.

## 14. Security Recommendations: Do and Don't

### DO

- Keep password authentication available.
- Enroll and verify fingerprints before editing PAM files.
- Back up PAM files before changing them.
- Make local overrides in `/etc/pam.d/` instead of editing package files in `/usr/lib/pam.d/`.
- Keep the original system `libfprint` library untouched.
- Keep the custom driver isolated under `/opt/focaltech/`.
- Keep a working terminal session open when testing PAM changes.
- Know your Linux password before logging out.
- Check PAM syntax carefully.
- Test the lock screen before testing the login screen.
- Keep the original configuration files for recovery.

### DON'T

- Do not overwrite `/usr/lib/libfprint-2.so.2.0.0`.
- Do not downgrade system `libgusb` just to satisfy the proprietary driver.
- Do not blindly replace `/etc/pam.d/system-auth`.
- Do not blindly replace `/etc/pam.d/system-login`.
- Do not edit multiple shared PAM files without understanding their effects.
- Do not remove password fallback.
- Do not reboot immediately after making PAM changes.
- Do not assume that a fingerprint prompt appearing means authentication has succeeded.
- Do not use `timeout=-1` without understanding its implications.
- Do not enable fingerprint authentication for privileged services such as `sudo`, `su`, or Polkit using the simple `sufficient` configuration.

### Why fingerprint authentication is not enabled for sudo

Arch Linux documents a fingerprint authentication hijacking vulnerability associated with using `pam_fprintd.so` as a sufficient authentication method for privileged services.

See CVE-2024-37408.

For this reason, the tested setup deliberately restores the original password-only sudo configuration.

The working configuration uses fingerprints for:

- KDE lock-screen authentication.
- Plasma Login Manager authentication.

It does not use fingerprints for:

- `sudo`.
- `su`.
- Polkit authentication.

This is an intentional security decision, not a driver limitation.

## 15. Troubleshooting

### Problem: Fingerprint device not detected

Check the USB ID:

```bash
lsusb -d 2808:a658
```

Check the service:

```bash
systemctl status fprintd.service
```

Check the logs:

```bash
journalctl -u fprintd.service -b --no-pager
```

Check that the custom library and shim exist:

```bash
ls -lh /opt/focaltech/
```

### Problem: Undefined libgusb symbols

Example:

```text
undefined symbol: g_usb_device_get_interfaces
```

Possible causes:

- The compatibility shim is missing.
- The shim path is incorrect.
- The custom library is not being loaded.
- The shim and driver binary are incompatible.

Check the systemd override:

```bash
cat /etc/systemd/system/fprintd.service.d/override.conf
```

Verify that `LD_LIBRARY_PATH` and `LD_PRELOAD` point to the actual installed files.

### Problem: Fingerprint enrollment fails

Check:

```bash
fprintd-list "$USER"
```

Then inspect:

```bash
journalctl -u fprintd.service -b --no-pager
```

Try restarting the service:

```bash
sudo systemctl restart fprintd.service
```

Then retry:

```bash
fprintd-enroll -f right-index-finger
```

### Problem: Fingerprint prompt appears late at login

This was observed in the tested Plasma Login Manager configuration.

Try:

1. Select your user.
2. Leave the password field empty.
3. Press Enter.
4. Wait for the fingerprint reader.

The fingerprint process may start only after this action.

### Problem: Password does not appear immediately

The fingerprint PAM module waits for its configured timeout.

With:

```text
timeout=10
```

wait for the fingerprint attempt to time out before expecting the password authentication stage.

### Problem: KDE Wallet still requests a password

Fingerprint authentication does not unlock the KDE Wallet encryption key.

The wallet may continue to request your account password, even after successful fingerprint authentication.

This is expected behavior.

### Problem: PAM changes cause login failure

Do not keep trying random PAM modifications.

Use an existing terminal session or switch to a TTY with:

```text
Ctrl + Alt + F3
```

Log in with your normal password and restore the original PAM files.

See the rollback instructions below.

## 16. Rollback and Recovery

Always keep a recovery path available when changing PAM.

### Restore sudo

If a backup exists:

```bash
sudo cp /etc/pam.d/sudo.backup /etc/pam.d/sudo
```

The recommended final setup keeps sudo password-only.

### Restore KDE lock-screen PAM

```bash
sudo cp /etc/pam.d/kde.backup /etc/pam.d/kde
```

Alternatively, remove the local override to return to the packaged configuration:

```bash
sudo rm /etc/pam.d/kde
```

### Restore Plasma Login Manager PAM

The original package configuration remains available at:

```text
/usr/lib/pam.d/plasmalogin
```

Remove the local override:

```bash
sudo rm /etc/pam.d/plasmalogin
```

### Restore the fingerprint service

Remove the custom systemd override:

```bash
sudo rm /etc/systemd/system/fprintd.service.d/override.conf
```

Reload systemd:

```bash
sudo systemctl daemon-reload
```

Restart the service:

```bash
sudo systemctl restart fprintd.service
```

Only remove `/opt/focaltech` after the service override has been removed and you have confirmed that the original driver is sufficient for your system.

**Do not delete the original system library.**

## 17. Important Notes

### Original system library remains untouched

The tested configuration stores the custom library in:

```text
/opt/focaltech/
```

The system library remains in:

```text
/usr/lib/
```

This separation reduces the risk of breaking other applications that depend on the system version of `libfprint`.

### Udev rules

The repository also contains:

```text
60-libfprint-2.rules
```

The tested setup successfully detected and used the fingerprint device without installing this rule file.

Do not assume it is required on every machine.

If device permissions are a problem, inspect the rules first. In particular, do not blindly assume the `plugdev` group exists on Arch-based distributions.

The repository's upstream instructions also describe a USB autosuspend workaround. Treat it as an optional troubleshooting measure rather than a mandatory step for this verified configuration.

### Proprietary driver

The FocalTech-enabled driver binary is proprietary software.

Check the applicable license and redistribution terms before republishing the binary or bundling it in another project.

The compatibility shim source and the proprietary driver binary may have different licensing terms.

### Updates

A system update can change:

- `libfprint`.
- `libgusb`.
- `fprintd`.
- PAM defaults.
- Plasma Login Manager.
- KDE lock-screen behavior.

After major updates, verify:

```bash
systemctl status fprintd.service
fprintd-list "$USER"
fprintd-verify
```

Also test the KDE lock screen and login screen.

Do not assume a setup that works on one Arch-based distribution will work identically on another.

## 18. References

- [FocalTech FT9366 Arch Shim Repository](https://github.com/leopalladium/focaltech-ft9366-arch-shim)
- [ArchWiki – Fprint](https://wiki.archlinux.org/title/Fprint)
- [Arch Linux – pam_fprintd Manual](https://man.archlinux.org/man/extra/fprintd/pam_fprintd.8.en)
- [Arch Linux – fprintd Manual](https://man.archlinux.org/man/fprintd.1)
- [KDE Plasma Login Manager](https://github.com/KDE/plasma-login-manager)
- [CVE-2024-37408](https://nvd.nist.gov/vuln/detail/CVE-2024-37408)

---

## Final Result

The FocalTech FT9366 fingerprint reader was successfully configured on CachyOS with KDE Plasma and Plasma Login Manager.

The final working arrangement is:

- Custom FocalTech-enabled `libfprint` installed under `/opt/focaltech`.
- Compatibility shim loaded by the `fprintd` systemd service.
- Fingerprint enrollment and verification working.
- KDE lock-screen fingerprint authentication enabled.
- Plasma Login Manager fingerprint authentication enabled.
- Password fallback preserved.
- Original system `libfprint` left untouched.
- `sudo` remains password-only for security.

**The most important lesson:** Get the driver working first, verify enrollment second, and only then modify PAM. Always preserve password fallback and keep a recovery session open when changing authentication configuration.
