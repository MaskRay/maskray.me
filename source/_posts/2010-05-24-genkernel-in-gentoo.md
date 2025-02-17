---
layout: post
title: genkernel
author: MaskRay
tags: [kernel, gentoo]
---

第一次爲gentoo編譯內核，發現默認選項沒有有線網絡支持（沒有eth0設備），也不管哪些選項是自己真正需要的，選了很多。恰好`linux-2.6.34-gentoo`出來了，就嘗試着重新配置一下。

```bash
genkernel --bootloader=grub --menuconfig --no-clean all
```

以前不知道要用```--no-clean```，每次編譯都要花很長時間，這個選項可以讓`genkernel`不去執行
`make clean`，第二次編譯花的時間就會少很多

File systems
    <*> The Extended 4 (ext4) filesystem #我大部分分區用的是 ext4，這一項默認沒有設
    <*> Reiserfs support #/usr/portage 下有很多目錄和小文件，所以我單獨掛載在一個 reiserfs 分區
    -*- Native language support
        <*> Simplified Chinese charset (CP936, GB2312)
        <*> Traditional Chinese charset (Big5)

Executable file formats / Emulations
    [*] IA32 Emulation #這個好像是執行 32-bit ELF 的，否則像 firefox-bin wine 等就無法運行


我的網卡是Broadcom Corporation NetLink BCM57780 Gigabit Ethernet PCIe

```
Device Drivers
    Network device support
        PHY Device support and infrastructure
            [*] Drivers for Broadcom PHYs
        Ethernet (100 Mbit)
            [*] Broadcom Tigon3 support
```

我的framebuffer的配置：

```bash
emerge -av v86d
```

```
General Setup ->
    (/usr/share/v86d/initramfs) Initramfs source file(s)
Device Drivers
    <*> Connector - unified userspace <-> kernelspace linker
    Support for frame buffer devices
        [*] Enable firmware EDID
        [*] Enable Video mode handling helpers
    [ ] Enable Tile Blitting Support #選擇了這一項 CONFIG_FB_CON_DECOR 選項就沒有了
        [*] Userspace VESA VGA graphics support
    Console display driver support
        Framebuffer Console Support
            [*] Support for the Framebuffer Console Decorations
```
