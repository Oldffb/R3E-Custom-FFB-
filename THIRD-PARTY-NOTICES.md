# Third-Party Notices — R3E Custom FFB

The R3E Custom FFB release package is **self-contained**: the .NET runtime and all
required libraries are bundled inside `R3E FFB.exe`. Those components remain under
their own licences, reproduced below.

This file satisfies the attribution requirement of those licences. It is included
in the release `.zip` and in the public repository.

---

## Bundled inside the executable

| Component | Version | Licence |
|---|---|---|
| .NET Runtime, ASP.NET Core, Windows Desktop | 8.0 | MIT |
| Microsoft.Extensions.FileProviders.Embedded | 8.0.11 | MIT |
| Microsoft.AspNetCore.SignalR | (part of ASP.NET Core 8.0) | MIT |
| SharpDX.DirectInput | 4.2.0 | MIT |
| YamlDotNet | 15.1.2 | MIT |
| Microsoft.Web.WebView2 (SDK) | 1.0.2277.86 | Microsoft Software Licence Terms |

## Required at runtime, installed separately

| Component | Notes |
|---|---|
| Microsoft Edge WebView2 Runtime | Not bundled. Ships with Windows 11 and with updated Windows 10. Distributed by Microsoft under its own terms. |

## Not bundled in this package

`Dirkster.AvalonDock` and `Dirkster.AvalonDock.Themes.VS2013` (MS-PL) are used by
the separate Telemetry Viewer application and are **not** part of this release.

---

## MIT License

Applies to the .NET Runtime and ASP.NET Core (Copyright (c) .NET Foundation and
Contributors), SharpDX (Copyright (c) 2010-2014 SharpDX — Alexandre Mutel), and
YamlDotNet (Copyright (c) Antoine Aubry and contributors).

```
Permission is hereby granted, free of charge, to any person obtaining a copy
of this software and associated documentation files (the "Software"), to deal
in the Software without restriction, including without limitation the rights
to use, copy, modify, merge, publish, distribute, sublicense, and/or sell
copies of the Software, and to permit persons to whom the Software is
furnished to do so, subject to the following conditions:

The above copyright notice and this permission notice shall be included in all
copies or substantial portions of the Software.

THE SOFTWARE IS PROVIDED "AS IS", WITHOUT WARRANTY OF ANY KIND, EXPRESS OR
IMPLIED, INCLUDING BUT NOT LIMITED TO THE WARRANTIES OF MERCHANTABILITY,
FITNESS FOR A PARTICULAR PURPOSE AND NONINFRINGEMENT. IN NO EVENT SHALL THE
AUTHORS OR COPYRIGHT HOLDERS BE LIABLE FOR ANY CLAIM, DAMAGES OR OTHER
LIABILITY, WHETHER IN AN ACTION OF CONTRACT, TORT OR OTHERWISE, ARISING FROM,
OUT OF OR IN CONNECTION WITH THE SOFTWARE OR THE USE OR OTHER DEALINGS IN THE
SOFTWARE.
```

## Microsoft.Web.WebView2

Copyright (c) Microsoft Corporation. Licensed under the Microsoft Software
Licence Terms for the WebView2 SDK, which permit redistribution of the SDK
binaries as part of an application. Full terms:
https://aka.ms/webview2-sdk-license

## Microsoft Public License (MS-PL)

Applies to Dirkster.AvalonDock — **not distributed in this package**, listed for
completeness. Full terms: https://opensource.org/licenses/MS-PL

---

## Telemetry data

R3E Custom FFB reads telemetry from the shared-memory block that RaceRoom Racing
Experience publishes for third-party applications. No game files are modified,
patched or redistributed.
