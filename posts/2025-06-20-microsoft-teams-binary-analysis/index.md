# How Microsoft Teams Desktop Application works Under the Hood: A Binary Analysis

*By Tomiwa, Damilola & AbdulAzeez | June 17, 2025*

Microsoft Teams has become indispensable for modern collaboration, but few understand the technical foundations that power it. In this analysis, we examine the portable executable of Microsoft Teams (`ms-teams.exe`) through static and dynamic methods to uncover how Microsoft's flagship communication tool operates at the binary level.

<figure>
  <img src="https://cdn-images-1.medium.com/max/800/1*G5JFdKj27WNIToC8XotlyQ.png" alt="Teams client interface showing meeting link generation and account management functions">
  <figcaption>Fig 1.0 — Teams client interface showing meeting link generation and account management functions</figcaption>
</figure>

Based on observations, Microsoft Teams is primarily built using the following technologies:

- **TypeScript/JavaScript** for the frontend (using Electron and React)
- **C++ and C#** for native components and backend integrations
- **Node.js** for some server-side logic in Electron apps

Using industry-standard tools including CFF Explorer, Procmon, and API Monitor, we:

- Verified the binary's authenticity through its digital signature
- Mapped its process architecture and dependencies
- Analyzed network behavior during calls

Our findings confirm Teams employs expected patterns for an Electron application, with particular attention to security and reliability in its design.

<figure>
  <img src="https://cdn-images-1.medium.com/max/800/1*MgoEys5wGuz-yJBwpVui1g.png" alt="Teams.exe digital signature and file hashes">
  <figcaption>Fig 1.1 — Teams.exe's digital signature and file hashes verify its authenticity as a Microsoft-signed binary</figcaption>
</figure>

---

## Static Analysis: Microsoft Teams Desktop Binary

Static analysis is the process of examining a program's structure and behavior without executing it. For this analysis, we focused on the Microsoft Teams binary (`Teams.exe`) to gain a deeper understanding of how the application is built and what system components it interacts with. By inspecting the file using tools like CFF Explorer, we explored key elements such as file metadata, PE headers, imported libraries, and embedded resources. This foundational step helps assess the application's authenticity, dependencies, and potential areas of vulnerability before observing it in a live environment.

### 1. PE Structure: A Memory Blueprint

**Tool: CFF Explorer**

When we loaded `Teams.exe` into CFF Explorer, its PE headers revealed a textbook-perfect Windows GUI executable:

- **AddressOfEntryPoint (0x0001A790):** The exact memory address where execution begins, like the ignition key for Teams' engine.
- **ImageBase (0x0000000140000000):** When a PE File is executed, the program is loaded into the virtual memory. However, how does the PE File know what address to inject this code? This is handled by ImageBase — the address where the program is designed to be loaded into virtual memory.
- **Subsystem (Windows GUI):** This field lets us know the type of PE executable we are dealing with — Native, Windows Console, Windows GUI, etc. `ms-teams.exe` is a Windows executable with a graphical interface, hence the selected field here was **Windows GUI**.

**Sections Breakdown:**

- **`.text`:** The executable code (marked `CODE, EXECUTE, READ`).
- **`.rdata`:** Read-only data (import tables, constant strings).
- **`.data`:** Global variables (read/write).

<figure>
  <img src="https://cdn-images-1.medium.com/max/800/1*RY3tOj-3cyM5jCvOo2veaw.png" alt="Teams.exe PE headers showing memory layout">
  <figcaption>Fig 2.0 — Teams.exe's PE headers reveal its memory layout and execution logic</figcaption>
</figure>

<figure>
  <img src="https://cdn-images-1.medium.com/max/800/1*vWRR6bSP24mMAJISvNYkBQ.png" alt="Section headers of Teams.exe">
  <figcaption>Fig 2.1 — Section headers of Teams.exe, listing executable segments like .text, .rdata and .data</figcaption>
</figure>

### 2. File Metadata

