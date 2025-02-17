layout: post
title: When QOI meets XZ
author: MaskRay
tags: [image,compression,xz]
---

[QOI](https://qoiformat.org/), the Quite OK Image format, has been gaining in popularity. Chris Wellons offers a great [analysis](https://nullprogram.com/blog/2022/12/18/).

QOI's key advantages is its simplicity.
Being a byte-oriented format without entropy encoding, it can be further compressed with generic data compression programs like LZ4, XZ, and zstd.
PNG, on the other hand, uses DEFLATE compression internally and is typically resistant to further compression.
By applying a stronger compression algorithm on QOI output, you can often achieve a smaller file size compared to PNG.

<!-- more -->

## XZ

Lasse Collin has shared some effective options for compressing uncompressed BMP/TIFF files.
I tested them on the [QOI benchmark images](https://qoiformat.org/benchmark/).

When the color table (palette) is used, a delta filter would increase the compressed size and should be disabled.

```
% cat ~/tmp/b.sh
#!/bin/zsh -ue
f() {
  pngcrush -fix -m 1 -l 0 $1 ${1/.png/.uncompressed.png}
  [[ -f ${1/.png/.uncompressed.png} ]] || cp $1 ${1/.png/.uncompressed.png}
  /tmp/p/qoi/qoiconv $1 ${1/.png/.qoi}
  convert $1 ${1/.png/.bmp}
  convert $1 -compress none ${1/.png/.tiff}
  xz --lzma2=pb=0 -fk ${1/.png/.qoi}
  if [[ $(file $1) =~ RGBA ]]; then
    pnm=${1/.png/.pam}
    convert $1 $pnm
    xz --delta=dist=4 --lzma2=lc=4 -fk $pnm
    xz --delta=dist=4 --lzma2=lc=4 -fk ${1/.png/.bmp}
    xz --delta=dist=4 --lzma2=lc=4 -fk ${1/.png/.tiff}
  elif [[ $(file $1) =~ 'colormap' ]]; then
    pnm=${1/.png/.ppm}
    convert $1 $pnm
    xz --lzma2=pb=0 -fk $pnm
    xz --lzma2=pb=0 -fk ${1/.png/.bmp}
    xz --lzma2=pb=0 -fk ${1/.png/.tiff}
  else
    pnm=${1/.png/.ppm}
    convert $1 $pnm
    xz --delta=dist=3 --lzma2=pb=0 -fk $pnm
    xz --delta=dist=3 --lzma2=pb=0 -fk ${1/.png/.bmp}
    xz --delta=dist=3 --lzma2=pb=0 -fk ${1/.png/.tiff}
  fi
  stat -c '%n %s' $1 ${1/.png/.qoi.xz} $pnm.xz ${1/.png/.bmp.xz} ${1/.png/.tiff.xz}
}

f $1
```

```sh
cd /tmp/dc-img/images/
ls -1 **/*.png | rush ~/tmp/b.sh '"{}"'
ls -1 **/*.uncompressed.png | rush 'xz -fk --lzma2=pb=0 "{}"'

ruby -e 'puts "directory\t.png\t.png.xz\t.qoi.xz\t.bmp.xz\t.tiff.xz\t.p[ap]m.xz"; Dir.glob("*").each{|dir| next unless File.directory? dir;
  png=pngxz=qoi=bmp=pnm=tiff=0; Dir.glob("#{dir}/*.qoi.xz").each{|f|
  png+=File.size(f.sub(/\.qoi.xz/,".png"));
  pngxz+=File.size(f.sub(/\.qoi.xz/,".uncompressed.png.xz"));
  qoi+=File.size(f); bmp+=File.size(f.sub(/\.qoi/,".bmp")); ppm=f.sub(/\.qoi/,".ppm");  pnm+=File.exists?(ppm) ? File.size(ppm) : File.size(f.sub(/\.qoi/,".pam")); tiff+=File.size(f.sub(/\.qoi/,".tiff"));
};
  puts "#{dir}\t#{png}\t#{pngxz}\t#{qoi}\t#{bmp}\t#{tiff}\t#{pnm}"
}'
```

While DEFLATE-compressed PNG files can hardly be further compressed, we can convert these PNG files to uncompressed ones then apply xz.
The `.png.xz` results below do not apply a filter, and the files are generally larger than `.qoi.xz`.

| directory       | .png      | .png.xz   | .qoi.xz   | .bmp.xz   | .tiff.xz  | .p[ap]m.xz |
| --------------- | --------- | --------- | --------- | --------- | --------- | ---------- |
| icon_512        | 11154424  | 7861652   | 7476640   | 8042032   | 8064476   | 8039192    |
| icon_64         | 828119    | 750836    | 708480    | 730472    | 757760    | 735296     |
| photo_kodak     | 15394305  | 14464504  | 12902852  | 13612440  | 13616140  | 13610844   |
| photo_tecnick   | 237834256 | 254803292 | 213268188 | 210591724 | 210508596 | 210468412  |
| photo_wikipedia | 88339751  | 100449996 | 86679696  | 86380124  | 86274480  | 86241296   |
| pngimg          | 229608249 | 134233476 | 193382668 | 186389368 | 186654256 | 186420564  |
| screenshot_game | 266238855 | 237976536 | 218915316 | 216626004 | 216847500 | 216765748  |
| screenshot_web  | 40272678  | 24690360  | 21321460  | 21458496  | 21532360  | 21533432   |
| textures_photo  | 37854634  | 36393340  | 28967008  | 30054968  | 30064236  | 30059784   |
| textures_pk     | 43523493  | 40868036  | 54117600  | 41990596  | 40632916  | 46695172   |
| textures_pk01   | 18946769  | 15734348  | 14950836  | 14835648  | 14853420  | 14839312   |
| textures_pk02   | 102962935 | 86037000  | 82279000  | 79374112  | 79348768  | 79336276   |
| textures_plants | 51765329  | 53044044  | 43681548  | 44913260  | 45021996  | 45048652   |

While compressing QOI with XZ (`.qoi.xz`) can achieve good results, using a delta filter directly on the uncompressed BMP format (`.bmp.xz`) can sometimes lead to even smaller files.
(TIFF and PPM/PAM, when compressed, can achieve similar file sizes to `.bmp.xz`.)
This suggests that QOI is probably not better than a plain delta filter.

It's important to note that uncompressed BMP/TIFF files are huge.
This can be problematic if the decompressed data can't be streamed directly into the program's internal structures.
In such cases, a large temporary buffer would be needed, wasting memory.

## Drop LZ match finders

`QOI_OP_INDEX` essentially does length-1 LZ77 using a conceptual window that contains 64 unique pixels.
When further compressed, another match finder seems to help very little.

Lasse Collin mentioned that the LZ layer cannot be disabled but it can be made really weak using `xz --lzma2=dict=4KiB,mode=fast,nice=2,mf=hc3,depth=1`.
Let's try it.

```
% =time -f '%e' xz -fk Prune_video_game_screenshot_2.qoi && stat -c %s Prune_video_game_screenshot_2.qoi.xz
0.76
2462360
% =time -f '%e' xz --lzma2=dict=4KiB,mode=fast,nice=2,mf=hc3,depth=1 -fk Prune_video_game_screenshot_2.qoi && stat -c %s Prune_video_game_screenshot_2.qoi.xz
0.27
2526664
```

Indeed, weakening the LZ layer improves compression speed signicantly.
Now, let's test all benchmark images.

```
% cat ~/tmp/qoi-weak-xz.sh
#!/bin/zsh
/tmp/p/qoi/qoiconv $1 ${1/.png/.qoi}
xz --lzma2=pb=0 -fk ${1/.png/.qoi}
xz --lzma2=dict=4KiB,mode=fast,nice=2,mf=hc3,depth=1 -c ${1/.png/.qoi} > ${1/.png/.qoi.weak-lz.xz}
% cd /tmp/dc-img/images
% ls -1 **/*.png | rush ~/tmp/qoi-weak-xz.sh '"{}"'

ruby -e 'puts "directory\tstrong\tweak\tincrease"; Dir.glob("*").each{|dir| next unless File.directory? dir;
  strong=weak=0; Dir.glob("#{dir}/*.qoi.weak-lz.xz").each{|f| weak+=File.size(f); strong+=File.size(f.sub(/\.weak-lz/,""));};
  puts "#{dir}\t#{strong}\t#{weak}\t#{(100.0*weak/strong-100).round(2)}%"
}'
```

| directory       | strong    | weak      | increase |
| --------------- | --------- | --------- | -------- |
| icon_512        | 7476640   | 8629900   | 15.42%   |
| icon_64         | 708480    | 735036    | 3.75%    |
| photo_kodak     | 12902852  | 13464072  | 4.35%    |
| photo_tecnick   | 213268188 | 217460392 | 1.97%    |
| photo_wikipedia | 86679696  | 88609716  | 2.23%    |
| pngimg          | 193382668 | 206679224 | 6.88%    |
| screenshot_game | 218915316 | 234889060 | 7.3%     |
| screenshot_web  | 21321460  | 24820020  | 16.41%   |
| textures_photo  | 28967008  | 31249492  | 7.88%    |
| textures_pk     | 54117600  | 57956168  | 7.09%    |
| textures_pk01   | 14950836  | 15749556  | 5.34%    |
| textures_pk02   | 82279000  | 87747576  | 6.65%    |
| textures_plants | 43681548  | 45494084  | 4.15%    |

This size increase is small for certain directories but quite large for the others.
For the directories with small size increases, relying purely on delta coding and a fast entropy encoder will give a strong competitor.

[The Dark Horse of the Image Codec World: Near-Lossless Image Formats Using Ultra-Fast LZ Codecs](https://richg42.blogspot.com/2023/04/a-dead-simple-lossless-or-lossy-lz4.html) remarks that fast LZ can make strong contenders.

## PNG

The PNG International Standard defines the compression method 0 as DEFLATE with a sliding window of at most 32768 bytes.
Technically new compression methods can be defined, but that would break compatibility of existing decoders and stakeholders would just resort to new image formats.
However, it would be a nice experiment to check that after the compression part is improved, how PNG compares with newer image formats.
