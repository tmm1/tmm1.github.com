---
layout: post
title: "Ruby 2.1: Process.setproctitle()"
tagline: 'improve visibility with customized proclines, now easier than ever'
tags: [ruby21, yearoftheprocline]
---

Custom proclines are a great way to add visibility into your code's execution. The various BSDs have long shipped a [`setproctitle(3)`](http://www.freebsd.org/cgi/man.cgi?query=setproctitle&sektion=3) syscall for this purpose. At GitHub, we use this technique extensively in our unicorns, resques and even our git processes:

```
$ ps ax | grep git:
32326  git:  upload-pack  AOKP/kernel_samsung_smdk4412  [173.x.x.x over git]  clone: create_pack_file: 75.27 MiB @ 102.00 KiB/s
32327  git: pack-objects  AOKP/kernel_samsung_smdk4412  [173.x.x.x over git]  writing: 9% (203294/2087544)
 2671  git:  upload-pack  CyanogenMod/android_libcore   [87.x.x.x over git]  clone: create_pack_file: 31.26 MiB @ 31.00 KiB/s
 2672  git: pack-objects  CyanogenMod/android_libcore   [87.x.x.x over git]  writing: 93% (90049/96410)
```

Linux never gained a specialized syscall, but still supports custom `/proc/pid/cmdline` via [one weird trick](https://github.com/torvalds/linux/blob/f5835372ebedf26847c2b9e193284075cc9c1f7f/fs/proc/base.c#L220-L222). The hack relies on the fact that [`execve(2)`](http://man7.org/linux/man-pages/man2/execve.2.html) allocates a contiguous piece of memory to pass `argv` and `envp` into a new process. When the null byte separating the two values is overwritten, the kernel assumes you've customized your cmdline and continues to read into the envp buffer. This means your procline can grow to contain more information as long as there's space in the envp buffer. (Environment variables are copied out of the original envp buffer, so overwriting this area of memory only affects `/proc/pid/environ`).

Starting with Ruby 2.1 `Process.setproctitle` is available as a complementary method to `$0=`, and can be used to customize proclines without affecting `$0`.