The File Metadata section delivered critical authenticity checks:

- **Timestamps:** Compile/modification dates aligned with Microsoft's build pipelines.
- **Hashes:**

```
MD5:  A5907BD8A27705C23C876772BF807E4C
SHA-1: 29F9BE21FBA04A623E625408DC61AE23E38299DB
```

- **Digital Signature:** A valid certificate chain rooted to the Microsoft Code Signing PCA, with no evidence of tampering.

### 3. DLL Imports

The Import Directory exposed Teams' reliance on Windows APIs:

**KERNEL32.dll** — Core OS functions:
- `CreateProcessW`: Launches subprocesses (e.g., `msedgewebview2.exe`).
- `OpenProcess`: Debugging/monitoring other processes.
- `SuspendThread`: Thread manipulation, critical for responsiveness.

**USER32.dll** — GUI Operations:
- `SetClipboardData`: Manipulates the clipboard, allowing you to copy or paste data programmatically.
- `GetWindowRect`: Manages window positioning during screen sharing.

**WININET.dll** — Networking:
- `InternetConnectW`: Initiates HTTPS connections to `login.microsoftonline.com`.
- `HttpSendRequestW`: Sends API calls to Microsoft's servers.

**SQLITE3.dll** — Local data storage (chat logs, settings, user profiles):
- `Sqlite3_exec`: Allows an application to run multiple SQL statements.
- `Sqlite3_finalize`: Deletes a prepared statement object.

**ADVAPI32.dll** — Registry and encryption operations.

<figure>
  <img src="https://cdn-images-1.medium.com/max/800/1*W5VclObe90WH2SqRd4pOfw.png" alt="Import Directory of Teams.exe">
  <figcaption>Fig 3.0 — Import Directory of Teams.exe, highlighting key DLL dependencies such as KERNEL32.dll and USER32.dll</figcaption>
</figure>

<figure>
  <img src="https://cdn-images-1.medium.com/max/800/1*jb_uXdQB7jzr7lAVKJj64g.png" alt="Imported functions from WININET.dll and SQLITE.dll">
  <figcaption>Fig 3.1 — Imported functions from WININET.dll and SQLITE.dll, indicating internet and local database capabilities</figcaption>
</figure>

### 4. Embedded Resources

The `.rsrc` section contained:

**Version Information:**
```
CompanyName: Microsoft Corporation
FileDescription: Microsoft Teams
LegalCopyright: © Microsoft Corporation. All rights reserved.
```

- **Icons:** The familiar Teams logo in 16x16 to 256x16 resolutions.

> *Note: Attackers often tamper with resources for icon hijacking…no signs of this here.*

<figure>
  <img src="https://cdn-images-1.medium.com/max/800/1*7Dl2_k5IXHi-LOEugKJdig.png" alt="Legitimate icons extracted from .rsrc section of Teams.exe">
  <figcaption>Fig 4.0 — Legitimate icons extracted from .rsrc section of Teams.exe</figcaption>
</figure>

### 5. Strings Analysis

Running Unicode strings extraction on `ms-teams.exe` revealed valuable indicators about the application's internal operations. We found references to imported libraries, executables and domains (one of which is used for authentication).

**Authentication Endpoints:**
```
https://login.microsoft.com
https://login.windows.local
```

User-facing features were also present, along with digital signature details confirming binary legitimacy. Notably, several `auth.*` strings were discovered, indicating internal functions for login, token handling, encryption, and account management.

**User-facing status strings:**
```
"Available"
"InAMeeting"
"Offline"
```

<figure>
  <img src="https://cdn-images-1.medium.com/max/800/1*DLjy7fc0L1qlr367GqMpjg.png" alt="Embedded strings exposing Teams backend interactions">
  <figcaption>Fig 5.0 — Embedded strings expose Teams' backend interactions and state tracking</figcaption>
</figure>

---

## Dynamic Analysis: Microsoft Teams Desktop Binary

