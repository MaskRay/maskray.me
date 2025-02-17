---
layout: post
title: 一次服務器BMC固件逆向經歷
author: MaskRay
tags: [firmware,bmc,reverse engineering]
---

## 導出文件

訪問BMC默認開啓的Web管理界面`http://$ip`，開啓`ssh`服務，之後即可以執行`ssh root@$ip`，得到一個受限的管理界面，功能很少，沒有系統shell。

在服務器上執行`ipmitool fru`可以查找到設備型號，使用廠商提供的flash備份工具導出flash，比較慢，每秒200多KB。在BMC網頁界面的`BMC System Audit Log`中看到BMC的Linux系統日誌：

<!-- more -->

```
kernel: Kernel command line: root=/dev/ramdisk ro ip=none ramdisk_blocksize=4096 console=ttyS4,38400 rootfstype=cramfs bigphysarea=6144 imagebooted=1
kernel: Memory: 201216KB available (3080K code, 261K data, 128K init)
kernel: Ractrends Flash mapping: 0x2000000 at 0x20000000
kernel: Creating 6 MTD partitions on "Ractrends":
kernel: 0x00000000-0x02000000 : "fullpart"
kernel: 0x00050000-0x000c0000 : "conf"
kernel: 0x000d0000-0x00140000 : "bkupconf"
kernel: 0x00150000-0x00f10000 : "root"
kernel: 0x010c0000-0x01410000 : "www"
kernel: 0x01600000-0x01ff0000 : "lmedia"
```

從中得到flash的分區信息，包含`conf` `bkupconf` `root` `www` `lmedia`等分區。用binwalk查看導出的flash可以看到u-boot和一個存儲內核的u-boot legacy image。

根據Linux系統日誌中描述的`conf`偏移提取`conf`，binwalk可以確定文件系統爲[JFFS2]，而報告的很多zlib壓縮塊則是JFFS2的數據。本機模擬一個mtdblock設備並掛載：

```
modprobe mtdram total_size=32768 erase_size=64
modprobe mtdblock
dd if=conf of=/dev/mtdblock0
mount -t jffs2 /dev/mtdblock0 /mnt
```

發現很多文件是通常Linux系統`/etc`下的配置文件。如果調大`conf`結束偏移量，還能看到更多文件，不清楚模擬出來的mtdblock有哪些差異。

## 修改登錄shell

猜測`conf`分區會被作爲`/etc`使用，因此修改flash中`conf`分區處的`passwd`，把`root`的shell改成`/bin/sh`，`umount /mnt`後再次執行上述掛載操作，在`dmesg`中能看到內核輸出的JFFS2 checksum錯誤，其中包含正確的校驗和。在flash中搜索錯誤的校驗和，修改成正確的。用flash備份工具寫入修改後的flash，稍等片刻等待BMC自動重啓，再次執行ssh即看到shell換成了busybox！實際測試沒有這麼順利，修改了幾次都發現沒效果，原因是系統檢測到JFFS2文件系統checksum錯誤後自動還原分區。

查看掛載點：

```
BusyBox v1.13.2 (2015-01-08 17:43:56 CST) built-in shell (ash)
Enter 'help' for a list of built-in commands.

# mount
rootfs on / type rootfs (rw)
/dev/root on / type cramfs (ro)
/dev/proc on /proc type proc (rw)
sys on /sys type sysfs (rw)
/dev/shm on /var type tmpfs (rw)
/dev/ram2 on /usr/local/www type cramfs (ro)
/dev/mtdblock1 on /conf type jffs2 (rw)
/dev/mtdblock2 on /bkupconf type jffs2 (rw)
/dev/mtdblock5 on /usr/local/lmedia type jffs2 (rw)
devpts on /dev/pts type devpts (rw,gid=5,mode=620)
tmpfs on /usr/local/www/tmp type tmpfs (rw)
/dev/mtdblock5 on /usr/local/www/blackbox type jffs2 (rw)
/dev/shm on /usr/local/tmplmedia type tmpfs (rw)

# cat /etc/fstab
/dev/mtdblock1 /conf      jffs2 defaults 0 0
/dev/mtdblock2 /bkupconf  jffs2 defaults 0 0
/dev/root      /          auto defaults,rw 0 0
/dev/mtdblock5 /usr/local/lmedia jffs2 defaults 0 0
proc           /proc      proc defaults 0 0
sys            /sys       sysfs defaults
devpts         /dev/pts   devpts gid=5,mode=620
```

注意`/dev/mtdblock{0..5}`，它們是同一塊SPI flash，kernel中應該編碼了劃分方案，但尚不清楚這個信息是怎麼存儲的。

## 導出[initrd](https://www.kernel.org/doc/Documentation/initrd.txt)

