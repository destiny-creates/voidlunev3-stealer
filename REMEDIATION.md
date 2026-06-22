# VoidLuneV3 Infostealer — Victim Remediation Guide

> **If you believe you have been infected by VoidLuneV3, follow this guide immediately.**
> Time is critical — assume all credentials and tokens are already compromised.

---

## Table of Contents

1. [Immediate Containment](#1-immediate-containment)
2. [Confirm Infection](#2-confirm-infection)
3. [Remove the Malware](#3-remove-the-malware)
4. [Rotate All Credentials](#4-rotate-all-credentials)
5. [Secure Your Discord Account](#5-secure-your-discord-account)
6. [Secure Cryptocurrency Wallets](#6-secure-cryptocurrency-wallets)
7. [Check for Unauthorized Access](#7-check-for-unauthorized-access)
8. [Harden Your System](#8-harden-your-system)
9. [Report the Incident](#9-report-the-incident)
10. [Technical IOC Reference](#10-technical-ioc-reference)

---

## 1. Immediate Containment

**Do these steps FIRST before anything else:**

- [ ] **Disconnect from the internet** (unplug ethernet / disable Wi-Fi)
- [ ] **Do NOT log into any accounts** from this machine until it is cleaned
- [ ] **Do NOT open any browser** on the infected machine
- [ ] If you have cryptocurrency wallets — **do not transact** until seeds are moved to a new wallet
- [ ] Take a screenshot or photo of your screen to document current state

---

## 2. Confirm Infection

### Check for Dropped Files

Open File Explorer, paste each path into the address bar:

```
%LOCALAPPDATA%
```

Look for any folder containing both:
- `bin\javam.exe`
- `lib\salams.jar`

Any folder matching this pattern (`%LOCALAPPDATA%\[random name]\bin\javam.exe`) is the malware drop directory.

### Check for Running Processes

Press `Ctrl+Shift+Esc` -> Task Manager -> Details tab.
Look for:
- `javam.exe` running from any location inside `AppData`

### Check for Network Connections (if still online)

Open PowerShell as Administrator and run:
```powershell
netstat -ano | findstr "ESTABLISHED"
```
Look for connections to:
- `swordfull.com` (any IP)
- `upload.gofile.io`

---

## 3. Remove the Malware

### Step 1 — Kill the Java process

Open Task Manager -> Details tab -> find `javam.exe` -> Right-click -> End Task.

Or in PowerShell (as Administrator):
```powershell
Stop-Process -Name "javam" -Force -ErrorAction SilentlyContinue
```

### Step 2 — Delete the drop directory

In PowerShell (as Administrator):
```powershell
# Find and remove all instances of the drop directory
Get-ChildItem "$env:LOCALAPPDATA" -Directory | ForEach-Object {
    $binPath = Join-Path $_.FullName "bin\javam.exe"
    $libPath = Join-Path $_.FullName "lib\salams.jar"
    if ((Test-Path $binPath) -or (Test-Path $libPath)) {
        Write-Host "Found malware directory: $($_.FullName)" -ForegroundColor Red
        Remove-Item $_.FullName -Recurse -Force
        Write-Host "Removed: $($_.FullName)" -ForegroundColor Green
    }
}
```

### Step 3 — Verify removal

```powershell
# Confirm no javam.exe files remain
Get-ChildItem "$env:LOCALAPPDATA" -Recurse -Filter "javam.exe" -ErrorAction SilentlyContinue
Get-ChildItem "$env:LOCALAPPDATA" -Recurse -Filter "salams.jar" -ErrorAction SilentlyContinue
# Should return no results
```

### Step 4 — Run a reputable antivirus scan

After manual removal, run a full system scan with:
- Windows Defender (built-in)
- Malwarebytes Free
- HitmanPro

> **Note:** AV may not detect this sample due to its heavy obfuscation and custom JVM. Manual removal steps above are the most reliable.

---

## 4. Rotate All Credentials

**Assume every password stored in any browser on this machine is stolen.**

### Priority order for password rotation:

1. **Email accounts** (Gmail, Outlook, Yahoo) — these can be used to reset everything else
2. **Banking and financial accounts**
3. **Work/corporate accounts** — notify your IT/security team immediately
4. **Social media** (Twitter/X, Facebook, Instagram, LinkedIn)
5. **Shopping accounts** (Amazon, eBay, PayPal)
6. **Gaming accounts** (Steam, Epic, Battle.net)
7. **All remaining saved passwords**

### How to safely rotate passwords:

- **Use a different, clean device** (phone, another computer) to change passwords
- Change to **strong, unique passwords** for every account (use a password manager)
- **Enable 2FA/MFA** on every account that supports it, using an authenticator app (not SMS where possible)
- After enabling 2FA, **revoke all active sessions** on each account

### Browsers to check for saved credentials:

This malware targeted all of the following — clear saved passwords from each:

- Google Chrome
- Microsoft Edge  
- Mozilla Firefox
- Brave Browser
- Opera / Opera GX
- Vivaldi
- Yandex Browser
- Any other Chromium or Firefox-based browser

To clear saved passwords in Chrome/Edge/Brave:
`Settings -> Autofill -> Passwords -> Delete all`

To clear saved passwords in Firefox:
`Settings -> Privacy & Security -> Saved Logins -> Remove All`

---

## 5. Secure Your Discord Account

**Your Discord token was almost certainly stolen.** A stolen token gives an attacker full access to your account without needing your password.

### Immediate steps:

1. **Change your Discord password** immediately (from a clean device)
   - This invalidates all existing tokens
2. **Enable Two-Factor Authentication (2FA)** if not already enabled
   - `User Settings -> My Account -> Enable Two-Factor Auth`
3. **Log out of all devices**
   - `User Settings -> My Account -> Remove All Devices` (scroll down)
4. **Check for unauthorized OAuth apps**
   - `User Settings -> Authorized Apps` -> Revoke anything you don't recognize
5. **Check your connected accounts**
   - `User Settings -> Connections` -> Remove any unrecognized connections
6. **Review your account for suspicious activity**
   - Check DMs sent from your account
   - Check servers you may have been added to
   - Check if your account was used to send phishing messages
7. **Enable phone number 2FA as backup**

### If you have Nitro:
The malware specifically profiles Nitro accounts. Consider whether your Nitro subscription details were exposed.

### Notify your Discord server members:
If you have any server admin roles, notify your members that your account may have been briefly compromised.

---

## 6. Secure Cryptocurrency Wallets

**This is the highest-urgency action if you hold any cryptocurrency.**

The malware targeted desktop wallets AND browser extension wallets. Assume **all wallet data on this machine was stolen**.

### If your wallet files were present on this machine:

> **CRITICAL: Do NOT send funds to a new wallet from the infected machine.**
> The malware may still be running or a keylogger could be active.

1. **Use a clean device** (freshly installed OS, or a device that was never infected)
2. **Create entirely new wallets** with new seed phrases on the clean device
3. **Write down the new seed phrase** on paper — never store it digitally
4. **Transfer all funds** from your old wallets to the new wallets
5. **Never use the old seed phrases again**

### Browser extension wallets specifically targeted:

If you have ANY of these extensions installed, treat them as compromised:

- MetaMask, Phantom, Coinbase Wallet, Trust Wallet
- Keplr, Ronin, Yoroi, Nami, Petra, Martian
- Math Wallet, Binance Chain Wallet, XDEFI, Liquality
- Safepal, Tokenpocket, Kaikas, TerraStation, Coin98
- Any other crypto wallet browser extension

### Desktop wallets targeted:

- Exodus, Electrum, Atomic Wallet, Guarda, Coinomi
- Jaxx Liberty, Zcash, Armory, Bytecoin
- Any Ethereum keystore files

### If funds have already been moved by the attacker:

- Document the transaction hashes
- Report to the relevant blockchain's fraud tracking community
- File a report with your local cybercrime unit
- For large amounts, consider contacting a blockchain forensics firm

---

## 7. Check for Unauthorized Access

After rotating credentials, check each account for signs of unauthorized access:

### Email
- Check "Sent" folder for emails you didn't send
- Check "Forwarding" rules for auto-forward rules added by attacker
- Check "Authorized apps" / connected apps
- Check login history for unfamiliar IPs/locations

### Banking
- Review all recent transactions
- Check for new payees or beneficiaries added
- Check for any new cards or accounts opened
- Contact your bank if anything looks suspicious

### Social Media / Gaming
- Check account activity logs
- Check for messages sent from your account
- Check for purchases made (Steam, Epic Games store, etc.)

### Corporate / Work Accounts
- **Notify your IT security team immediately**
- Your employer's systems may be at risk
- Check for any files accessed or downloaded

---

## 8. Harden Your System

### After cleaning the infection:

1. **Keep Windows updated** — ensure all security patches are applied

2. **Use a password manager** (Bitwarden, 1Password, KeePass)
   - Stop saving passwords in browsers
   - Use a unique strong password for every account

3. **Enable 2FA on everything** — use an authenticator app (Aegis, Authy, Google Authenticator)

4. **Be extremely cautious about what you download and run**
   - This malware was distributed as a fake application
   - Only download software from official websites
   - Be suspicious of links in Discord DMs, emails, and social media

5. **Consider using a hardware wallet** for cryptocurrency (Ledger, Trezor)
   - Hardware wallets are immune to this type of attack

6. **Install a reputable endpoint security solution**

7. **Network protection** — if you have a router with DNS filtering, block:
   - `swordfull.com`

8. **Browser hardening:**
   - Remove any browser extensions you don't recognize
   - Consider using separate browser profiles for different activities
   - Enable enhanced tracking protection

### EDR / Detection Rules

For security teams, detect this malware with the following behavioral rules:

```
ALERT: Java process spawned from %LOCALAPPDATA%\*\bin\ with -jar flag
ALERT: javam.exe executing from AppData subdirectory
ALERT: New directory created under %LOCALAPPDATA% containing both bin/ and lib/ subdirectories
ALERT: POST request to swordfull.com
ALERT: Java process making HTTP POST to upload.gofile.io
ALERT: Electron app (parent) spawning detached Java child process
```

---

## 9. Report the Incident

### Report to authorities:
- **US:** FBI IC3 — ic3.gov
- **UK:** Action Fraud — actionfraud.police.uk
- **EU:** Europol — europol.europa.eu/report-a-crime
- **Australia:** ACSC — cyber.gov.au

### Report the C2 domain:
- Report `swordfull.com` to the domain registrar (look up via whois)
- Report to Google Safe Browsing: safebrowsing.google.com/safebrowsing/report_phish/
- Report to abuse.ch: abuse.ch

### Report to Discord Trust & Safety:
- If your Discord account was used to spread this malware, report it
- discord.com/safety

### Share IOCs with the security community:
- VirusTotal
- MalwareBazaar (bazaar.abuse.ch)
- AnyRun sandbox

---

## 10. Technical IOC Reference

For use by IT teams and security professionals:

### File Indicators
```
%LOCALAPPDATA%\[random]\bin\javam.exe     (dropped JVM)
%LOCALAPPDATA%\[random]\lib\salams.jar    (Java payload)
Filename: javam.exe (masquerading as legitimate JVM)
Filename: salams.jar (obfuscated with Zelix KlassMaster v26.0.2)
```

### Network Indicators
```
Domain:  swordfull.com  [BLOCK]
Host:    upload.gofile.io  [monitor for Java process connections]
URL:     https://swordfull.com/m/
URL:     https://upload.gofile.io/uploadfile
Method:  POST (both endpoints)
```

### Process Indicators
```
Process name: javam.exe
Parent:       Electron app (or orphaned/detached)
Location:     %LOCALAPPDATA%\*\bin\javam.exe
CLI args:     -jar [path]\lib\salams.jar
Window:       None (hidden, no console)
```

### String Signatures (YARA)
```yara
rule VoidLuneV3_Infostealer {
    meta:
        description = "VoidLuneV3 infostealer - Electron+Java multi-stage"
        author = "Malware Analysis"
        date = "2026-06-22"
        severity = "CRITICAL"
    strings:
        $s1 = "VoidLuneV3" ascii wide
        $s2 = "salams.jar" ascii
        $s3 = "javam.exe" ascii
        $s4 = "swordfull.com" ascii wide
        $s5 = "ZKM26.0.2" ascii
        $s6 = "https://upload.gofile.io/uploadfile" ascii wide
        $s7 = "https://swordfull.com/m/" ascii wide
    condition:
        2 of them
}
```

### PowerShell Detection Script
```powershell
# Run as Administrator to check for VoidLuneV3 indicators
Write-Host "=== VoidLuneV3 Indicator Check ==" -ForegroundColor Cyan

# Check for dropped files
$found = $false
Get-ChildItem "$env:LOCALAPPDATA" -Directory | ForEach-Object {
    $bin = Join-Path $_.FullName "bin\javam.exe"
    $lib = Join-Path $_.FullName "lib\salams.jar"
    if ((Test-Path $bin) -or (Test-Path $lib)) {
        Write-Host "[INFECTED] Found malware drop: $($_.FullName)" -ForegroundColor Red
        $found = $true
    }
}
if (-not $found) { Write-Host "[OK] No drop directory found" -ForegroundColor Green }

# Check for running process
$proc = Get-Process -Name "javam" -ErrorAction SilentlyContinue
if ($proc) {
    Write-Host "[INFECTED] javam.exe is running! PID: $($proc.Id)" -ForegroundColor Red
} else {
    Write-Host "[OK] javam.exe not running" -ForegroundColor Green
}

# Check network connections
$conns = netstat -ano | Select-String "swordfull"
if ($conns) {
    Write-Host "[INFECTED] Active C2 connection detected!" -ForegroundColor Red
} else {
    Write-Host "[OK] No C2 connections detected" -ForegroundColor Green
}

Write-Host "=== Check complete ==" -ForegroundColor Cyan
```

---

*This remediation guide was produced as part of the VoidLuneV3 malware analysis.*
*For the full technical analysis, see README.md.*