Dynamic analysis involves observing how an application behaves while it is actively running. In this phase, we executed `Teams.exe` in a controlled environment to monitor its real-time interactions with the operating system. Using tools like Process Monitor, API Monitor, and network capture utilities, we tracked Teams' file operations, registry access, process creation, network communications, and API calls.

### Analyzing Microsoft Teams' Internal Operations via ProcMon

To understand how Microsoft Teams behaves during execution, Process Monitor (ProcMon) was used to capture its real-time system activity. ProcMon tracks file access, registry usage, and process operations, helping to reveal how the application loads components, manages user data, and interacts with the Windows environment.

<figure>
  <img src="https://cdn-images-1.medium.com/max/800/1*VH7VBqyGy12WiR6w1PbrrQ.png" alt="ProcMon showing Teams reading system libraries and user settings">
  <figcaption>Fig 6.0 — ProcMon showing Teams reading system libraries and user settings</figcaption>
</figure>

### Loading Essential Components via the File System

Microsoft Teams exhibits extensive file system activity upon launch, primarily focused on loading essential DLLs. These DLLs are loaded from both its own installation directory and Windows system directories like `C:\Windows\System32`:

- **Graphics and UI rendering:** `uxtheme.dll`, `d3d11.dll`, `dxgi.dll`, `d2d1.dll`
- **System-level operations:** `ntdll.dll`, `kernel32.dll`
- **Networking:** `wininet.dll`, `mswsock.dll`, `libcurl.dll`
- **Secure communication and authentication:** `bcrypt.dll`, `secur32.dll`, `sspicli.dll`

<figure>
  <img src="https://cdn-images-1.medium.com/max/800/1*dU3B9S6_ODuUwmxRtfO52A.png" alt="Teams loading DLLs and accessing registry keys during startup">
  <figcaption>Fig 6.1 — Teams loading DLLs and accessing registry keys during startup</figcaption>
</figure>

Teams also makes use of web technologies. It accesses older web rendering components like `mshtml.dll` and `jscript9.dll`, alongside modern webview technologies, to handle embedded web content. For structured local data management, libraries like `sqlite3.dll` and `boost_json-vc143-mt-x64-1_86.dll` are loaded, supporting Teams' local caching and configuration storage mechanisms.

<figure>
  <img src="https://cdn-images-1.medium.com/max/800/1*6Gh7Uj-6nyOEIyyxleu3qA.png" alt="Microsoft Teams loads DLLs and reads system files during initialization">
  <figcaption>Fig 6.2 — Microsoft Teams loads DLLs and reads system files during initialization</figcaption>
</figure>

### Local Data Management

Microsoft Teams actively writes data to the user-specific AppData directory during its runtime. This includes authentication credentials, configuration files, and activity logs. For example, login tokens are written to paths such as:

```
C:\Users\Yuki\AppData\Local\Packages\MSTeams_8wekyb3d8bbwe\LocalCache\Local\Microsoft\OneAuth\accounts\d991faadcdffb2b3
```

<figure>
  <img src="https://cdn-images-1.medium.com/max/800/1*gQCsH-Mw-eZg1D3GNlXZeA.png" alt="Teams accesses and modifies OneAuth account data in the local cache">
  <figcaption>Fig 7.0 — Teams accesses and modifies OneAuth account data in the local cache</figcaption>
</figure>

> Application settings like UI preferences and account-specific configurations are stored in JSON files under: `LocalCache\Microsoft\MSTeams\app_settings.json`.
> File operations such as `CreateFile` and `WriteFile` confirm the app's dynamic interaction with the file system.

<figure>
  <img src="https://cdn-images-1.medium.com/max/800/1*1ta6O2vhcxRr_8huiJRtvw.png" alt="Teams frequently reads and writes to app_settings.json during runtime">
  <figcaption>Fig 7.1 — Teams frequently reads and writes to 'app_settings.json' during runtime</figcaption>
</figure>

### Registry Activity

