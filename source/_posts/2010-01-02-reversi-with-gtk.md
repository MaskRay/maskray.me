---
layout: post
title: GTK+黑白棋reversi
author: MaskRay
tags: [game, gtk]
---

language: C99 + GTK+
algorithm：對棋盤各個格子設置權值，計算總值
icon：程序截圖
license: GNU General Public License v3
revision control: Mercurial
bead-black.png bead-white.png texture1.png texture2.png: 8pm(http://hi.baidu.com/eightpm) 提供

gdk透明貼圖好像有點麻煩（以前 Win32 API 用得是 TransparentBlt）

![](/static/2010-01-02-reversi-with-gtk/reversi.png)

菜單採用GtkUIManager，可以看demo：`/usr/share/gtk-2.0/demo/ui_manager.c`的透明貼圖：

```cpp
void pixbuf_transparent(GdkPixbuf *dst, const GdkPixbuf *src, gint dstx, gint dsty, guint transcolor)
{
    guchar r = (transcolor & 0xFF0000) >> 16;
    guchar g = (transcolor & 0x00FF00) >> 8;
    guchar b = transcolor & 0x0000FF;
    guchar *p = gdk_pixbuf_get_pixels(dst),
           *q = gdk_pixbuf_get_pixels(src);
    int d = gdk_pixbuf_get_n_channels(src),
        dd = gdk_pixbuf_get_rowstride(src),
        d2 = gdk_pixbuf_get_n_channels(dst),
        dd2 = gdk_pixbuf_get_rowstride(dst),
        w = gdk_pixbuf_get_width(src),
        h = gdk_pixbuf_get_height(src);
    for (int y = 0; y < h; ++y)
        for (int x = 0; x < w; ++x)
        {
            guchar *pp = p + (dsty+y) * dd2 + (dstx+x) * d2;
            guchar *qq = q + y * dd + x * d;
            if (qq[0] != r || qq[1] != g || qq[2] != b)
                pp[0] = qq[0], pp[1] = qq[1], pp[2] = qq[2];
        }
}
```
