---
layout: post
title: Real World CTF 2018 rwext5命題報告
author: MaskRay
tags: [ext4, ctf, forensics]
---

> 史上最强国际顶尖战队集结5大洲、15个国家地区、20支战队。近5年共计获得150次CTF全球冠军。100名最强CTF选手，48小时不间断赛制。12月1-2日，中国郑州、长亭科技，首届Real World国际CTF网络安全大赛，线下总决赛，大幕开启！

我也好想去鄭州啊😭

In November, I created a forensics/reversing challenge "rwext5" for Real World CTF 2018 Finals.

<!-- more -->

kelwya找我來出題，他說可以forensics。因爲很久沒有做題，在ccls-fringe出完後感覺又陷入了出題困難的尷尬狀態QAQ。因爲我對文件系統有一點興趣，就盤算着是否找個成熟的fs，稍微修補下使得和正常行爲不同，然後製作一個磁盤格式化爲該fs，塞入藏匿的flag和其他輔助解密的線索。

It is about the hypothetical filesystem "ext5", which is obviously based on ext4.
Actually, I patched lwext4 a bit to make it different from a standard ext4 filesystem. Contestants were supposed to reverse engineer `rwext5-mkfs` and `rwext5-import` and find the nuance.

```
% tar tf rwext5.tar.xz
rwext5-import
rwext5-mkfs
rwext5.img
```

After successful deobfuscation, the following files can be found in `rwext5.img`:

```
% tree -a
.
├── 0001-Raise-cmake_minimum_required-2.8-3.4.patch
├── 0002-Fix-const-const-warning.patch
├── 0003-Don-t-copy-include.patch
├── 0004-Fix-ext4_mkfs_info-feat_-ro_compat-compat-incompat.patch
├── 0005-Fix-jbd_commit_header-chksum_-type-size.patch
├── bin
│   ├── clang
│   ├── ld.lld
│   └── llvm-objcopy
├── .config
│   ├── Code
│   │   └── User
│   │       └── settings.json
│   ├── doom
│   │   ├── init.el
│   │   └── modules
│   │       └── private
│   │           └── my-cc
│   │               ├── autoload.el
│   │               ├── flag.0.o
│   │               ├── flag.1.o
│   │               ├── config.el
│   │               └── packages.el
│   └── nvim
│       └── init.vim
├── lwext4
│   ├── .ccls-cache
│   │   └── .keep
│   └── compile_commands.json
└── README
```

## flag、hint and other files

The image includes some form of the flag file, a copy of lwext4 (as a hint that the two distributed tools `rwext5-mkfs` `rwext5-import` should be reversed and compared with standard ones).
It would be interesting to put some other stuff (also served as market promotion purpose for my project and the LSP ecosystem :)

鏡像裏除了`flag`文件和一整套lwext4源碼`lwext4`(提示需要改什麼)，還得有點其他東西起到迷惑作用，因此我塞了下面這些東西：

