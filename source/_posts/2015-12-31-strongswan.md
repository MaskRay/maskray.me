---
layout: post
title: 支持Android、iOS 9內置IPSec客戶端的strongSwan 5.3.5配置
author: MaskRay
tags: [vpn,ipsec]
---

## 服務器

Arch Linux可以安裝`aur/strongswan`，Debian可以安裝unstable倉庫的`strongswan`和`libcharon-extra-plugins`。Ubuntu等發行版，軟件倉庫中strongswan較舊，建議編譯安裝最新版本。

<!-- more -->

在服務器上執行：

```bash
# CA key & certificate
ipsec pki --gen > caKey.der
ipsec pki --self --in caKey.der --dn 'C=JP, O=mynet, CN=CA' --ca > caCert.der
openssl x509 -inform der -in caCert.der -out caCert.pem

# server key & certificate
ipsec pki --gen > serverKey.der
ipsec pki --pub --in serverKey.der | ipsec pki --issue --cacert caCert.der --cakey caKey.der --dn 'C=JP, O=mynet, CN=your.ip' --san 'your.domain' > serverCert.der
openssl x509 -inform der -in serverCert.der -out serverCert.pem
openssl rsa -inform der -in serverKey.der -out serverKey.pem

# 本文未用到(iOS 9公鑰認證未試驗成功，Linux strongSwan客戶端公鑰認證產生1580字節packet超過MTU，分片未試驗成功)。若需生成client key & certificate
#ipsec pki --gen > clientKey.der
#ipsec pki --pub --in clientKey.der | ipsec pki --issue --cacert caCert.der --cakey caKey.der --dn "C=JP, O=azurejp, CN=client" --san 'client@your.domain' > clientCert.der
#openssl x509 -inform der -in clientCert.der -out clientCert.pem
#openssl rsa -inform der -in clientKey.der -out clientKey.pem

mkdir -p /etc/ipsec.d/{certs,cacerts,private}
cp caCert.pem /etc/ipsec.d/cacerts/
cp serverCert.pem /etc/ipsec.d/certs/
cp serverKey.pem /etc/ipsec.d/private/
```

編輯`/etc/ipsec.conf`：

```
config setup
  uniqueids = never

conn ikev2ios
  keyexchange = ikev2
  ike = aes256-sha256-modp1024,3des-sha1-modp1024,aes256-sha1-modp1024!  # for iOS 9
  esp = aes256-sha256,3des-sha1,aes256-sha1!                             # for iOS 9

  left = %any
  leftid = your.domain                                                   # iOS 9似乎要求此處爲域名或IP
  leftsendcert = always                                                  # iOS 9會驗證證書
  leftcert = serverCert.pem
  leftsubnet = 0.0.0.0/0                                                 # 給iOS 9推送的路由

  right = %any
  rightauth = eap-mschapv2
  rightsourceip = 10.99.1.0/24
  eap_identity = %any
  auto = add
  fragmentation = yes

conn ikev1android
  keyexchange = ikev1
  left = %any
  leftid = your.ip
  leftsendcert = always
  leftcert = serverCert.pem
  leftsubnet = 0.0.0.0/0
  right = %any
  rightauth = xauth
  rightsourceip = 10.99.1.0/24
  auto = add
```

注意修改`your.ip`和`your.domain`。

iOS 9內置的IKEv2 VPN客戶端會驗證服務端證書是否爲本地某信任CA簽署的，且“遠程ID”匹配證書的CN(生成`serverCert.der`時的`--dn`選項中)或subject alternative name (生成`serverCert.der`時的`--san`選項)。CN、subject alternative name可以填域名或IP地址，域名可以亂寫，不會檢查域名是否和服務器IP對應。填寫服務器的域名或IP地址可行，其他可能項如郵箱地址等不知是否可行。

編輯`/etc/ipsec.secrets`：

```
: RSA serverKey.pem
client : EAP "your.password"       # iOS及Linux的EAP-MSCHAPv2
client : XAUTH "your.password"     # Android IPSec hybrid RSA
```

注意修改用於EAP-MSCHAPv2的密碼`your.password`。

服務器需要允許外界訪問500/udp和4500/udp。客戶端連接後，可以把服務端作爲路由的下一跳，服務端需要配置轉發：

