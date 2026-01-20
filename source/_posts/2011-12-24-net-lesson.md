---
layout: post
title: 網絡學堂feeds2mail
author: MaskRay
tags: [ruby]
---

用網頁來看網絡學堂的公告很費時間，就打算像 `rss2email` 那樣，把消息製作成郵件。
用 `Ruby` 的 `mechanize` 來和網站交互，讀取“課程公告”和“課程文件”中的消息，
把鏈接 `sha1` 後判斷是否生成過提示郵件，沒有則跟蹤鏈接，用 `w3m` 輸出成純文本，
生成的郵件用 `sendmail` 投遞。

<!-- more -->

```ruby
##!/usr/bin/ruby
require 'base64'
require 'date'
require 'digest/sha1'
require 'etc'
require 'mechanize'
require 'set'

agent = Mechanize.new
agent.max_history = 1

conf_dir = File.expand_path('~/.net_lesson')
unless File.directory?(conf_dir) && File.file?(File.join(conf_dir, 'passwd'))
  STDERR.puts 'echo [userid] [passwd] > ~/.net_lesson/passwd'
  exit 1
end
userid, passwd = File.open(File.join(conf_dir, 'passwd'), 'r') {|f| f.gets.split }
feeds = Set.new
File.open(File.join(conf_dir, 'feeds.dat'), 'r:binary') do |f|
  begin
    while (h = f.read 20)
      feeds.add h
    end
  rescue EOFError
  end
end
new_feeds = []
puts "loaded #{feeds.size} feeds" unless feeds.empty?

page = agent.get('http://learn.tsinghua.edu.cn/')
form = page.form('form1')
form.field_with(:name => 'userid').value = userid
form.field_with(:name => 'userpass').value = passwd
agent.submit(form)
puts 'login'

page = agent.get('http://learn.tsinghua.edu.cn/MultiLanguage/lesson/student/MyCourse.jsp?language=cn')
page.links_with(:href => /course_locate.jsp/).each do |lesson|
  lesson_name = lesson.text.gsub(/\s/, '').sub(/\(.*/, '')
  page = lesson.click
  puts "checking #{lesson_name}"
  ['getnoteid_student.jsp', 'download.jsp'].collect do |uri|
    download = uri == 'download.jsp'
    page2 = page.link_with(:href => /#{uri}/).click
    page2.links_with(:href => /note_reply|filePath/).each do |note|
      h = Digest::SHA1.digest note.href
      next if feeds.member? h
      puts "  found #{note.text.strip}"
      author = (download ? 'file ' : '') + note.node.xpath("../following-sibling::td")[-2].text
      IO.popen(['/usr/sbin/sendmail', Etc.getlogin], 'w') do |f|
        bar = download ? "[#{lesson_name}]" : "*#{lesson_name}*"
        time = Date.parse(note.node.xpath("../following-sibling::td")[-1].text).strftime '%a, %d %b %Y 00:00:00 +0800'
        f.puts(<<EOF)
From: #{author} <#{Etc.getlogin}>
Subject: =?utf-8?B?#{Base64.strict_encode64("#{bar} #{note.text.strip}")}?=
Date: #{time}
User-Agent: net_lesson
Content-Type: text/plain; charset="utf-8"
Content-Transfer-Encoding: binary

EOF
        if download
          f.puts note.text
        else
          IO.popen(['w3m', '-dump', '-T', 'text/html'], 'r+') do |p|
            p.puts(note.click.body)
            p.close_write
            f.puts p.read
          end
        end

        f.puts "\nURI: #{page2.uri.merge URI.escape(note.href, /[\u4E00-\u9FFF]/)}"
      end
      new_feeds << h
    end
  end
end

unless new_feeds.empty?
  puts "appending to feeds.dat"
  File.open(File.join(conf_dir, 'feeds.dat'), 'a:binary') {|f| new_feeds.each {|a| f.write(a) } }
end
```

## 2014年11月30日更新

這個方案已廢棄，現在改用newsbeuter閱讀rss了。並使用一個Ruby腳本抓取通知。

```ruby
#!/usr/bin/ruby
#encoding: utf-8
require 'open-uri'
require 'nokogiri'
require 'rss'

s = open('http://oars.tsinghua.edu.cn/zzh/30630.nsf/infobytime?openview').read
m = Nokogiri.parse(s).xpath('//script/text()')[0].text.match(/(?<=    location\.replace\(")[^"]*(?=")/)
s = open("http://oars.tsinghua.edu.cn#{m[0]}").read
s = s.force_encoding('gbk').encode 'utf-8'
d = Nokogiri.parse(s)

rss = RSS::Maker.make("atom") do |maker|
  maker.channel.author = ''
  maker.channel.about = ''
  maker.channel.updated = Time.now.to_s
  maker.channel.title = '教務通知'

  d.xpath('//tr[contains(@valign, "top")]').each {|tr|
    tds = tr.search('td')
    next if tds.size != 4
    maker.items.new_item {|item|
      #item.link = tds[2].search('a')[1].attr 'href'
      item.link = (URI('http://oars.tsinghua.edu.cn') + tds[2].search('a')[1].attr('href')).to_s
      item.title = tds[2].text
      item.updated = Time.parse tds[3].text
    }
  }
end

File.write '/tmp/教務通知.rss', rss.to_s


rss = RSS::Maker.make("atom") do |maker|
  maker.channel.author = ''
  maker.channel.about = ''
  maker.channel.updated = Time.now.to_s
  maker.channel.title = '重要通知'

  t = Time.now
  d = Nokogiri::HTML open('http://info.tsinghua.edu.cn/html/view/notice_beforelogin.htm')
  d.xpath('//td').each {|td|
    next if td.children.size != 2
    a = td.search('a')[0]
    maker.items.new_item {|item|
      item.link = (URI('http://oars.tsinghua.edu.cn') + a.attr('href')).to_s
      item.title = a.text
      item.updated = t
    }
  }
end

File.write '/tmp/重要通知.rss', rss.to_s
```

然後用fcron定期執行上面的腳本，產生`/tmp/教務通知.rss`和`/tmp/重要通知.rss`：
```
% fcrontab -l
@ 1h /home/ray/bin/教務通知.rb 2>> /tmp/stderr
```

`~/.config/newsbeuter/urls`裏添加下面兩行：
```
file:///tmp/教務通知.rss
file:///tmp/重要通知.rss
```
