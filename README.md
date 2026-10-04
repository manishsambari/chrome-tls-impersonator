# ssl_imp: Chrome TLS Fingerprint Impersonator (OpenSSL)

`ssl_imp` is a lightweight C project designed to impersonate **Google Chrome's TLS ClientHello fingerprint** using modern **OpenSSL (>= 3.6.0)**.

---

## 📌 Overview

Modern anti-bot solutions, web application firewalls (WAFs), and CDNs (such as Cloudflare, Akamai, and DataDome) inspect the TLS handshake to identify automated scripts. Traditional tools using OpenSSL (like standard `curl` or Python's `requests`) are easily flagged because OpenSSL's default ClientHello looks drastically different from a real web browser.

`ssl_imp` bridges this gap by configuring OpenSSL to replicate the exact TLS signature of Google Chrome (Chrome 139+), including:
- **Cipher Suite ordering & selection** (TLS 1.3 & TLS 1.2)
- **Supported Groups / Curves**, including post-quantum key exchange (`*X25519MLKEM768`, `*X25519`)
- **GREASE** (Generate Random Extensions And Sustain Extensibility) injection
- **ALPS** (Application-Layer Protocol Settings - extension `17613 / 0x44cd`)
- **ECH** (Encrypted Client Hello - extension `65037 / 0xfe0d`)
- **Certificate Compression** with **Brotli**
- **Signature Algorithms & ALPN** negotiated identically to Chrome
- **Matching HTTP Request Headers** & User-Agent string

---

## 🚀 Quick Start (Running Pre-built Executable)

If you already have a compiled binary (or want to run it on another Windows PC), **you do not need to install CMake, compilers, or the OpenSSL SDK**.

### What you need:
Copy the following 3 files from `build/Release/` into any folder:
- `ssl_imp.exe`
- `libcrypto-3-x64.dll`
- `libssl-3-x64.dll`

> [!NOTE]
> The destination Windows machine only requires the standard **Microsoft Visual C++ Redistributable (x64)** (`vcruntime140.dll`), which is already present on most Windows PCs.

### Run:
```cmd
ssl_imp.exe
```

By default, the program connects to `https://tls.browserleaks.com/json` and returns your detected **JA3** and **JA4** fingerprints.

#### Sample Output:
```json
{
  "user_agent": "Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/139.0.0.0 Safari/537.36",
  "ja4": "t13d1516h2_8daaf6152771_d8a2da3f94cd",
  "ja4_r": "t13d1516h2_002f,0035,009c,009d,1301,1302,1303,c013,c014,c02b,c02c,c02f,c030,cca8,cca9_0005,000a,000b,000d,0012,0017,001b,0023,002b,002d,0033,44cd,fe0d,ff01_0403,0804,0401,0503,0805,0501,0806,0601",
  "ja3_hash": "9eb3aaad7427354c5027f54c4c99624c"
}
```

---

## 🛠️ Building from Source

### Prerequisites
1. **CMake** (version 3.20 or newer)
2. **C99-compatible Compiler**:
   - Windows: Microsoft Visual Studio (MSVC) 2019+ or Clang/GCC
   - Linux: GCC or Clang
3. **OpenSSL (Shared Libraries) >= 3.6.0**:
   - Must be compiled with **Brotli** support (`TLSEXT_comp_cert_brotli`)
   - Requires commit [`03541d7`](https://github.com/openssl/openssl/commit/03541d7302d016aa28d436364d72d58baa3e2114) or newer.

### Build Steps

#### 1. Configure with CMake
Point `OPENSSL_ROOT_DIR` to your custom OpenSSL installation path:

**Windows (cmd / PowerShell):**
```cmd
cmake -S . -B build -DOPENSSL_ROOT_DIR="C:/path/to/openssl"
```
*(Use forward slashes `/` in paths even on Windows).*

**Linux / macOS:**
```bash
cmake -S . -B build -DOPENSSL_ROOT_DIR="/usr/local/openssl"
```

#### 2. Compile the Project
```cmd
cmake --build build --config Release
```

On Windows with MSVC, CMake automatically copies `libcrypto-3-x64.dll` and `libssl-3-x64.dll` into the `build/Release/` directory next to `ssl_imp.exe`.

---

## ⚙️ Customizing Target Host & Request

The request destination and payload are defined directly in [`main.c`](main.c).

### Changing Hostname and Path
Locate lines ~655-668 in [`main.c`](main.c):
```c
#define REQUEST_HOSTNAME "tls.browserleaks.com"
#define REQUEST_PATH     "/json"
```

To test against a protected endpoint (e.g. Cloudflare or a specific WAF site):
```c
#define REQUEST_HOSTNAME "example.com"
#define REQUEST_PATH     "/api/data"
```

### Changing HTTP Headers
You can customize the HTTP request headers inside the `httpreq` buffer in [`main.c`](main.c#L678-L701):
```c
const char httpreq[] =
    "GET " REQUEST_PATH " HTTP/1." REQUEST_VSN_MINOR "\r\n"
    "Host: " REQUEST_HOSTNAME "\r\n"
    "Accept: text/html,application/xhtml+xml,...\r\n"
    "Sec-Ch-Ua: \"Not;A=Brand\";v=\"99\", \"Google Chrome\";v=\"139\", \"Chromium\";v=\"139\"\r\n"
    "User-Agent: Mozilla/5.0 (Windows NT 10.0; Win64; x64) ...\r\n"
    "\r\n";
```
After modifying [`main.c`](main.c), re-run `cmake --build build --config Release`.

---

## 📂 Project Structure

```text
ssl_imp-master/
├── CMakeLists.txt        # CMake build configuration and DLL copy step
├── main.c                # TLS client, custom extensions (ALPS, ECH, GREASE), and HTTP client
├── README.md             # Project documentation
├── spec/                 # Relevant TLS extension specification documents
│   ├── TLS Application-Layer Protocol Settings Extension.html
│   └── TLS Encrypted Client Hello.html
└── build/                # Build output directory
    └── Release/
        ├── ssl_imp.exe
        ├── libcrypto-3-x64.dll
        └── libssl-3-x64.dll
```

---

## 🔍 How It Works Under the Hood

1. **Extension Callbacks**: OpenSSL custom extension hooks (`SSL_CTX_add_custom_ext`) are registered to inject GREASE (`0x0a0a`), ALPS (`TLSEXT_TYPE_application_settings`), and ECH (`TLSEXT_TYPE_ech`) frames into the ClientHello.
2. **Cipher & Curve Alignment**: OpenSSL context options are configured to restrict supported groups to Chrome's default (`*X25519MLKEM768:*X25519:secp256r1:secp384r1`) and match Chrome's exact cipher suite order.
3. **Certificate Compression**: Enables Brotli certificate decompression to replicate Chrome's `compress_certificate` extension.
4. **Platform Sockets**: Uses native Winsock on Windows and POSIX sockets on Linux/macOS for the TCP connection before upgrading through OpenSSL.

---

## 📄 License
Check source files for license details or project origin.