內核啓動選項爲`root=/dev/ramdisk ro ip=none ramdisk_blocksize=4096 console=ttyS4,38400 rootfstype=cramfs bigphysarea=6144 imagebooted=1`，`/dev/ramdisk`即爲u-boot提供的initrd，使用只讀文件系統[cramfs]，啓動後不會通過`switch_root`等方式更換根文件系統。可以用`ssh root@$ip cat /dev/ramdisk`把initrd複製到本地，掛載後發現`/etc`下有很多指向`/conf`的軟鏈接，比如`/etc/passwd -> /conf/passwd`。這就解釋了爲什麼之前修改`/conf/passwd`可以改變系統的登錄shell。

但是以initrd的數據在flash中按圖索驥卻查不到。暫時擱置。

值得注意的是負責掛載網頁管理界面程序所用分區的啓動腳本`/etc/init.d/init-sp.sh`，它把flash的分區`/dev/mtdblock4`(www)掛載到`/usr/local/www`：

```
dd if=/dev/$wwwmtdblock of=/dev/ram2
/usr/local/bin/dencryption /dev/ram2 /dev/$wwwmtd
mount -t cramfs /dev/ram2 /usr/local/www
```

這裏用到了程序`/usr/local/bin/dencryption`，它從第二個參數中獲取長度，使用ECB模式的[Tiny Encryption Algorithm](https://en.wikipedia.org/wiki/Tiny_Encryption_Algorithm)原地解密第一個參數(此處爲`/dev/ram2`)。程序中編碼了密鑰，而在flash的u-boot bootloader區域也能找到該密鑰，發現initrd也是被同樣方式解密後掛載的，而執行這一解密過程的只能是u-boot。猜測開發人員修改u-boot，使用了Tiny Encryption Algorithm解密initrd傳遞給kernel。

提取flash中0x150000到0xf10000處內容，用`dencryption`解密算法得到u-boot legacy image，`tail -c+65`即可得到initrd，和`/dev/ramdisk`一致。

## 交叉編譯工具鏈

```
# uname -a
Linux 6C92BF0BA1B6 2.6.28.10-ami #1 Thu Jan 8 17:50:26 CST 2015 armv5tejl unknown
# head -n1 /proc/meminfo
MemTotal:         215668 kB
```

安裝crosstool-ng，執行`ct-ng menuconfig`配置交叉編譯工具鏈：

```
Paths and misc options  --->
  [*] Debug crosstool-NG
  [ ]   Pause between every steps
  [*]   Save intermediate steps
  [*]     gzip saved states
  [*]   Interactive shell on failed commands

Target options  --->
      Target Architecture (arm)  --->
  ()  Suffix to the arch-part
      *** Generic target options ***
  [ ] Build a multilib toolchain (READ HELP!!!)
  [*] Use the MMU
      Endianness: (Little endian)  --->
      Bitness: (32-bit)  --->
      *** Target optimisations ***
  (armv5te) Architecture level
  (arm926ej-s) Emit assembly for CPU
  ()  Tune for CPU
  ()  Use specific FPU
      Floating point: (software (no FPU))  --->
  ()  Target CFLAGS
  ()  Target LDFLAGS
      *** arm other options ***
      Default instruction set mode (arm)  --->
  [ ] Use Thumb-interworking (READ HELP)
  -*- Use EABI
```

gcc、glibc、binutils等有複雜的版本依賴關係，近年gcc make sed又都升級了主版本號，如果選擇與BMC系統相近的glibc-2.11，`./configure`時會報錯。方便起見glibc選擇新版本，這樣某些程序會因爲依賴高版本符號而無法在舊glibc系統上執行，暫且不管。爲了調試方便把strace和ltrace也選上。

`ct-ng build`。失敗後可以用`ct-ng list-steps`和`ct-ng libc+`等，避免前功盡棄。我在tmpfs下編譯，完成後佔用了近7GiB。

## 其他

內核啓動參數bigphysarea=6144用於分配設置連續的物理內存，`/proc/bigphysarea`、`/proc/iomem`可以看到地址的使用情況。

很多服務如kvm、虛擬磁盤等都用到了service location protocol，讀取SLP配置文件，有待了解。

`/usr/local/bin/IPMIMain`負責處理system/LAN等interface的IPMI請求。Web管理界面和原來的shell都經過自行實現的libipmi.so，請求最終發往IPMIMain監聽的unix socket。很多符號名和<https://www.virustotal.com/en/file/e72785167675ab46c8abd7bc29e66488d78bc5c9ac494c8ca4abcfd5ae94e0e0/analysis/>相同。

默認開啓snmpd，但Web管理界面不顯示。可以用`snmpwalk -u $username -l authPriv -a MD5 -x DES -A $password -X $passsword $ip`獲取MIB樹信息。

IPMI SDR數據取自`/dev/{adc0,gpio0,i2c2,netmon,pwmtach0}`等。

調試IPMIMain時需要關閉`/etc/init.d`中的watchdogapp以避免服務不可用導致系統自動重啓。

## 參考資料

- [bigphysarea for 2.6.9](https://lwn.net/Articles/111132/)
- [Unpacking and repacking U-Boot uImage files](http://www.isysop.com/unpacking-and-repacking-u-boot-uimage-files/)
- <http://www.linux-mtd.infradead.org/faq/general.html>
