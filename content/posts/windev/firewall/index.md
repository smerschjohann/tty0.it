---
title: "WSL: Firewall issues"
date: 2023-10-05
weight: 1
chapter: false
series: ["Windows development"]
series_order: 2
---

Different problems with the network can occur on the WSL2, most of them can be addressed in one or the other way.

Microsoft implemented the WSL2 environment using Hyper-V and by default a NAT network, where the host system (your Windows OS) and the WSL environment share a common network with each other.

This approach can lead to various problems on a company device as the company firewall or VPN solution might not corporate very well with it.

## Solutions

### network adapter order
Some VPN/firewall solutions simply require a specific network adapter order. If you setup WSL before the VPN software, that can help to solve issues.

### wsl-vpnkit

One solution is the use of [wsl-vpnkit](https://github.com/sakai135/wsl-vpnkit), it solves most problemms by establishing a Windows-Socket connection on Linux and Windows side and tunnel the traffic in the same way as [Docker for Windows](https://docs.docker.com/desktop/install/windows-install/) is using.

This solution comes with its own quirks and is not always needed. 

### Windows experimental feature

Since September 2023, it is possible to configure the WSL environment to use mirroring instead of NATting. In this mode, the WSL shares the same network as the Windows side. This also means that the WSL can reach ports on the windows side using _localhost_.

More information on this: https://devblogs.microsoft.com/commandline/windows-subsystem-for-linux-september-2023-update/

I did not test this feature until now, but this sounds promissing in situations where no other method works. But note, that there are known issues with VSCode at the moment.