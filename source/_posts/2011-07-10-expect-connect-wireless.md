---
layout: post
title: 用Expect連接無線網絡
author: MaskRay
tags: [expect, network]
---

### 垃圾的無線網卡驅動

我的無線網卡是 Broadcom BCM57780，這個東西，Linux下的無線驅動做得非常爛。以前我用Gentoo Portage中的`net-wireless/broadcom-sta`，後來聽 _microcai_ 的，直接在 menuconfig 裏配置。這個驅動對應的模塊名稱是 brcmsmac，必須編譯成模塊（編譯進內核的話，我沒成功打開無線過）。它還需要 firmware，而且路徑是定死的（使用其他名稱是不行的，至少我沒成功）。它在 dmesg 中的信息非常簡略，如果你 firmware 的路徑配置錯的話，每次啓動有一定機率 dmesg 會提示你正確的路徑（這個……）。

<!-- more -->

這個驅動，偶爾會造成 kernel panic，掉線也挺頻繁的，其實要連線都很困難……

以前我都是用`wpa_supplicant -B -i wlan0 -c /etc/wpa_supplicant/wpa_supplicant.conf`後在`wpa_cli`裏輸入`select_network`，然後間斷性輸入`status`希望出現一個r`wpa_state CONNECTED`，這樣就連上了。如果十幾秒還沒連上的話，就不用指望再過幾分鐘它會連上。我一般是 kill 掉 wpa_supplicant 再來。有時如此幾次還是不行，我就用`modprobe`重新加載`brcmsmac.ko`，如果再不行我就該重啓了。

### 用 Expect 自動化連接過程

    #+BEGIN_SRC tcl -n
    #!/bin/sh
    # -*- tcl -*- \            (ref:1)
    exec tclsh "$0" "$@"       (ref:2)

    package require Expect

    for {set i 0} {$i < 5} {incr i} {
        catch {system wpa_supplicant -B -i wlan0 -c /etc/wpa_supplicant/wpa_supplicant.conf}
        catch {system wpa_cli select_network 0}

        for {set try 0} {$try < 12} {incr try} {
            spawn wpa_cli status
            expect {
                -re {wpa_state=(.*)\r} {
                    if {$expect_out(1,string) eq "COMPLETED"} {
                        system dhcpcd wlan0
                        exit
                    }
                    sleep 0.7
                }
                -re .+ exp_continue
                default
            }
            wait
        }
        catch {system killall wpa_supplicant}
        catch {system modprobe -r brcmsmac}
        catch {system modprobe brcmsmac}
    }

注意 (ref:1) (ref:2) 這兩行，這裏用了一個技巧，就是 sh 的 # 註釋是不續行的，而 Tcl 的是續行的(Makefile 也續行)。如果用 sh 執行，就會執行行 (ref:2)，用 tclsh 來解釋這個腳本。tclsh 則會忽略行 (ref:2)。
