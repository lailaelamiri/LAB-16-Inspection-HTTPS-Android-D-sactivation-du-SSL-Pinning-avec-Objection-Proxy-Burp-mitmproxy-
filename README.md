# Android SSL Pinning Bypass — Lab Report

**Tools:** Frida 17.9.1 · Objection 1.12.4 · Burp Suite · Android Emulator (x86, API 30)  
**Platform:** Windows 11 · Python 3.14  

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
<img width="1709" height="314" alt="Screenshot 2026-05-18 094643" src="https://github.com/user-attachments/assets/9eb86107-4bbb-492b-b1ca-90c0d02ac84d" />

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
<img width="1229" height="108" alt="image" src="https://github.com/user-attachments/assets/a7d2d881-cf1c-480f-a58d-d12f22ebb979" />


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
<img width="920" height="178" alt="image" src="https://github.com/user-attachments/assets/73b974fd-b1ed-4e95-a6f7-e7fa05d36576" />


### 2.2 Push and launch frida-server

```bash
adb push frida-server /data/local/tmp/
adb shell chmod 755 /data/local/tmp/frida-server
adb shell "/data/local/tmp/frida-server &"
```
<img width="1381" height="32" alt="image" src="https://github.com/user-attachments/assets/454c1abb-1aaa-490b-a5de-516b3e176c6a" />

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
<img width="1308" height="672" alt="image" src="https://github.com/user-attachments/assets/b57efc71-adf6-4a9a-a24a-2d4b536d0304" />

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
<img width="619" height="584" alt="Screenshot 2026-05-17 110949" src="https://github.com/user-attachments/assets/2a946f37-124e-45a3-8b9c-64a35b9c385d" />

```bash
adb push cacert.der /data/local/tmp/cert-der.crt
adb shell am start -n com.android.certinstaller/.CertInstallerMain \
  -a android.intent.action.VIEW \
  -t application/x-x509-ca-cert \
  -d file:///data/local/tmp/cert-der.crt
```
<img width="198" height="137" alt="Screenshot 2026-05-17 112037" src="https://github.com/user-attachments/assets/c77314db-58b5-4d32-bcbc-e5d1aa0f940e" />

Give the certificate any name when prompted on the emulator (e.g. `Burp`).


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


<img width="1658" height="612" alt="image" src="https://github.com/user-attachments/assets/ebf5539e-8fe9-447b-9c18-1526a89135a9" />

---

## Step 5 — Validate Traffic Interception

Generate HTTPS traffic by browsing any site in Chrome on the emulator.

In **Burp → Proxy → HTTP history**, HTTPS requests appear in full — headers, cookies, request body — with the TLS column marked, confirming decryption is working.



---
<img width="1919" height="694" alt="image" src="https://github.com/user-attachments/assets/4a1453eb-53c7-4d1e-be25-a7460dceb01c" />


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
