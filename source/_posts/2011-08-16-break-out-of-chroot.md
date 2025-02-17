---
layout: post
title: 脫離chroot的枷鎖
author: MaskRay
tags: [linux, chroot]
---

2015年8月更新

《The Linux Programming Interface》的Chapter 18 Directories and Links提到chroot jail有幾個注意點：

- `chroot()`不改變工作目錄。因此通常在調用`chroot()`之後會緊跟`chdir("/")`，把工作目錄設定到新的root；否則仍可使用工作目錄訪問jail外的文件。只是之後訪問jail外的文件不可以用絕對路徑了，因爲root目錄還在jail裏。
- 可以使用jail外文件的文件描述符脫離jail，使用`fchdir()`即可改變工作目錄到jail外。如果是特權進程的話(精確地，指擁有`CAP_SYS_CHROOT`權限)，還可以在`fchdir()`後使用`chroot(".")`以把root目錄設置到jail外。倘若多`chdir("..")`幾次，可以回到原先的root目錄。
- Unix domain socket提供了進程間傳遞文件描述符的方法。限定在chroot jail內的進程可以從外部獲取文件描述符，之後即可`fchdir()`使工作目錄脫離jail。

下面的例子展示如何使用jail外的文件描述符脫離jail：

```c
#include <fcntl.h>
#include <stdlib.h>
#include <sys/stat.h>
#include <sys/types.h>
#include <unistd.h>

int main(void)
{
  int fd = open(".", O_RDONLY), i; // jail外的文件描述符，供之後脫離
  mkdir("tempdir", 0755);
  if (fd == -1) return 1;
  if (chroot("tempdir") == -1) return 1; // chroot
  if (fchdir(fd) == -1) return 1; // 脫離
  for (i = 0; i < 1024; i++) // 回到原先的root目錄。這裏不能使用絕對路徑`/`，只能逐步上移
      chdir("..");
  if (chroot(".") == -1) return 1; // 若是特權進程，則可進一步，把root設回去；不是的話也足以訪問jail外的文件
  system("ls");
  return 0;
}
```
