# GIT tips

Useful skills but not used day-to-day

#### git blame 查找"元凶"

`blame` Show what revision and author last modified each line of a file.

 通常只是这样用

```
git blame -L <line number 1>,<line number 2> [--] <file>
```

或者不通过 -L 指定行号.  问题是，有时某行代码上一次 modification 可能只是简单 Macro 替换, 这时通过上面的 blame 用法是无法找到引入这行代码的原始 commit 的。但 `blame` 允许指定 revision range, 也就是说，可以通过指定 revision range 的方式来找到该行代码的 second last, third last modification, 直到引入该行代码的 commit.

举例说明用法. arch/x86/kernel/head_64.S 有如下 code piece:

```
51         .text
52         __HEAD
53         .code64
54         .globl startup_64
55 startup_64:
```

想找到 52 行的来历。直接 blame 的结果是

> 4ae59b916d269 arch/x86/kernel/head_64.S (Tim Abbott                2009-09-16 16:44:28 -0400  52)       __HEAD

`git show 4ae59b916d269` 发现只是 macro 替换。这时就要用到 blame 的 revision range 来帮忙了。

```
git blame 4ae59b916d269^ arch/x86/kernel/head_64.S
```

意思是，从 commit 4ae59b916d269 的 parent 开始，展示每一行代码的最后一次 modification 信息，结果是:

> 92417df076f75 arch/x86_64/kernel/head.S (Andi Kleen                2007-07-22 11:12:45 +0200  43)       .section .text.head

查勘 commit 92417df076f75 发现只是简单的改了文本，而且该 commit 发生时候，文件名叫 arch/x86_64/kernel/head.S, 所以需要看一下再上一次的 modification:

```
git blame 92417df076f75^ arch/x86_64/kernel/head.S
```

得到:

> eaeae0cc985fa (Arjan van de Ven       2006-03-25 16:30:28 +0100  28)    .section .bootstrap.text

查看 commit eaeae0cc985fa 发现，它就是引入该行代码的元凶

参考:

1. "Revision Selection" of [progit2](https://github.com/progit/progit2)
2. `man 7 gitrevisions`: 如何为 git 命令指定某个 commit 或者 commit range 
3. `git help blame`: blame 时如何指定 revision range