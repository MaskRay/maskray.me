---
layout: post
title: 使用Suricata進行IDS/IPS
author: MaskRay
tags: [security, suricata]
---

## Suricata

Intrusion detection System(IDS)監控網絡或系統，尋找各類違反安全方針的行爲。Suricata是Open Information Security Foundation和其他相關支持協會從2009年開始開發的一套用於網路入侵檢測(IDS)、入侵防護(IPS)及網絡監控的系統。這方面最著名的產品是Sourcefire的Snort。那麼爲什麼要選擇Suricata呢？我在這方面的知識還幾近空白，無法在架構之類的方面作出判斷，想的到的用Suricata的理由只有這三點：

<!-- more -->

- 支持多線程，而Snort沒有。
- 配置文件是YAML的，而Snort的是基於行的一種比較簡單的格式。
- Suricata也使用Snort的規則。

Suricata的文檔在起步階段，目前有一個[wiki](https://redmine.openinfosecfoundation.org/projects/suricata/wiki/Suricata_User_Guide)。看到Redmine又感慨萬分，此中有真意，欲辯已忘言。詩中的“真意”以前會和“深意”混淆，也確實很應景，閒話還是不說了。好在Suricata規則和Snort一致，看[Snort User Manual](http://www.snort.org/docs)就好了。

### 安裝

先下載Suricata的依賴`libhtp`：

```bash
git clone https://github.com/ironbee/libhtp.git
./autogen.sh
./configure
make install
```

如果怕弄髒系統可以在`./configure`時加上`--prefix=$HOME/.local`，INCLUDE文件會在`~/.local/include/`下。我現在的想法是裝到`ldconfig -v`列出來的目錄裏比較好，免去`CFLAGS`裏`-I$HOME/.local/include`，`LDLIBS`裏`-l$HOME/.local/lib64`的麻煩。

我是Gentoo用戶就寫了個簡易的`.ebuild`裝到`/usr/`下了。另外可以考慮用`stow`或`xstow`來管理這些源碼安裝的包。`stow`是Perl腳本，沒記錯的話是`Perl 4`時代的產物了，修修補補似乎能支持`Perl 5`了，但是用`Perl`新版本運行似乎會有warning。`xstow`是C寫的，看上去跟牢靠一些。以後有空還想嘗試Nix和基於Nix的GNU Guix。

```
Nix packages are configured using a lazy, pure-functional language especially designed for this purpose
```

能說出"lazy, pure-functional"就很不簡單了，不是嗎？

```bash
git clone git://phalanx.openinfosecfoundation.org/oisf.git
cd oisf
./autogen.sh
/configure --prefix=$HOME/.local --enable-nfqueue --enable-geoip --enable-non-bundled-htp
make install-full
```

如果沒有`--enable-nfqueue`的話，就無法使用IPS功能(相當於Snort的inline模式)，開啓這項功能需要用到這兩個包`libnetfilter_queue`、`libnfnetlink`。一些二進制發行版注意需要安裝相應的`-dev`開發包。

`--enable-non-bundled-htp`使用已經裝好的`libhtp`。`make install-full`會安裝一些配置文件和rules。

### 霜刃初試

#### IDS模式

如果之前執行過`make install-conf`或`make install-full`，就會有這個文件：`$PREFIX/etc/suricata/suricata.yaml`。使用

創建目錄`/tmp/suricata/`用於存放Suricata的日誌。另外新建一個規則文件`/tmp/a.rules`：

```snort
alert tcp $EXTERNAL_NET any -> $HOME_NET 8888 (msg: "meow"; content: "meow"; )
```

然後修改`suricata.yaml`中的這幾行：

```yaml
default-log-dir: /tmp/suricata/

rule-files:
  - /tmp/a.rules

outputs:
  - fast:
      enabled: yes
      filename: fast.log
      append: yes
  - unified2-alert:
      enabled: yes
      filename: unified2.alert
```

執行：

```bash
sudo suricata -c ~/.local/etc/suricata/suricata.yaml -i eth0
```

`-i`讓Suricata以IDS模式運行，抓`eth0`網絡接口的包進行分析，這個相當於Snort的passive模式。

執行`nc -l -p 8888`，在另外一臺機器上用`nc IP 8888`連接本機，鍵入`meow`並回車。觀察`/tmp/suricata/`目錄，會發現文件`fast.log`裏多了條記錄，這個就是Suricata檢測到的可疑數據包(規則文件`/tmp/a.rules`定義的)。而目錄下還會有個文件名形如`unified2.alert.$timestamp`的文件，使用了unified2格式，包含可以數據包的具體信息，可以轉化爲pcap格式供Wireshark分析。

#### IPS模式

修改之前的規則文件`/tmp/a.rules`：

```snort
alert tcp $EXTERNAL_NET any -> $HOME_NET 8888 (msg: "meow"; content: "meow"; replace: "wow "; )
```

和之前的區別是多了`replace`用來修改數據包，要求是替換前後的字串長度相同。

```bash
sudo suricata -c ~/.local/etc/suricata/suricata.yaml -q 0
```

`-q`讓Suricata以IPS模式運行，相當於`snort`的inline模式。不過還需要設置一下`iptables`的`NFQUEUE`，讓Suricata能訪問到相應的數據包，最簡單的方法是：

```bash
sudo iptables -I INPUT -p tcp -j NFQUEUE
sudo iptables -I OUTPUT -p tcp -j NFQUEUE
```

Suricata退出後別忘了用`sudo iptables -F`刪除這兩條規則，否則就無法使用TCP了。

參見[https://redmine.openinfosecfoundation.org/projects/suricata/wiki/Setting_up_IPSinline_for_Linux]()。

## Misc

### Rules和Oinkmaster

有了檢測系統還不夠，我們還需要規則集。根據[1]的說法，這方面的權威是Snort VRT Rules，但它個價格不菲，不過還在一段時間後就免費提供給公衆了。另外Emerging Threats ETPro提供了一些免費的[rules](http://rules.emergingthreatspro.com/open/suricata/)。

[`oinkmaster`](http://oinkmaster.sf.net/)可以用來自動化下載、管理這些rules。我用下面的腳本同步：

```bash
oinkmaster.pl -C /etc/oinkmaster.conf -o ~/.local/etc/suricata/rules -i
```

### Barnyard2

[Barnyard2](https://github.com/firnsy/barnyard2)是個"open source interpreter for Snort unified2 binary output files"。那麼什麼是`spool`呢？Simultaneous peripheral operations on-line。這個詞不算陌生，看看你的系統裏是不是有個叫`/var/spool/`的目錄？
對於spooler，我的理解是一個給某程序提供了output backend的外部程序。

Suricata配置文件`suricata.yaml`中的`outputs2 > unified2-alert`可以設定在產生alert時dump出可疑數據包的信息，這個格式的好處是：

- 方便歸檔管理
- 生成速度快。

`Barnyard2`就是個類似`Syslog`的東西，從Snort/Suricata處取得unified2格式的輸入，產生其他格式的輸出，比如給Prelude Hybrid IDS system、Syslog、MySQL。

Barnyard2的源碼包裏提供了一份配置文件`etc/barnyard2.conf`，我把它複製到了`~/.local/etc/suricata/barnyard2.conf`並修改了下面這幾行：

```snort
config reference_file:      /home/ray/.local/etc/suricata/reference.config
config classification_file: /home/ray/.local/etc/suricata/classification.config
config gen_file:            /home/ray/.local/etc/suricata/rules/gen-msg.map
config sid_file:            /home/ray/.local/etc/suricata/rules/sid-msg.map

config interface:       eth0

config set_gid: 1000
config set_uid: 1000

config waldo_file: /tmp/suricata/suricata.waldo

input unified2

output alert_fast
```

開頭幾行`reference_file`、`sid_file`之類的約定的一些狀態嗎之類的東西，如果之前用`make install-full`了，就省去了手動創建這些文件的麻煩。

創建目錄`/tmp/barnyard2/`放置Barnyard2的輸出。執行：

```bash
barnyard2 -c ~/.local/etc/suricata/barnyard2.conf -l /tmp/barnyard2 -f unified2.alert -d /tmp/suricata
```

Barnyard2會處理`/tmp/suricata/`下文件名以`unified2.alert`開頭的文件，並監控這個目錄等候新的unified2格式的alert文件。
Barnyard2處理完文件後會留下這樣的信息：

```
Opened spool file '/tmp/suricata/unified2.alert.1374809319'
Closing spool file '/tmp/suricata/unified2.alert.1374809319'. Read 0 records
Waiting for new data
```

最後一句"Waiting for new data"就是告訴你它已經處理完所有的已有unified2 alert文件，開始監控目錄等待新的alert文件了。

第一次運行Barnyard2，因爲`/tmp/suricata/suricata.waldo`不存在，Barnyard2會輸出警告：

```
WARNING: Unable to open waldo file '/tmp/suricata/suricata.waldo' (No such file or directory)
```

但是不影響Barnyard2的使用。`waldo_file`文件的作用就是記錄Barnyard2已經處理過了哪些文件，Barnyard2會自動創建這個文件。如果把`waldo_file`刪掉，重啓Barnyard2，會發現`/tmp/barnyard2/alert`裏又多了一份和之前一樣的alert。

### U2boat

U2boat是Snort附帶的工具，用來把unified2格式的文件轉換成pcap格式，這樣就可以使用Wireshark分析數據包了。

```bash
u2boat unified2.alert.1374814463 a.pcap
wireshark a.pcap
```

[1]: http://www.vorant.com/static/EZ_Snort_Rules.pdf