```bash
sysctl -w net.ipv4.ip_forward=1
iptables -t nat -A POSTROUTING -s 10.99.1.0/24 -j MASQUERADE
```

某些使用環境可能會設定默認DROP、禁止FORWARD等，需要更復雜的配置，此處不贅述。

若要推送nameserver給客戶端，可以編輯`/etc/strongswan.conf`：

```
charon {
  load_modular = yes
  duplicheck.enable = no
  compress = yes

  dns1 = your.dns1
  dns2 = your.dns2

  plugins {
    include strongswan.d/charon/*.conf
  }
}
```

`ipsec start`啓動服務端守護進程，如果使用systemd的話，可能是`systecmtl start strongswan`之類。

## Linux strongSwan客戶端

把`caCert.pem`複製到本地，`cp caCert.pem /etc/ipsec.d/cacerts/`。

編輯`/etc/ipsec.conf`，設定名爲`myikev2`的連接：

```
config setup
  niqueids = never

conn myikev2
  keyexchange = ikev2
  right = your.ip
  rightca = caCert.pem
  rightid = your.domain
  rightsubnet = 0.0.0.0/0

  left = %any
  leftsourceip = %config
  leftauth = eap-mschapv2
  eap_identity = client
  auto = start
```

注意修改`your.domain`。

編輯`/etc/ipsec.secrets`，需要服務端中指定的EAP密碼：

```
client : EAP "your.password"
```

`rightsubnet`會影響到`myikev2`連接建立後的新建的路由。`ip ru`可以看到多了一個ID爲220的路由表，若把`rightsubnet`改爲`8.0.0.0/8`，會看到`ip r s t 220`輸出：
```
# 我本地無線網接口wlp3s0 IP 192.168.0.3，網關爲192.168.0.1。wlp3s0上多出了一個10.99.1.1/32的由服務端推送的[virtual IP
8.0.0.0/8 via 192.168.0.1 dev wlp3s0  table 220  proto static  src 10.99.1.1
```

若要測試是否可以把服務端作爲網關，可以用如下命令：
```bash
# 添加
ip r a 180.149.132.47 dev enp4s0f2 src 10.99.1.1
ping 180.149.132.47
# 刪除
ip r d 180.149.132.47 dev enp4s0f2 src 10.99.1.1
```