Teams frequently interacts with the Windows Registry to retrieve system and user-specific settings via `RegOpenKey` and `RegQueryValue`. The application accesses:

- **Personalization settings** (e.g., `AppsUseLightTheme`) via `HKCU\SOFTWARE\Microsoft\Windows\CurrentVersion\Themes\Personalize`
- **UI language preferences** through `HKCU\Control Panel\Desktop`

<figure>
  <img src="https://cdn-images-1.medium.com/max/800/1*P8dUt-VDsOpwwJl33WGkRQ.png" alt="Teams queries Windows theme and desktop-related registry keys">
  <figcaption>Fig 8.0 — Teams queries Windows theme and desktop-related registry keys, checking UI preferences</figcaption>
</figure>

To gather environment configuration, it queries:
```
HKLM\System\CurrentControlSet\Control\Session Manager\Environment
```

<figure>
  <img src="https://cdn-images-1.medium.com/max/800/1*V4QcV2pzB5SvHXeQj9zUhw.png" alt="Teams reads system environment variables from the registry">
  <figcaption>Fig 8.1 — Teams reads system environment variables from the registry during initialization</figcaption>
</figure>

Security posture is also verified — Teams checks if FIPS mode is enabled by querying:
```
HKLM\System\CurrentControlSet\Control\Lsa\FipsAlgorithmPolicy
```

<figure>
  <img src="https://cdn-images-1.medium.com/max/800/1*rfplQxMwDUTe7BLmHSC_VQ.png" alt="Teams queries LSA cryptographic policy settings">
  <figcaption>Fig 8.2 — Teams queries Local Security Authority (LSA) cryptographic policy settings, indicating security or compatibility checks</figcaption>
</figure>

For network-related configuration, it reads from `HKLM\System\CurrentControlSet\Services\WinSock2\Parameters`, including subkeys like `Protocol_Catalog9` and `AppId_Catalog`.

<figure>
  <img src="https://cdn-images-1.medium.com/max/800/1*scL4MtSb-4ak5DOp9VoI9g.png" alt="Teams queries Winsock parameters and protocol catalog entries">
  <figcaption>Fig 8.3 — Teams queries Winsock parameters and protocol catalog entries, showing network stack initialization</figcaption>
</figure>

---

## Authentication via the ms-teams.exe Executable

To effectively carry out a network communications analysis of the Microsoft Teams Portable Executable, the login process was simulated and captured with various tools — specifically Wireshark.

<figure>
  <img src="https://cdn-images-1.medium.com/max/800/1*7ym92fmwuLoGVa38jTpj5Q.png" alt="OAuth 2.0 authentication flow initiation interface">
  <figcaption>Fig 9.0 — OAuth 2.0 authentication flow initiation interface</figcaption>
</figure>

### Authentication & Identity Services

During login, Teams communicates with domains such as `onedscolprdwus20.westus.cloudapp.azure.com` (20.189.173.25) and `euripv4.consent.config.office.akadns.net` (20.74.39.211). These are closely tied to Azure Active Directory (AAD) and Office 365 consent/configuration services, handling sign-in workflows, consent screens, and token issuance via OAuth and OpenID Connect.

### Core Teams Services

Essential domains include `teams.live.com`, `presence.teams.live.com`, `res.cdn.office.net`, and `admin.microsoft.com`. The Teams endpoints manage chat sessions and update presence indicators. The CDN domain delivers static assets like JavaScript files and CSS.

### Real-Time Communication (RTC / Calling / Meetings)

Real-time communications rely heavily on Trouter services — domains like `go.trouter.skype.com` and partition-based trouter endpoints (e.g., IPs 74.248.74.117 and 74.248.73.245). These manage session signaling, call setup, and NAT traversal using TURN and STUN over both UDP and TCP.

### Content Delivery Networks (CDNs)

