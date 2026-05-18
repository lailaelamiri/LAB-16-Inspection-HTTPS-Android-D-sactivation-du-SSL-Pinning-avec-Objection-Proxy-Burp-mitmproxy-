# Android SSL Pinning Bypass — Lab Report

**Tools:** Frida 17.9.1 · Objection 1.12.4 · Burp Suite · Android Emulator (x86, API 30)  
**Platform:** Windows 11 · Python 3.14  

---

## Table of Contents

1. [Objective](#objective)
2. [Environment](#environment)
3. [Step 1 — Install Frida and Objection](#step-1--install-frida-and-objection)
4. [Step 2 — Deploy frida-server on the Emulator](#step-2--deploy-frida-server-on-the-emulator)
5. [Step 3 — Configure Proxy and Install CA](#step-3--configure-proxy-and-install-ca)
6. [Step 4 — Launch Objection and Disable SSL Pinning](#step-4--launch-objection-and-disable-ssl-pinning)
7. [Step 5 — Validate Traffic Interception](#step-5--validate-traffic-interception)
8. [How Objection Works](#how-objection-works)
9. [Deliverables Summary](#deliverables-summary)

---

## Objective

Demonstrate how to intercept HTTPS traffic from an Android application running on an emulator by:

- Deploying `frida-server` on the device
- Using Objection to disable SSL pinning at runtime
- Routing traffic through Burp Suite as a man-in-the-middle proxy

---

## Environment

| Component | Value |
|---|---|
| Host OS | Windows 11 |
| Python | 3.14 |
| Frida (PC) | 17.9.1 |
| Objection | 1.12.4 |
| Android emulator | API 30 (Android 11) |
| Emulator architecture | x86 |
| Proxy tool | Burp Suite |
| Proxy address | `10.0.2.2:8080` |
| Target app | `com.android.chrome` |

---

## Step 1 — Install Frida and Objection

Install pipx to isolate the tooling environment, then install Objection (which pulls in Frida as a dependency):

```bash
pip install --user pipx
python -m pipx ensurepath
```

Close and reopen the terminal after `ensurepath`, then:

```bash
pipx install objection
```

> **Windows note:** If `objection` or `frida` are not recognized after installation, add the Scripts directory to PATH manually:
> `C:\Users\<you>\AppData\Roaming\Python\Python3XX\Scripts`

### Verification

```
frida --version
-> 17.9.1

python -c "import frida; print(frida.__version__)"
-> 17.9.1
```

Objection does not expose a `--version` flag. Its version (1.12.4) is shown in the banner at launch.

> The frida version on the PC **must match exactly** the frida-server binary deployed on the device.

---

## Step 2 — Deploy frida-server on the Emulator

### 2.1 Identify the device and architecture

```bash
adb devices
```

```
List of devices attached
emulator-5554   device
```

```bash
adb shell getprop ro.product.cpu.abi
```

```
x86
```

### 2.2 Download the matching frida-server binary

Go to [https://github.com/frida/frida/releases/tag/17.9.1](https://github.com/frida/frida/releases/tag/17.9.1) and download:

```
frida-server-17.9.1-android-x86.xz
```

Extract with 7-Zip and rename the binary to `frida-server`.

### 2.3 Push and launch frida-server

```bash
adb push frida-server /data/local/tmp/
adb shell chmod 755 /data/local/tmp/frida-server
adb shell "/data/local/tmp/frida-server &"
```

The last command does not return — this is expected. frida-server is running in the background.

### 2.4 Verify device visibility

Open a new terminal and run:

```bash
frida-ps -Uai
```

```
  PID  Name          Identifier
-----  ------------  ------------------------------
12140  Diva          jakhar.aseem.diva
10534  Uncrackable1  owasp.mstg.uncrackable1
12633  Chrome        com.android.chrome
13713  YouTube       com.google.android.youtube
  ...
```

The emulator is visible and frida-server is communicating correctly.

---

## Step 3 — Configure Proxy and Install CA

### 3.1 Start Burp Suite

Launch Burp Suite on the PC. The default listener is `127.0.0.1:8080`.

### 3.2 Set the proxy on the emulator

Android emulators reach the host machine via the alias `10.0.2.2`:

```bash
adb shell settings put global http_proxy 10.0.2.2:8080
```

### 3.3 Install the Burp CA certificate

Download the Burp CA from `http://127.0.0.1:8080` (click **CA Certificate**), then push and install it:

```bash
adb push cacert.der /data/local/tmp/cert-der.crt
adb shell am start -n com.android.certinstaller/.CertInstallerMain \
  -a android.intent.action.VIEW \
  -t application/x-x509-ca-cert \
  -d file:///data/local/tmp/cert-der.crt
```

Give the certificate any name when prompted on the emulator (e.g. `Burp`).

### 3.4 Quick validation

Open Chrome on the emulator and visit any HTTPS site. Requests should appear in **Burp → Proxy → HTTP history**.

> **Why the CA is still needed even with Objection:** Objection neutralizes the app-level certificate verification. The CA is needed so the OS-level TLS stack trusts the proxy certificate and does not throw errors before the app even processes the response.

---

## Step 4 — Launch Objection and Disable SSL Pinning

### Strategy: spawn mode (recommended)

Spawn mode injects into the app at startup, catching pinning checks that run before the first screen renders:

```bash
objection -g com.android.chrome explore --startup-command "android sslpinning disable"
```

### Output

```
DeprecationWarning: The option 'gadget' is deprecated. Please use '-n' or '--name' instead
Running a startup command... android sslpinning disable
(agent) Custom TrustManager ready, overriding SSLContext.init()
(agent) Found com.android.org.conscrypt.TrustManagerImpl, overriding TrustManagerImpl.verifyChain()
(agent) Found com.android.org.conscrypt.TrustManagerImpl, overriding TrustManagerImpl.checkTrustedRecursive()
(agent) Registering job 414329. Name: android-sslpinning-disable

com.android.chrome on (Android: 11) [usb] #
```

Each hook line confirms a specific SSL verification function has been overridden in memory.

### Alternative: attach mode

If spawn crashes the app, start the app manually first, then attach:

```bash
objection -g com.android.chrome explore
# Once in the console:
android sslpinning disable
```

### Useful console commands

```bash
android sslpinning disable
android hooking search classes pin
help android sslpinning
```

---

## Step 5 — Validate Traffic Interception

Generate HTTPS traffic by browsing any site in Chrome on the emulator.

In **Burp → Proxy → HTTP history**, HTTPS requests appear in full — headers, cookies, request body — with the TLS column marked, confirming decryption is working.

Example captured traffic:

| Host | Method | Status | Notes |
|---|---|---|---|
| `www.google.com` | GET | 200 | Search request, fully decrypted |
| `clients2.google.com` | POST | 200 | Domain reliability upload |
| `beacons.gcp.gvt2.com` | POST | 307 | Analytics beacon |

---

## How Objection Works

SSL pinning is a technique where an app hardcodes the expected server certificate or public key directly in its code. Even with a trusted CA installed on the device, the app refuses to connect through a proxy because it does not recognize the proxy certificate as the one it expects.

Objection uses Frida to hook into the app process at runtime and replace the SSL verification functions with permissive versions that accept any certificate. No APK modification is required.

```
Without Objection:
  App checks cert -> does not match pinned value -> connection refused -> Burp sees nothing

With Objection:
  App checks cert -> hook intercepts -> always returns valid -> connection proceeds -> Burp sees everything
```

Two layers work together:

| Layer | Tool | What it does |
|---|---|---|
| OS trust store | Burp CA certificate | Convinces the OS to trust the proxy certificate |
| App-level pinning | Objection | Convinces the app to trust any certificate |

Both are needed for hardened targets. Apps without pinning (like Chrome, DIVA) only require the CA.

---

## Deliverables Summary

| Deliverable | Value |
|---|---|
| `frida --version` | 17.9.1 |
| `frida-ps -Uai` | Full app list captured |
| Objection version | 1.12.4 (shown in banner) |
| Launch strategy used | Spawn mode (`--startup-command`) |
| SSL pinning hooks fired | `SSLContext.init`, `TrustManagerImpl.verifyChain`, `TrustManagerImpl.checkTrustedRecursive` |
| Proxy traffic confirmed | HTTPS requests from Chrome visible in Burp HTTP history |