[Virtual IP](https://wiki.strongswan.org/projects/strongswan/wiki/VirtualIP)。待補充。

## iOS 9內置IKEv2客戶端

在iOS 9中打開`caCert.pem`，會提示導入“描述文件”，之後可以在“設置->通用->描述文件”看到該證書。

iOS 9中亦可導入用於客戶端認證的私鑰及證書，但必須是PKCS12格式，且有四位passphrase，可以用如下命令創建：
```bash
openssl pkcs12 -export -inkey clientKey.pem -in clientCert.pem -name 'client' -certfile caCert.pem -caname 'CA' -passout 'pass:1234' -out client.p12
```

尚未弄明如何使用iOS 9內置客戶端的公鑰認證，但可以使用EAP-MSCHAPv2方式認證。“設置->通用->VPN->添加VPN配置”，填寫如下字段：

- 類型：`IKEv2`
- 服務器：`your.ip`
- 遠程ID：服務端`/etc/ipsec.conf`中指定的`leftid`
- 用戶鑑定：`用戶名`
- 用戶名：`client` (服務端`/etc/ipsec.secrets`中配置了名爲`client`的EAP identity)
- 密碼：`your.password`

iOS調試還是挺麻煩的，VPN連接失敗什麼錯誤消息都沒有，只能看服務端的日誌……對iOS認識太少。

## Android

- 類型：IPSec Hybrid RSA
- 服務器地址：`your.ip`
- IPSec CA證書：`caCert.pem`
- IPSec服務器證書：`serverCert.pem`。“設置->安全->從存儲設備安裝”中可以安裝。

測試時，“IPSec CA證書”、“IPSec服務器證書”可留空。長按新建的VPN條目可以修改配置。

較新的strongSwan似乎關閉Aggressive Mode PSK，默認無法使用IPSec Xauth PSK：
```
Aggressive Mode PSK disabled for security reasons.
```

2025年更新：Android 14

* 用`jmtpfs`安裝證書文件到Android手机
* Settings->Security and privacy->More security settings->Install from device storage，選擇證書文件
* Connections->More connection settings->VPN->Add VPN profile。選擇Type: IKEv2/IPSec RSA，填寫Server address:及IPSec user certificate:

## 調試

`journal -fb _SYSTEM_UNIT=strongswan.service`看服務端日誌，或者用`ipsec start --nofork`在前臺開啓。客戶端修改後`ipsec restart`看日誌，某些改動可以使用`ipsec reload`、`ipsec rereadsecrets`等。

### 常見錯誤

#### 服務端找不到匹配的`conn`配置：

```
looking for peer configs matching 10.0.0.4[server]...1.2.3.4[client]   # 未仔細研究，似乎對應配置文件中的：left[leftid]...right[rightid]
no matching peer config found
```

### 服務端strongswan不支持特定認證方式

`journalctl -fb`後重啓ipsec服務，看到如下字樣：
```
14[IKE] EAP-Identity request configured, but not supported
14[IKE] loading EAP_MSCHAPV2 method failed
```

#### 服務端未推送nameserver給客戶端

Android、iOS難以看到IPSec客戶端錯誤信息，可以嘗試用IP訪問網頁。

#### Path MTU discovery

假設服務器上外網接口爲`eth0`，根據該接口的MTU與網站協商了一個MSS，但該MSS加上IP header很可能會大於IPSec客戶端到服務器的MTU。服務器向網站發送ICMP Unreachable，但網站很可能屏蔽了該消息。

在服務器上使用命令`tcpdump -ni eth0 'port 80 and tcp[13] & 2 == 2'`觀察80/tcp帶有SYN的TCP packets發現協商的MSS。`tcpdump -ni eth0 'net 180.97.33.0/24'`發現網站發送的TCP packet與服務器發送的不被理會的ICMP Unreachable。
```
12:31:51.778576 IP (tos 0x0, ttl 44, id 42813, offset 0, flags [DF], proto TCP (6), length 1480)
    180.97.33.107.80 > 10.0.0.4.49016: Flags [.], cksum 0xba48 (correct), seq 1026:2466, ack 78, win 193, length 1440
12:31:51.778633 IP (tos 0xc0, ttl 64, id 59986, offset 0, flags [none], proto ICMP (1), length 576)
    10.0.0.4 > 180.97.33.107: ICMP 10.0.0.4 unreachable - need to frag (mtu 1438), length 556
        IP (tos 0x0, ttl 43, id 42813, offset 0, flags [DF], proto TCP (6), length 1480)
    180.97.33.107.80 > 10.0.0.4.49016: Flags [.], seq 1026:2466, ack 78, win 193, length 1440
12:31:52.194666 IP (tos 0x0, ttl 44, id 42814, offset 0, flags [DF], proto TCP (6), length 1480)
    180.97.33.107.80 > 10.0.0.4.49016: Flags [.], cksum 0xba48 (correct), seq 1026:2466, ack 78, win 193, length 1440
12:31:52.194740 IP (tos 0xc0, ttl 64, id 59995, offset 0, flags [none], proto ICMP (1), length 576)
    10.0.0.4 > 180.97.33.107: ICMP 10.0.0.4 unreachable - need to frag (mtu 1438), length 556
        IP (tos 0x0, ttl 43, id 42814, offset 0, flags [DF], proto TCP (6), length 1480)
    180.97.33.107.80 > 10.0.0.4.49016: Flags [.], seq 1026:2466, ack 78, win 193, length 1440
```

iOS客戶端初始協商的MSS爲1360，一般不會出問題。Linux strongswan客戶端/Android客戶端初始協商的MSS可能和MTU有關，對於MTU 1500，協商了MSS 1460，無法訪問網頁。解決辦法是調低服務器向IPSec客戶端通報的MSS：
```bash
iptables -t mangle -A FORWARD -s 10.99.1.0/24 -p tcp -m tcp --tcp-flags SYN,RST SYN -m tcpmss --mss 1361:1536 -j TCPMSS --set-mss 1360
```

## 參考

- <http://linsir.org/post/how_to_install_IPSec_IKEV2_base_on_strongswan_with_CentOS7>
- <http://blog.itnmg.net/centos7-ipsec-vpn/>
- <https://www.strongswan.org/uml/testresults/index.html>
- <https://strongswan.net/blog/how-to-resolve-mtu-issue-with-ipsec-tunnel/>
