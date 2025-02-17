---
layout: post
title: PyGTK迷宮生成器mazer
author: MaskRay
tags: [algorithm, maze, python]
---

隨機生成一個迷宮
![](/static/2010-01-15-maze-generator-with-pygtk/mazer.png)

```python
#!/usr/bin/env python

from sys import stdout
from random import randrange, shuffle
import gtk

BORDER = 10
LENGTH = 24

class mazer():
    def __init__(self, maxw, maxh):
        self.maxw, self.maxh = maxw, maxh
        self.map = [[False for c in xrange(self.maxw*2+1)] for r in xrange(self.maxh*2+1)]
        self.map[randrange(1, self.maxh*2, 2)][0] = True
        self.map[randrange(1, self.maxh*2, 2)][self.maxw*2] = True
        self.DFS(randrange(1, self.maxw*2, 2), randrange(1, self.maxh*2, 2))

    def run(self):
        window = gtk.Window()
        window.set_title('mazer')
        window.connect('destroy', gtk.main_quit)

        darea = gtk.DrawingArea()
        darea.size(LENGTH*self.maxw+BORDER*2, LENGTH*self.maxh+BORDER*2)
        darea.connect('expose_event', self.expose)
        window.add(darea)

        window.show_all()
        gtk.main()

    def DFS(self, x, y):
        self.map[y][x] = True
        ds = [(1,0),(0,1),(-1,0),(0,-1)]
        shuffle(ds)
        for d in ds:
            if 0 <= x+d[0]*2 < self.maxw*2 and 0 <= y+d[1]*2 < self.maxh*2 and not self.map[y+d[1]*2][x+d[0]*2]:
                self.map[y+d[1]][x+d[0]] = True
                self.DFS(x+d[0]*2, y+d[1]*2)

    def expose(self, widget, data):
        for y in xrange(self.maxh+1):
```

## 2013年8月17日更新

現在知道這個算法的名稱了：recursive backtracking，可以參見拙作[完美迷宮生成算法](/blog/2012-11-02-perfect-maze-generation)
