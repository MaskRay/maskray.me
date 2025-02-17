---
layout: post
title: Awesome 3 debian menu
author: MaskRay
tags: [awesome, desktop]
---

Awesome 3 裏可以啓用 debian menu，方法是新建一個帶 x 屬性的文件

```
#!/usr/bin/install-menu
# this file has to be executable
# put under ~/.menu-methods
# will run by update-menus
# default generate ~/.config/awesome/menu.lua
# you need to require("menu") to use menu.debian_menu

compat="menu-1"

!include menu.h

compat="menu-2"
outputencoding= "UTF-8";

function q($s) = "\"" esc($s,"\\\"") "\"";
function s($s) = replacewith(replacewith($s,"/","_"), " ", "_");
function findicon($filename)=
       ifelsefile($filename, q($filename),
        iffile("/usr/share/pixmaps/" $filename,
                   q("/usr/share/pixmaps/" $filename)));
function x11menu()= "\t{"q(title())","q($command) ifnempty($icon, ","findicon($icon))"},\n";
function textmenu()= "\t{"q(title())", \"x-terminal-emulator -e \".."q($command) ifnempty($icon, ","findicon($icon))"},\n";

supported;
    x11= x11menu();
    text= textmenu();
endsupported;

startmenu=      s($section)" = {\n";
endmenu=        "}\n";
submenutitle=   "\t{"q(title())","s($section)"},\n";
genmenu=        "menu.lua";
rootsection=    "debian_menu";
userprefix=     "/.config/awesome/";
preoutput=      "-- automatically generated file. Do not edit (see /usr/share/doc/menu/html)\n\nmodule(\"menu\")\n\n";
```

放在 ~/.menu-methods下，然後運行

```
$ update-menus
```

這樣就會在 ~/.config/awesome 下面建立 menu.lua，然後在 lua.rc 裏做如下修改：

```lua
-- Load Debian menu entries
require("debian.menu")



mymainmenu = awful.menu({ items = { { "awesome", myawesomemenu, beautiful.awesome_icon },
                                    { "open terminal", terminal },
                    { "debian", debian.menu.Debian_menu.Debian }
                                  }
                        })
```

參考：[http://awesome.naquadah.org/wiki/Awful.menu]()
