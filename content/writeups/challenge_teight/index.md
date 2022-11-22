---
title: Flare-On 2022 - T8
description: Write-up for T8 Flare-on 2022 challenge
toc: true
authors: Alpharivs
tags:
  - reversing
  - static
  - dynamic
  - challenge
  - ctf
  - flare-on
categories: write-ups
date: '2022-11-22'
featuredImage: /images/writeups/flare.png
draft: true
---

## Challenge Description
```text
FLARE FACT #823: Studies show that C++ Reversers have fewer friends on average than normal people do. That’s why you’re here, reversing this, instead of with them, because they don’t exist.

We’ve found an unknown executable on one of our hosts. The file has been there for a while, but our networking logs only show suspicious traffic on one day. Can you tell us what happened?
```
## Challenge Overview

We are given a Windows PE32 executable and a .pcap file.

```bash
+------------------------+------------------------------------------------------------------------------------+
| md5                    | ece22f965b73135469a29a15ae978fb8                                                   |
| sha1                   | 2fd7acefe6c1d258a5666f2b33e123af2eecab50                                           |
| sha256                 | 892a46d456d3ac2d4b406728b0b15dc40450c03bc40af03fd3c510f52eb6f3f7                   |
| os                     | windows                                                                            |
| format                 | pe                                                                                 |
| arch                   | i386                                                                               |
| path                   | t8.exe                                                                             |
+------------------------+------------------------------------------------------------------------------------+
```

```bash
traffic.pcapng: pcapng capture file - version 1.0
```
## Basic Analysis.

### Functionality
We can use capa to get a quick overview of the executable's capabilities.
```bash
+------------------------------------------------------+------------------------------------------------------+
| CAPABILITY                                           | NAMESPACE                                            |
|------------------------------------------------------+------------------------------------------------------|
| get geographical location                            | collection                                           |
| initialize WinHTTP library                           | communication/http                                   |
| prepare HTTP request                                 | communication/http/client                            |
| receive HTTP response                                | communication/http/client                            |
| encode data using Base64 (2 matches)                 | data-manipulation/encoding/base64                    |
| reference Base64 string                              | data-manipulation/encoding/base64                    |
| encode data using XOR                                | data-manipulation/encoding/xor                       |
| encrypt data using RC4 KSA                           | data-manipulation/encryption/rc4                     |
| encrypt data using RC4 PRGA                          | data-manipulation/encryption/rc4                     |
| hash data with MD5                                   | data-manipulation/hashing/md5                        |
| contain a resource (.rsrc) section                   | executable/pe/section/rsrc                           |
| print debug messages (2 matches)                     | host-interaction/log/debug/write-event               |
| allocate RWX memory                                  | host-interaction/process/inject                      |
+------------------------------------------------------+------------------------------------------------------+
```
That gives us a quick overview of the executable's functionality which includes communication, encoding and encryption.

### Imports

Looking at the imports we see potential anti-debugging techniques among other interesting functions that give us a bigger picture of the functionality of the exe.
```text
KERNEL32.dll - Sleep
KERNEL32.dll - OutputDebugStringW - Can be used as an anti-debugging technique.
KERNEL32.dll - CreateFileW
KERNEL32.dll - GetModuleHandleW - Can be used to locate and modify code in a loaded module.
KERNEL32.dll - GetProcAddress - Retrieves the address of a function in a DLL loaded into memory.
KERNEL32.dll - IsDebuggerPresent - Can be used as an anti-debugging technique but often added by the compiler.
KERNEL32.dll - GetStartupInfoW - Retrieves a structure containing details about how the current process was configured to run.
KERNEL32.dll - QueryPerformanceCounter - Can be used as an anti-debugging technique.
KERNEL32.dll - LoadLibraryExW - Dynamically loads a DLL.
KERNEL32.dll - WriteFile
KERNEL32.dll - FindFirstFileExW - Searches through a directory.
KERNEL32.dll - FindNextFileW - - Searches through a directory.
```
And we can confirm what capa already showed us, that the malware imports libraries needed for communication vie http.
```text
WINHTTP.dll - WinHttpReceiveResponse
WINHTTP.dll - WinHttpOpen
WINHTTP.dll - WinHttpReadData
WINHTTP.dll - WinHttpOpenRequest
WINHTTP.dll - WinHttpCloseHandle
WINHTTP.dll - WinHttpSendRequest
WINHTTP.dll - WinHttpConnect

urlmon.dll  - ObtainUserAgentString - Retrieves the User-Agent HTTP request header string that is currently being used.
```

### Strings

Using Floss we find a few interesting strings.
```bash
# Standard Base64 string
ABCDEFGHIJKLMNOPQRSTUVWXYZabcdefghijklmnopqrstuvwxyz0123456789+/
ABCDEFGHIJKLMNOPQRSTUVWXYZabcdefghijklmnopqrstuvwxyz0123456789-_
Input is not valid base64-encoded data.

# Lot's of mangled function names typical of C++ code.
.?AVCClientSock@@

# A domain name.
flare-on.com
```

### Pcap File.

The pcap file contains two HTTP Post Requests with their respective responses which contain what seems to be a b64 encoded body.

![pcap](images/pcap.png)

Decoding the b64 strings doesn't yield any useful data.

### Behavior


## Advanced Static Analysis