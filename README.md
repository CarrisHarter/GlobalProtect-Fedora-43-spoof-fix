# GlobalProtect VPN Fix for Fedora 43

**Fix "Not Acceptable" OS version error when connecting to GlobalProtect VPN on Fedora 43**

Many enterprise VPN gateways reject Fedora 43 because it's too new and not on their approved OS list. This guide shows you how to spoof your OS version to Fedora 42 so GlobalProtect HIP checks will pass.

---

## The Problem

When connecting to GlobalProtect VPN on Fedora 43, you get:
- ❌ "Not Acceptable" error
- ❌ Connection immediately rejected
- ❌ HIP check reports `Linux Fedora 43` which isn't approved

## The Solution

Spoof your OS as Fedora 42 by modifying system release files and creating an automatic boot script.

---

## Quick Install (Automated)

### Step 1: Create the Spoof Script

```bash
sudo tee /usr/local/sbin/gp-fedora-spoof.sh > /dev/null <<'EOF'
#!/bin/bash
#
# GlobalProtect Fedora Version Spoofer
# Automatically spoofs Fedora 43 -> Fedora 42 for GlobalProtect HIP compliance
#

set -e

# Check if running as root
if [ "$EUID" -ne 0 ]; then 
    echo "Error: This script must be run as root"
    exit 1
fi

BACKUP_SUFFIX="bak-original"
OS_RELEASE_FILE="/usr/lib/os-release"
FEDORA_RELEASE_FILE="/usr/lib/fedora-release"

# Create backups if they don't exist
if [ ! -f "${OS_RELEASE_FILE}.${BACKUP_SUFFIX}" ]; then
    echo "Creating backup: ${OS_RELEASE_FILE}.${BACKUP_SUFFIX}"
    cp "${OS_RELEASE_FILE}" "${OS_RELEASE_FILE}.${BACKUP_SUFFIX}"
fi

if [ ! -f "${FEDORA_RELEASE_FILE}.${BACKUP_SUFFIX}" ]; then
    echo "Creating backup: ${FEDORA_RELEASE_FILE}.${BACKUP_SUFFIX}"
    cp "${FEDORA_RELEASE_FILE}" "${FEDORA_RELEASE_FILE}.${BACKUP_SUFFIX}"
fi

# Spoof os-release to Fedora 42
echo "Spoofing ${OS_RELEASE_FILE} to Fedora 42..."
cat > "${OS_RELEASE_FILE}" <<'OSRELEASE'
NAME="Fedora Linux"
VERSION="42 (Workstation Edition)"
RELEASE_TYPE=stable
ID=fedora
VERSION_ID=42
VERSION_CODENAME=""
PRETTY_NAME="Fedora Linux 42 (Workstation Edition)"
ANSI_COLOR="0;38;2;60;110;180"
LOGO=fedora-logo-icon
CPE_NAME="cpe:/o:fedoraproject:fedora:42"
DEFAULT_HOSTNAME="fedora"
HOME_URL="https://fedoraproject.org/"
DOCUMENTATION_URL="https://docs.fedoraproject.org/en-US/fedora/f42/"
SUPPORT_URL="https://ask.fedoraproject.org/"
BUG_REPORT_URL="https://bugzilla.redhat.com/"
REDHAT_BUGZILLA_PRODUCT="Fedora"
REDHAT_BUGZILLA_PRODUCT_VERSION=42
REDHAT_SUPPORT_PRODUCT="Fedora"
REDHAT_SUPPORT_PRODUCT_VERSION=42
SUPPORT_END=2025-05-14
VARIANT="Workstation Edition"
VARIANT_ID=workstation
OSRELEASE

# Spoof fedora-release to Fedora 42
echo "Spoofing ${FEDORA_RELEASE_FILE} to Fedora 42..."
cat > "${FEDORA_RELEASE_FILE}" <<'FEDRELEASE'
Fedora release 42 (Forty Two)
FEDRELEASE

echo "Spoof applied successfully!"
echo "OS now reports as Fedora 42"
EOF

sudo chmod +x /usr/local/sbin/gp-fedora-spoof.sh
```

### Step 2: Create Systemd Service (Auto-Run on Boot)

```bash
sudo tee /etc/systemd/system/gp-fedora-spoof.service > /dev/null <<'EOF'
[Unit]
Description=Spoof Fedora version for GlobalProtect HIP compliance
Before=gpd.service
DefaultDependencies=no
After=local-fs.target

[Service]
Type=oneshot
ExecStart=/usr/local/sbin/gp-fedora-spoof.sh
RemainAfterExit=yes
StandardOutput=journal
StandardError=journal

[Install]
WantedBy=multi-user.target
EOF

sudo systemctl daemon-reload
sudo systemctl enable gp-fedora-spoof.service
```

### Step 3: Run the Spoof

```bash
# Apply the spoof immediately
sudo systemctl start gp-fedora-spoof.service

# Restart GlobalProtect daemon to detect the new OS version
sudo systemctl restart gpd.service
```

### Step 4: Verify

```bash
# Check OS version (should show 42)
cat /etc/os-release | grep VERSION_ID

# Check GlobalProtect detected it correctly
sudo grep "Get OS info" /opt/paloaltonetworks/globalprotect/PanGPS.log | tail -1
# Should show: "Get OS info: Fedora 42"
```

---

## Manual Installation (Step-by-Step)

If you prefer to understand each step:

### 1. Backup Original Files

```bash
sudo cp /usr/lib/os-release /usr/lib/os-release.bak
sudo cp /usr/lib/fedora-release /usr/lib/fedora-release.bak
```

### 2. Modify `/usr/lib/os-release`

```bash
sudo nano /usr/lib/os-release
```

Change `VERSION_ID=43` to `VERSION_ID=42` and update all references from 43 → 42.

### 3. Modify `/usr/lib/fedora-release`

```bash
sudo nano /usr/lib/fedora-release
```

Change to:
```
Fedora release 42 (Forty Two)
```

### 4. Restart GlobalProtect

```bash
sudo systemctl restart gpd.service
```

---

## How It Works

1. **Boot Time**: The `gp-fedora-spoof.service` runs automatically before GlobalProtect starts
2. **OS Detection**: GlobalProtect's `PanGPS` daemon reads `/etc/os-release` at startup
3. **HIP Report**: The daemon reports `Linux Fedora 42` to the VPN gateway
4. **Gateway Accepts**: Fedora 42 is on the approved OS list ✅

---

## Verification Commands

```bash
# Check if spoof service is running
systemctl status gp-fedora-spoof.service

# Verify OS version shows Fedora 42
cat /etc/os-release | grep "VERSION_ID\|PRETTY_NAME"

# Check GlobalProtect logs
sudo grep "Get OS info" /opt/paloaltonetworks/globalprotect/PanGPS.log | tail -3

# Check HIP report sent to gateway
sudo cat /opt/paloaltonetworks/globalprotect/pan_gp_hrpt.xml | grep "<os>"
```

Expected output:
```xml
<os>Linux Fedora 42</os>
```

---

## Management Commands

```bash
# Manually run the spoof
sudo systemctl start gp-fedora-spoof.service

# Check service status
systemctl status gp-fedora-spoof.service

# Disable auto-spoof on boot
sudo systemctl disable gp-fedora-spoof.service

# Re-enable auto-spoof
sudo systemctl enable gp-fedora-spoof.service

# View logs
journalctl -u gp-fedora-spoof.service
```

---

## Restoring Original OS Version

If you need to restore your real Fedora 43 identity:

```bash
# Restore from backups
sudo cp /usr/lib/os-release.bak-original /usr/lib/os-release
sudo cp /usr/lib/fedora-release.bak-original /usr/lib/fedora-release

# Restart GlobalProtect
sudo systemctl restart gpd.service

# Disable auto-spoof
sudo systemctl disable gp-fedora-spoof.service
```

---

## After System Updates

⚠️ **Important**: Package updates may restore the original release files.

If GlobalProtect stops working after a system update:

```bash
# Reapply the spoof
sudo systemctl start gp-fedora-spoof.service
sudo systemctl restart gpd.service
```

The auto-spoof service will protect you on every boot, but manual package updates to `fedora-release` or `systemd` might override the files between boots.

---

## Troubleshooting

### GlobalProtect Still Shows "Not Acceptable"

1. **Check if spoof is active:**
   ```bash
   cat /etc/os-release | grep VERSION_ID
   ```
   Should show `VERSION_ID=42`

2. **Restart the daemon:**
   ```bash
   sudo systemctl restart gpd.service
   ```
   The daemon caches OS info at startup.

3. **Check daemon logs:**
   ```bash
   sudo grep "Get OS info" /opt/paloaltonetworks/globalprotect/PanGPS.log | tail -1
   ```
   Should show `Fedora 42`, not `Fedora 43`.

4. **Clear HIP cache and reconnect:**
   ```bash
   sudo rm -f /opt/paloaltonetworks/globalprotect/HIP_*_Report_V4.dat
   sudo rm -f /opt/paloaltonetworks/globalprotect/pan_gp_hrpt.xml
   ```
   Then disconnect and reconnect VPN.

### Service Won't Start

```bash
# Check for errors
journalctl -xeu gp-fedora-spoof.service

# Verify script is executable
ls -l /usr/local/sbin/gp-fedora-spoof.sh

# Run manually to see errors
sudo /usr/local/sbin/gp-fedora-spoof.sh
```

---

## Why This Works

GlobalProtect performs Host Integrity Profile (HIP) checks by:
1. Reading `/etc/os-release` (which is a symlink to `/usr/lib/os-release`)
2. Sending OS version to the VPN gateway
3. Gateway compares against approved OS list
4. Rejects if OS is too new/not approved

By spoofing the version to Fedora 42 (an approved version), the gateway accepts the connection.

---

## Security Considerations

- **System Integrity**: This only changes cosmetic version strings, not actual system files or security
- **Updates Work Fine**: DNF/RPM still function normally
- **No System Breakage**: Easily reversible by restoring backups
- **VPN Policy**: Check your organization's acceptable use policy before using

---

## Credits

This solution was developed to solve GlobalProtect connectivity issues on Fedora 43 systems when connecting to enterprise VPN gateways that have restrictive OS version policies.

---

## License

This guide is provided as-is for educational purposes. Use at your own risk and ensure compliance with your organization's IT policies.

---

## Additional Resources

- [GlobalProtect for Linux Documentation](https://docs.paloaltonetworks.com/globalprotect)
- [Fedora Release Information](https://docs.fedoraproject.org/)
- [systemd Service Files](https://www.freedesktop.org/software/systemd/man/systemd.service.html)

