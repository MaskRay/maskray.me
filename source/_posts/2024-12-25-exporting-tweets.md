---
layout: post
title: Exporting Tweets
author: MaskRay
tags: [website]
---

On <https://x.com/settings/>, click `More -> Settings and privacy -> Download an archive of your data`.
Wait for a message from x.com: "@XXX your X data is ready"
Download the archive.

```sh
cp data/tweets.js tweets.ts
```

Change the first line from `window.YTD.tweets.part0 = [` to `let part0 = [`,
and append

```typescript
import { unescape } from "@std/html/entities";

let out = part0.map(tw => [new Date(tw.tweet.created_at), tw.tweet.full_text])
out.sort((a,b) => a[0] - b[0])

let yy0 = 0, mm0 = 0, str = ''
for (let i=0, j=0; i<=out.length; i++) {
  let d = i<out.length ? out[i][0] : new Date('9999-12-31')
  let yy = d.getYear()+1900, mm = d.getMonth()+1
  if (yy0 != yy) {
    if (str.length) {
      try {
        Deno.mkdirSync(String(yy0))
      } catch (e) {
      }
      Deno.writeTextFileSync(`${yy0}/index.md`, str)
    }
    yy0 = yy
    mm0 = 0
    str = `# ${yy0}\n`
    if (i == out.length) break
  }
  if (mm0 != mm) {
    str += `\n## ${yy}-${String(mm).padStart(2,'0')}\n`
    mm0 = mm
  }
  str += `\n${unescape(out[i][1]).replace(/(http(s)?:[-/.\w]+)/, "<$1>")}\n`
}
```

Then run `deno run --allow-write=. tweets.ts`

```
% cat 2022/index.md
# 2022

## 2022-01

tweet0

tweet1

## 2022-02

...
```

tweet0

tweet1
