---
title: "WSL: Firewall issues"
date: 2023-10-05
weight: 1
chapter: false
series: ["Windows development"]
series_order: 2
---

Different problems with the network can occur on the WSL2, most of them can be addressed in one or the other way.

Microsoft implemented the WSL2 environment using Hyper-V and a NAT network, where the host system (your Windows OS) and the WSL environment share a common network with each other.

This approach can lead to various problems on a corporate device as the firewall or VPN solution might not play well together with the NAT network approach.

## Solutions

### network adapter order
Some VPN/firewall solutions simply require a specific network adapter order. If you setup WSL before the VPN software, that can help to solve issues.

### wsl-vpnkit

One solution is the use of [wsl-vpnkit](https://github.com/sakai135/wsl-vpnkit), it solves most problems by establishing a Windows-Socket connection between Linux and Windows and tunnel the traffic via that newly created socket. It is mostly invisible to any Firewall tool and uses the tools implemented by [Docker for Windows](https://docs.docker.com/desktop/install/windows-install/).

This solution comes with its own quirks and is most likely not required anymore. 

### Windows experimental feature

Since September 2023, it is possible to configure the WSL environment to use mirroring instead of NAT. In this mode, the WSL shares the same network as the Windows side. This also means that the WSL can reach ports on the windows side using _localhost_.

More information: https://devblogs.microsoft.com/commandline/windows-subsystem-for-linux-september-2023-update/

I did not test this feature, but this sounds promising in situations where no other method works. But note, that there are known issues with VSCode at the moment, which will be most likely resolved soon.