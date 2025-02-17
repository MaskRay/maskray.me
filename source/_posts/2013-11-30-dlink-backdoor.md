---
layout: post
title: D-Link路由器後門註記
author: MaskRay
tags: [reverse engineering]
---

這是學習 http://www.devttys0.com/2013/10/reverse-engineering-a-d-link-backdoor 動手實踐時做的一些記錄。

下載 ftp://ftp.dlink.eu/Products/dir/dir-100/driver_software/DIR-100_fw_reva_113_ALL_en_20110915.zip

```
% binwalk -e DIR100_v5.0.0EUb3_patch02.bix

DECIMAL         HEX             DESCRIPTION
-------------------------------------------------------------------------------------------------------------------
4               0x4             Realtek firmware header (ROME bootloader) image type: RUN, header version: 1, created: 9/17/2011, image size: 1690484 bytes, body checksum: 0xA, header checksum: 0x67
13046           0x32F6          mcrypt 2.2 encrypted data, algorithm: blowfish-256, mode: CBC, keymode: 8bit
403343          0x6278F         gzip compressed data, has CRC, extra field, last modified: Tue Jun  6 18:51:20 2028
646032          0x9DB90         Squashfs filesystem, big endian, version 2.0, size: 1041011 bytes, 528 inodes, blocksize: 65536 bytes, created: Thu Sep 15 17:11:28 2011
```
得到`squashfs`文件系統`9DB90.squashfs`。留着這個文件不動，還要準備另一個工具。

<!-- more -->

`git clone https://code.google.com/p/firmware-mod-kit/` 後修改下面這個文件：

```
diff --git i/src/lzma/C/7zip/Compress/LZMA_Lib/ZLib.cpp w/src/lzma/C/7zip/Compress/LZMA_Lib/ZLib.cpp
index cb89b54..12b5b35 100644
--- i/src/lzma/C/7zip/Compress/LZMA_Lib/ZLib.cpp
+++ w/src/lzma/C/7zip/Compress/LZMA_Lib/ZLib.cpp
@@ -180,7 +180,7 @@ protected:
        UInt64 m_offset;
 };

-ZEXTERN int ZEXPORT compress2 OF((Bytef *dest,   uLongf *destLen,
+ZEXTERN int ZEXPORT compress2 _Z_OF((Bytef *dest,   uLongf *destLen,^M
                                   const Bytef *source, uLong sourceLen,
                                   int level))
 {
@@ -243,7 +243,7 @@ ZEXTERN int ZEXPORT compress2 OF((Bytef *dest,   uLongf *destLen,
        return Z_OK;
 }

-ZEXTERN int ZEXPORT uncompress OF((Bytef *dest,   uLongf *destLen,
+ZEXTERN int ZEXPORT uncompress _Z_OF((Bytef *dest,   uLongf *destLen,^M
                                    const Bytef *source, uLong sourceLen))
 {
        /* CJH: 7zip ID implemented by some LZMA implementations */
```

編譯`squashfs-2.1-r2`：

```bash
make -C src/squashfs-2.1-r2
```

可以使用`firmware-mod-kit`裏的`unsquashfs_all.sh`腳本解壓縮得到rootfs：

```
% unsquashfs_all.sh _DIR100_v5.0.0EUb3_patch02.bix.extracted/9DB90.squashfs rootfs
```

可以執行`strings rootfs/bin/webs`看到作爲後門的User-Agent。
