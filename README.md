# PrintSpoofer Details

[![License](https://img.shields.io/badge/license-MIT-blue.svg)](LICENSE)
[![Platform](https://img.shields.io/badge/platform-Windows-lightgrey.svg)](https://www.microsoft.com/windows)
[![C++](https://img.shields.io/badge/C++-17-blue.svg)](https://isocpp.org/)

PrintSpoofer is a tool that performs local privilege escalation by exploiting a vulnerability in the Windows Print Spooler service (CVE-2019-1040 / CVE-2019-1019). This project implements the PrintSpoofer exploit in **Reflective DLL** format, making it suitable for use with penetration testing frameworks such as **Cobalt Strike**.

## Table of Contents

* [Overview](#overview)
* [How It Works](#how-it-works)
* [Technical Details](#technical-details)
* [Requirements](#requirements)
* [Build](#build)
* [Usage](#usage)
* [Project Structure](#project-structure)
* [Security Disclaimer](#security-disclaimer)
* [References](#references)

## Overview

PrintSpoofer exploits a vulnerability in the Windows Print Spooler service that allows a low-privileged user to obtain **SYSTEM**-level privileges. The exploit abuses the way the Print Spooler service handles named pipes.

### Features

* **Reflective DLL format** — can be loaded entirely in memory without touching disk
* Works with **SE_IMPERSONATE_NAME** privilege
* Communicates with the Print Spooler service using the **RPC** protocol
* Uses the **Named Pipe** mechanism to perform token impersonation

## How It Works

PrintSpoofer operates by performing the following steps:

### 1. Privilege Check

The exploit first checks whether the current token has the `SE_IMPERSONATE_NAME` privilege. This is required to impersonate another user’s token.

### 2. Named Pipe Creation

A named pipe is created using a random UUID. The pipe path is:

```
\\.\pipe\<UUID>\pipe\spoolss
```

### 3. Wait for Pipe Connection

An asynchronous connection is awaited on the created named pipe. An event is created, and the pipe is prepared to accept incoming connections.

### 4. Trigger Print Spooler Connection

In a separate thread, an RPC call is made to the Print Spooler service.
By invoking `RpcRemoteFindFirstPrinterChangeNotificationEx`, the service attempts to connect to the pipe we created.

### 5. Token Impersonation

When the Print Spooler service (running as SYSTEM) connects to the named pipe, the exploit uses `ImpersonateNamedPipeClient` to impersonate the service’s token.
This results in obtaining SYSTEM privileges.

### 6. Privileged Operation

With a SYSTEM token, high-privilege actions can now be executed.

## Technical Details

### Print Spooler Vulnerability

The Windows Print Spooler service processes printer notifications through named pipes.
Because the service does not properly validate pipe connections, an attacker can coerce it into connecting to a malicious pipe and then impersonate the SYSTEM token.

### RPC Communication

The project communicates with the Print Spooler service using the **MS-RPRN** (Print System Remote Protocol).
This protocol is defined through an IDL (Interface Definition Language) file and enables RPC calls.

### Reflective DLL Injection

A reflective DLL behaves differently from standard DLLs:

* It can be loaded directly into memory without touching disk
* Does not require the `LoadLibrary` API
* Manually parses and loads the PE (Portable Executable) format

## Requirements

### For Building

* **Visual Studio 2022** (C++ Desktop Development workload)
* **Windows 10/11**

### For Running

* **Windows 10/11**
* **Print Spooler** service running
* **SE_IMPERSONATE_NAME** privilege (available to most standard users by default)
* **Cobalt Strike** or a similar framework (optional)

## Build

### Building with Visual Studio

1. Open the **PrintSpoofer.sln** file in Visual Studio.

2. Select your **Solution Platform** and **Configuration**:

   * Platform: `x64` or `Win32` (depending on the target architecture)
   * Configuration: `Release` (production) or `Debug` (development)

3. Build the solution through `Build Solution` (Ctrl+Shift+B).

4. The compiled DLL will be located at:

   ```
   PrintSpoofer\x64\Release\PrintSpoofer.dll   (for x64)
   PrintSpoofer\Win32\Release\PrintSpoofer.dll (for x86)
   ```

### Build Options

The project uses the following preprocessor definitions:

* `REFLECTIVE_DLL_EXPORTS`: Enables Reflective DLL exports
* `REFLECTIVEDLLINJECTION_CUSTOM_DLLMAIN`: Uses a custom DllMain
* `REFLECTIVEDLLINJECTION_VIA_LOADREMOTELIBRARYR`: Enables LoadRemoteLibraryR (in Release mode)

## Usage

### Using with Cobalt Strike

PrintSpoofer can be used with Cobalt Strike's `elevate` command:

```
elevate PrintSpoofer LISTENER_NAME
```

Where:

* `PrintSpoofer`: The elevator module name
* `LISTENER_NAME`: The listener that will receive the elevated shell

### Manual Usage

The DLL can be manually loaded using Reflective DLL Injection techniques.

### Expected Output

A successful exploit execution will show:

```
[+] Found privilege: SeImpersonatePrivilege
[+] Named pipe listening...
[+] ImpersonateNamedPipeClient OK
[+] Exploit successfully, enjoy your shell
```

### File Descriptions

#### `dllmain.cpp`

* DLL entry point (`DllMain`)
* Controls exploit logic
* Handles reflective DLL–specific functionality

#### `PrintSpoofer.cpp`

* **CheckAndEnablePrivilege**: Checks and enables token privileges
* **GenerateRandomPipeName**: Generates pipe names using random UUIDs
* **CreateSpoolNamedPipe**: Creates named pipes and configures security
* **ConnectSpoolNamedPipe**: Prepares pipes for asynchronous connections
* **TriggerNamedPipeConnection**: Creates a thread to trigger Print Spooler connection
* **TriggerNamedPipeConnectionThread**: Performs RPC call
* **GetSystem**: Performs token impersonation

#### `ms-rprn.idl`

* IDL definition of Microsoft’s MS-RPRN protocol
* Processed by the MIDL compiler to generate RPC stubs

#### `ReflectiveLoader.cpp`

* Implements PE parsing and memory loading for Reflective DLL injection

## Disclaimer

This tool is intended for **educational purposes only** and for **authorized penetration testing**.

## References

* [https://www.itm4n.fr/printspoofer-abusing-impersonate-privileges/](https://www.itm4n.fr/printspoofer-abusing-impersonate-privileges/)
* [https://book.hacktricks.xyz/windows-hardening/windows-local-privilege-escalation](https://book.hacktricks.xyz/windows-hardening/windows-local-privilege-escalation)

## License

This project is licensed under the MIT License. For more information, see the [LICENSE file](LICENSE).