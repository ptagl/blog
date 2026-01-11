---
title: "Claiming WSL Disk Space"
description: "How to reclaim disk space used by WSL distributions on Windows"
publishDate: "11 Jan 2026"
tags: ["windows", "wsl"]
#pinned: true
#updatedDate: "12 Jan 2026"
---

## TLDR

Shut down your WSL distributions and use PowerShell (as administrator) to run `Optimize-VHD` on the VHDX file used by your WSL distribution:

```powershell
wsl --shutdown
Optimize-VHD -Path '<Path to your VHDX file>' -Mode Full
```

## Introduction

Working with WSL I sometimes see my disk usage grow significantly until the day I'm almost out of space and need to reclaim it somehow. In my case, the reason is that WSL uses a virtual hard disk (VHDX) file to store the file system of each distribution, and over time these files grow in size as data is added, but they don't shrink when data is removed.

:::note
This is true unless your WSL distribution is configured to use the "sparse mode", which should automatically resize the VHDX file as needed. However, many people reported issues with this mode (for instance [here](https://github.com/microsoft/WSL/issues/10703), [here](https://github.com/microsoft/WSL/issues/12103), and [here](https://github.com/microsoft/WSL/issues/4699#issuecomment-565700099)), so it's not considered a reliable solution as of today.
:::

It’s easy to check whether you have this issue too. Since I always forget the path of my WSL VHDX file, I typically use PowerShell to find it:

```powershell
(Get-ChildItem -Path $env:USERPROFILE\AppData\Local\Packages\ -Filter "*.vhdx" -Recurse -ErrorAction SilentlyContinue).FullName
```

Copy the path and store it in a variable for convenience:

```powershell
$VHDXPath = '<Path to your VHDX file>'
```

Then, check the size of the VHDX file:

```powershell
"{0:N2} GB" -f ((Get-Item $VHDXPath).Length / 1GB)
```

Now, it's time to compare this size with the actual used space inside the WSL distribution. To do this, start your WSL distribution and run:

```bash
df -h /
```

If the size of the VHDX file is significantly larger than the used space reported by `df`, then it's definitely time to reclaim some bytes!

## Optimize-VHD to the rescue

Luckily, there is a PowerShell cmdlet called [Optimize-VHD](https://learn.microsoft.com/en-us/powershell/module/hyper-v/optimize-vhd?view=windowsserver2025-ps) that can be used to compact VHDX files. This command reclaims unused space in the VHDX file, effectively reducing its size on disk.

:::note
`Optimize-VHD` is part of the Hyper-V module. Availability on your system depends on your Windows edition and features installed. If you don't have it, you might need to enable the Hyper-V feature on your Windows system via "Turn Windows features on or off" in the Control Panel.
:::

First of all, make sure the VHDX file is not mounted (the easiest way is to stop WSL), then run the `Optimize-VHD` command as administrator on the VHDX file you found earlier:

```powershell
wsl --shutdown
Optimize-VHD -Path $VHDXPath -Mode Full
```

The process may take some time depending on the size of the VHDX file and the amount of unused space to reclaim.

After the command completes, you can check the size of the VHDX file again with the same commands used above to see how much space you've reclaimed.

And that's all, see you next time your disk is full!
