---
title: 在主引导记录（MBR）中实现 Hello world!
tags:
  - 主引导记录（MBR）
  - Nasm 汇编
categories:
  - 操作系统（OS）
abbrlink: 1799537993
date: 2023-09-08 05:01:57
---

我们平时所写的程序都依赖于操作系统和各种库，而我们接下来将要实现一个裸机（Bare Metal）程序——在主引导记录（MBR）中实现 Hello world!

<!--more-->

对应的 Nasm 汇编源码如下：

```x86asm
; file: mbr.asm
; function: hello world!

  ; 调用显示服务中断（int 0x10）来设置显示模式（0x00），从而清空屏幕
  mov ax, 3                 ; al -> 显示模式
  int 0x10

  ; 调用显示服务中断（int 0x10）来打印字符串（0x13）
  mov ax, 0x1301            ; al -> 显示模式
  mov bx, 7                 ; bh -> 页码；bh -> 字符属性
  mov cx, 12                ; 字符串长度
  mov dx, 0                 ; 屏幕位置（dh -> y；dl -> x）
  mov bp, msg + 0x7C00      ; 字符串在内存的起始地址（es:bp）
  int 0x10

  jmp $                     ; 跳转到当前位置，实际上形成了死循环，防止 CPU 继续往下执行代码

msg:                        ; msg 是一个标号，表示当前的汇编地址
  db 'Hello world!'         ; 伪指令 db 生成一个字节的数据，对于字符串，则会生成对应的 ASCII 值

  times 510 - ($ - $$) db 0 ; 伪指令 times 表示重复执行一个操作，$$ 表示当前节（section）的起始位置

  db 0x55, 0xAA             ; MBR 魔数
```

这段代码调用了两次中断，一次是 0x10/0x00 用于设置显示屏幕；另一次是 0x10/0x13 用于打印字符串。此外，我们还要