Domains like `a726.dscd.akamai.net` (2.18.188.79) and `s-0005.s-msedge.net` (52.113.194.132) deliver static content — UI components, JavaScript files, media elements, and application updates. Microsoft's long-standing relationship with Akamai (since 1999) means many resources are served through Akamai's infrastructure. [More info here.](https://news.microsoft.com/1999/09/27/microsoft-and-akamai-form-strategic-relationship-to-enhance-internet-content-delivery/)

### Telemetry / Analytics / Consent Logging

Domains such as `teams.events.data.microsoft.com` and `tfl.nel.measure.office.net` are contacted for logging application usage events, tracking network errors, and collecting performance data.

---

## Understanding the Process of Initiating a Teams Call

Here we tried to understand how Microsoft Teams initiates a successful call, focusing on key interactions and their significance.

<figure>
  <img src="https://cdn-images-1.medium.com/max/800/1*X1dv0S82fPDCDXtokJvfQg.png" alt="Meeting connection handshake in progress">
  <figcaption>Fig 10.0 — Meeting connection handshake in progress (UI feedback state)</figcaption>
</figure>

The computer (`192.168.216.135`) served as the central point of communication, interacting with a primary Teams server (`48.208.209.177`), other Microsoft relay servers (`52.123.151.79`, `52.114.239.228`, `52.114.239.218`), and the local DNS server (`192.168.216.2`).

Teams leverages several network protocols to ensure a smooth and efficient call experience:

**STUN & TURN**

These protocols enable the computer to establish connections with other participants behind a NAT. STUN discovers the public internet address (e.g., `102.68.XX.XX:21265`), while TURN relays call data if a direct peer-to-peer connection is not feasible. Our analysis showed the computer constantly exchanging STUN messages with `48.208.209.177`.

<figure>
  <img src="https://cdn-images-1.medium.com/max/800/1*Cg2-4qwELPO6mXad6aZkA.png" alt="STUN/TURN negotiation with Microsoft servers">
  <figcaption>Fig 10.1 — STUN/TURN negotiation with Microsoft servers (48.208.209.177)</figcaption>
</figure>

**RTCP (RTP Control Protocol)**

RTCP continuously monitors audio/video quality during a call. We observed frequent Receiver Report and Sender Report message exchanges between the computer and Teams server (`48.208.209.177`). Payload-specific Feedback messages allowed Teams to adapt quality (e.g., reducing video resolution) if network performance deteriorated.

<figure>
  <img src="https://cdn-images-1.medium.com/max/800/1*qUVFExsycMhNSDvW_cFusg.png" alt="RTCP quality control packets monitoring call performance">
  <figcaption>Fig 10.2 — RTCP quality control packets monitoring call performance</figcaption>
</figure>

**UDP (User Datagram Protocol)**

UDP is the preferred protocol for real-time audio/video transmission due to its speed. Microsoft Teams specifically utilizes UDP ports `3478-3481` for media transport related to STUN, TURN, and ICE. Significant UDP traffic was observed between the computer and `52.123.151.79`.

<figure>
  <img src="https://cdn-images-1.medium.com/max/800/1*pUuOIVLrpiWRx6HrMkfN9g.png" alt="UDP media streams relayed via TURN server">
  <figcaption>Fig 10.3 — UDP media streams relayed via TURN server (52.123.151.79:3478)</figcaption>
</figure>

**DNS**

During the call, the client connected to `teams.events.data.microsoft.com`, confirming that Teams was transmitting usage data even while the call was active.

<figure>
  <img src="https://cdn-images-1.medium.com/max/800/1*Dmfs9lwKJ4rvRE5Gurt8HA.png" alt="Teams sends a DNS request to teams.events.data.microsoft.com">
  <figcaption>Fig 10.4 — Teams sends a DNS request to teams.events.data.microsoft.com, indicating telemetry or event tracking communication</figcaption>
</figure>

---

*Progressively, in the next attempt, we will decompile `ms-teams.exe` to reverse-engineer its components and gain a deeper understanding of its inner workings.*