> 另外也想給自己的項目打廣告，萌生了從[ccls](https://github.com/MaskRay/ccls)的cache弄一道forensics的想法，因爲惰性拖了兩個星期沒有動手。之前一次LeetCode Weekly前在和一個學弟聊天，就想把他的id嵌入了flag。LeetCode第一題Leaf-Similar Trees沒有叫成same-fringe，所以我的題目就帶上fringe字樣“科普”一下吧。

* `lwext4/.ccls-cache`: `.ccls-cache` is the default cache directory of ccls.
* 5 `.patch` files have been uploaded (and merged) after the contest <https://github.com/gkostka/lwext4/pull/43>~47
* `.config/nvim/init.vim`: language clients of Vim/NeoVim：ale, coc.nvim, LanguageClient-neovim, vim-lsp。
* `.config/Code/User/settings.json`: VSCode configuration of ccls

Create a file `flag` of 1111 blocks (512 bytes per block) filled with random bytes and place the plaintext flag in the 555th block. Encrypt it with `openssl enc -aes-128-cbc -pass pass:filesystem -in flag.plain -out flag`.

至於`bin/{clang,lld,llvm-objcopy}`，因爲我對這些工具分別有50+,90+,10+ commits(努力學習📚)，而且覺得用他們繼續藏匿🏁挺好玩就加進去啦。

```zsh
% cat clang
I am not used in the challenge, but I empowers language servers.
% cat ld.lld
I can concatenate sections.
% cat llvm-objcopy
I can dump sections.
```

```makefile
$(flag0): flag
	head -c 1000000 flag > $@
	llvm-objcopy -I binary -B powerpc:common64 $@
	llvm-objcopy --rename-section=.data=.openssl_aes-128-cbc_pass:filesystem $@
	echo empty > empty; llvm-objcopy --add-section=.turkey_for_thanksgiving=empty $@; rm empty

$(flag1): flag
	tail -c +1000001 flag > $@
	llvm-objcopy -I binary -B powerpc:common64 $@
	llvm-objcopy --rename-section=.data=.openssl_aes-128-cbc_pass:filesystem $@
	echo empty > empty; llvm-objcopy --add-section=.gift_for_christmas=empty $@; rm empty
```

`flag` encrypted by `openssl` was split by llvm-objcopy into two halves: `flag.0.o` and `flag.1.o`.

openssl加密後的`flag`拆成兩部分用llvm-objcopy簡單處理得到`flag.0.o`和`flag.1.o`。這兩個文件放在`.config/doom/modules/private/my-cc/`(`.config/doom/`是放置Doom Emacs個人定製的目錄)。

## lwext4 obfuscation

I modified `ext4_ext_find_goal` (lwext4 uses a first-fit algorithm to find the first free block) so that it would return a random block, otherwise the extent block tended to be continuous and the contestants wouldn't need to decrypt the metadata to recover the two object files.

```diff
diff --git a/src/ext4_extent.c b/src/ext4_extent.c
index abac59b..791f8a7 100644
--- a/src/ext4_extent.c
+++ b/src/ext4_extent.c
@@ -26,6 +26,7 @@
 #include <string.h>
 #include <inttypes.h>
 #include <stddef.h>
+#include <sys/time.h>

 #if CONFIG_EXTENTS_ENABLE
 /*
@@ -593,6 +594,15 @@ static ext4_fsblk_t ext4_ext_find_goal(struct ext4_inode_ref *inode_ref,
 				       struct ext4_extent_path *path,
 				       ext4_lblk_t block)
 {
+	// CTF
+	static int z;
+	if (!z) {
+		struct timeval t;
+		gettimeofday(&t, NULL);
+		srand(t.tv_sec ^ t.tv_usec / 1000);
+		z = 1;
+	}
+	return rand() % ext4_sb_get_blocks_cnt(&inode_ref->fs->sb);
 	if (path) {
 		uint32_t depth = path->depth;
 		struct ext4_extent *ex;
```

The struct members of superblock inode are reordered a bit:

```diff
diff --git a/include/ext4_types.h b/include/ext4_types.h
index c9cdd34..a4c5050 100644
--- a/include/ext4_types.h
+++ b/include/ext4_types.h
@@ -78,9 +78,10 @@ struct ext4_sblock {
 	uint32_t first_data_block;	 /* First Data Block */
 	uint32_t log_block_size;	   /* Block size */
 	uint32_t log_cluster_size;	 /* Obsoleted fragment size */
-	uint32_t blocks_per_group;	 /* Number of blocks per group */
-	uint32_t frags_per_group;	  /* Obsoleted fragments per group */
+	// CTF swap
 	uint32_t inodes_per_group;	 /* Number of inodes per group */
+	uint32_t frags_per_group;	  /* Obsoleted fragments per group */
+	uint32_t blocks_per_group;	 /* Number of blocks per group */
 	uint32_t mount_time;		   /* Mount time */
 	uint32_t write_time;		   /* Write time */
 	uint16_t mount_count;		   /* Mount count */
@@ -245,6 +246,8 @@ struct ext4_sblock {
 #define EXT4_FINCOM_MMP 0x0100
 #define EXT4_FINCOM_FLEX_BG 0x0200
 #define EXT4_FINCOM_EA_INODE 0x0400	 /* EA in inode */
+// CTF
+#define EXT4_FINCOM_CTF 0x800
 #define EXT4_FINCOM_DIRDATA 0x1000	  /* data in dirent */
 #define EXT4_FINCOM_BG_USE_META_CSUM 0x2000 /* use crc32c for bg */
 #define EXT4_FINCOM_LARGEDIR 0x4000	 /* >2GB or 3-lvl htree */
@@ -281,7 +284,7 @@ struct ext4_sblock {
 #define EXT4_SUPPORTED_FINCOM                              \
 	(EXT4_FINCOM_FILETYPE | EXT4_FINCOM_META_BG |      \
 	 EXT4_FINCOM_EXTENTS | EXT4_FINCOM_FLEX_BG |       \
-	 EXT4_FINCOM_64BIT)
+	 EXT4_FINCOM_64BIT | EXT4_FINCOM_CTF)

 #define EXT4_SUPPORTED_FRO_COM                             \
 	(EXT4_FRO_COM_SPARSE_SUPER |                       \
@@ -371,13 +374,14 @@ struct ext4_bgroup {
  * Structure of an inode on the disk
  */
 struct ext4_inode {
-	uint16_t mode;		    /* File mode */
 	uint16_t uid;		    /* Low 16 bits of owner uid */
-	uint32_t size_lo;	   /* Size in bytes */
+	uint16_t mode;		    /* File mode */
+	// CTF
 	uint32_t access_time;       /* Access time */
 	uint32_t change_inode_time; /* I-node change time */
 	uint32_t modification_time; /* Modification time */
 	uint32_t deletion_time;     /* Deletion time */
+	uint32_t size_lo;	   /* Size in bytes */
 	uint16_t gid;		    /* Low 16 bits of group id */
 	uint16_t links_count;       /* Links count */
 	uint32_t blocks_count_lo;   /* Blocks count */
diff --git a/src/ext4_extent.c b/src/ext4_extent.c
index 791f8a7..375a2f6 100644
--- a/src/ext4_extent.c
+++ b/src/ext4_extent.c
@@ -92,10 +92,11 @@ struct ext4_extent_tail
  * It's used at the bottom of the tree.
  */
 struct ext4_extent {
-    uint32_t first_block; /* First logical block extent covers */
-    uint16_t block_count; /* Number of blocks covered by extent */
-    uint16_t start_hi;    /* High 16 bits of physical block */
-    uint32_t start_lo;    /* Low 32 bits of physical block */
+	// CTF
+	uint16_t block_count; /* Number of blocks covered by extent */
+	uint16_t start_hi;    /* High 16 bits of physical block */
+	uint32_t start_lo;    /* Low 32 bits of physical block */
+	uint32_t first_block; /* First logical block extent covers */
 };

 /*
```

Two tools compiled from the obfuscated lwext4 are provided:

* `rwext5-mkfs`: used to create a ext5 filesystem
* `rwext5-import`: used to import files into the filesystem

Fill `rwext5.img` with `dd if=/dev/urandom`, run `rwext5-mkfs`, and then populate it with `flag.0.o`, `flag.1.o`, lwext4 and other files. The final `rwext5.img` was distributed among teams.

By reversing the two programs contestants can figure out how struct members are reordered. They may write an export tool (I wrote a `rwext5-export`) to copy the two object files out or assemble extent blocks by hand.
Both object files have the section with a weird name ".openssl_aes-128-cbc_pass:filesystem". A linker concatenates the section contents and llvm-objcopy may be used to dump the section. It is straightforward to decrypt the contents with openssl and get the plaintext flag:

```
The flag is rwctf{ext4 adds extent tree to ext3}
```

逆向以上兩個程序即可發現fields打亂的方式。可以自行編寫導出工具(我寫了一個`rwext5-export`)或者手工查找組成`flag.0.o` `flag.1.o`的extents，恢復出來後用lld拼接，用llvm-objcopy dump被openssl加密的section，按照section名提示的密鑰解密後讀出明文flag。

```makefile
test: bin/rwext5-cat $(image) flag.plain
        mkdir -p test
        bin/rwext5-cat $(image) ctf/README
        bin/rwext5-cat $(image) ctf/.config/doom/modules/private/my-cc/flag.0.o > test/flag.0.o
        bin/rwext5-cat $(image) ctf/.config/doom/modules/private/my-cc/flag.1.o > test/flag.1.o
        ld.lld test/flag.0.o test/flag.1.o -o /dev/stdout | llvm-objcopy --dump-section=.openssl_aes-128-cbc_pass:filesystem=/tmp/flag - /dev/null
        openssl enc -d -aes-128-cbc -pass pass:filesystem -in /tmp/flag | cmp - flag.plain
```

## Code

<https://github.com/MaskRay/RealWorldCTF-2018-ccls-fringe-and-rwext5>

Thanks to gkostka for the lwext4 project (I should really learn more about ext4), and ngkaho1234 for the help to modify lwext4!
