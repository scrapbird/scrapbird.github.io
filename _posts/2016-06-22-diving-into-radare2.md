---
layout: post
title: Diving Into Radare2
---

I've been improving my reverse engineering skills lately and decided to have a go at using [radare2](http://radare2.org) after a recommendation on an IRC channel I frequent. After reading through some blog posts and the [radare2 book](https://www.gitbook.com/book/radare/radare2book/details) (which is awesome, by the way) I decided to reverse a small shellcode using only r2 to see how easy it would be to get used to.

First I picked up a random shellcode from exploit-db and settled on [this one](https://exploit-db.com/exploits/39869/) which promised to contain some XOR encoding, which I figured would give me some semi-complicated operations to carry out using only r2. I would have to manually XOR some bytes and decompile the output to read the shellcode's final payload, at least.

First lets compile the following payload:

{% highlight c %}
char shellcode[]="\xeb\x1d\x5e\x48\x31\xc9\xb1\x31\x99\xb2\x90\x48\x31\xc0\x8a"
                 "\x06\x30\xd0\x48\xff\xcc\x88\x04\x24\x48\xff\xc6\xe2\xee\xff"
                 "\xd4\xe8\xde\xff\xff\xff\x95\x9f\xab\x50\x13\xd8\x76\x19\xd8"
                 "\xc7\xc6\xc2\x76\x19\xd8\xf9\xbd\xb4\x94\x57\xf6\xc0\xc0\x77"
                 "\x19\xd8\xf8\xe3\xbf\xfe\x94\xb4\xd4\x57\xf9\xf2\xbf\xbf\xb4"
                 "\x94\x57\xc0\xc0\x42\xa1\xd8\x50\xa1";
int main(int i,char *a[])
{
    (* (int(*)()) shellcode)();
}
{% endhighlight %}

With the following simple command:

{% highlight plaintext %}
➜  ~ gcc --version
gcc (Debian 4.7.2-5) 4.7.2
Copyright (C) 2012 Free Software Foundation, Inc.
This is free software; see the source for copying conditions.  There is NO
warranty; not even for MERCHANTABILITY or FITNESS FOR A PARTICULAR PURPOSE.

➜  ~ gcc shellcode.c -fno-stack-protector -z execstack -o shellcode
➜  ~
{% endhighlight %}

Then open it with r2:

{% highlight nasm %}
➜  ~ r2 shellcode
 -- See you in shell
[0x7fef42cacaf0]>
{% endhighlight %}

As can be seen by the r2 prompt, we are currently positioned at offset **0x7fef42cacaf0**. Lets analyze the whole file and seek to the main subroutine:

{% highlight nasm %}
[0x7fef42cacaf0]> aaa
[x] Analyze all flags starting with sym. and entry0 (aa)
[Cannot determine xref search boundariesr references (aar)
[x] Analyze len bytes of instructions for references (aar)
[Oops invalid rangen calls (aac)
[x] Analyze function calls (aac)
[ ] [*] Use -AA or aaaa to perform additional experimental analysis.
[x] Constructing a function name for fcn.* and sym.func.* functions (aan))
[0x7fef42cacaf0]> s main
[0x004004ac]> pdf
            ;-- main:
/ (fcn) sym.main 29
|           ; var int local_10h @ rbp-0x10
|           ; var int local_4h @ rbp-0x4
|           ; DATA XREF from 0x004003bd (entry0)
|           0x004004ac      55             push rbp
|           0x004004ad      4889e5         mov rbp, rsp
|           0x004004b0      4883ec10       sub rsp, 0x10
|           0x004004b4      897dfc         mov dword [rbp - local_4h], edi
|           0x004004b7      488975f0       mov qword [rbp - local_10h], rsi
|           0x004004bb      baa0086000     mov edx, obj.shellcode
|           0x004004c0      b800000000     mov eax, 0
|           0x004004c5      ffd2           call rdx
|           0x004004c7      c9             leave
\           0x004004c8      c3             ret
[0x004004ac]>
{% endhighlight %}

We can see that r2 has analyzed our bin and named the shellcode subroutine for us and that the program is putting the address of the shellcode into `edx`, then calling it. Lets take a look at the shellcode now.

{% highlight nasm %}
[0x004004ac]> s obj.shellcode
[0x006008a0]> pdf
Cannot find function at 0x006008a0
[0x006008a0]>
{% endhighlight %}

Hmm.. looks like r2 hasn't recognized this part of the code as a function, lets jump into visual disassembly mode by executing `V` to get into visual mode and pressing `p` to cycle to the disassembly view. You may need to jump back to this position by pressing `o` and then typing `obj.shellcode`, this is because when you first enter visual mode r2 will seek to the current instruction pointer.

{% highlight nasm %}
[0x006008a0 256 ./shellcode]> pd $r @ obj.shellcode
       ,.-> ;-- shellcode:
            ; DATA XREF from 0x004004bb (sym.main)
       ,.-> 0x006008a0      eb1d           jmp 0x6008bf                ;[1]
       ||   ;-- str._H1__1:
       ||   0x006008a2     .string "^H1\x,9\x+11" ; len=7
       ||   0x006008a9      b290           mov dl, 0x90                ; 144
      .---> 0x006008ab      4831c0         xor rax, rax
      |||   0x006008ae      8a06           mov al, byte [rsi]
      |||   0x006008b0      30d0           xor al, dl
      |||   0x006008b2      48ffcc         dec rsp
      |||   0x006008b5      880424         mov byte [rsp], al
      |||   0x006008b8      48ffc6         inc rsi
      `===< 0x006008bb      e2ee           loop 0x6008ab               ;[2]
       ||   0x006008bd      ffd4           call rsp
       `--> 0x006008bf      e8deffffff     call str._H1__1             ;[3]
        |   0x006008c4      95             xchg eax, ebp
        |   0x006008c5      9f             lahf
        |   0x006008c6      ab             stosd dword [rdi], eax
        |   0x006008c7      50             push rax
        |   0x006008c8      13d8           adc ebx, eax
       ,==< 0x006008ca      7619           jbe 0x6008e5                ;[4]
       ||   0x006008cc      d8c7           fadd st(7)
       ||   0x006008ce      c6c276         mov dl, 0x76                ; 'v' ; 118
       ||   0x006008d1      19d8           sbb eax, ebx
       ||   0x006008d3      f9             stc
       ||   0x006008d4      bdb49457f6     mov ebp, 0xf65794b4
       ||   0x006008d9      c0c077         rol al, 0x77
       ||   0x006008dc      19d8           sbb eax, ebx
       ||   0x006008de      f8             clc
       |`=< 0x006008df      e3bf           jrcxz obj.shellcode         ;[5]
       |    0x006008e1      fe             invalid
       |    0x006008e2      94             xchg eax, esp
       |    0x006008e3      b4d4           mov ah, 0xd4                ; 212
       `--> 0x006008e5      57             push rdi
            0x006008e6      f9             stc
            0x006008e7      f2bfbfb49457   mov edi, 0x5794b4bf
            0x006008ed      c0c042         rol al, 0x42
{% endhighlight %}

After a quick look at this we can see that the shellcode jumps down to **0x6008bf** which is an instruction to call.. a string? Looks like r2 has mistaken that line of code for data, but that's okay because we can fix it. Lets scroll down to that section by pressing `j` once and mark it as code by pressing `dc` (a good mnemonic for this is **d**efine **c**ode). After doing that we get the following:

{% highlight nasm %}
[0x006008a0 325 ./shellcode]> pd $r @ obj.shellcode
        ,=< ;-- shellcode:
            ; DATA XREF from 0x004004bb (sym.main)
        ,=< 0x006008a0      eb1d           jmp 0x6008bf                ;[1]
        |   ;-- str._H1__1:
        |   0x006008a2      5e             pop rsi
        |   0x006008a3      4831c9         xor rcx, rcx
        |   0x006008a6      b131           mov cl, 0x31                ; '1' ; 49
        |   0x006008a8      99             cdq
        |   0x006008a9      b290           mov dl, 0x90                ; 144
        |   0x006008ab      4831c0         xor rax, rax
        |   0x006008ae      8a06           mov al, byte [rsi]
        |   0x006008b0      30d0           xor al, dl
        |   0x006008b2      48ffcc         dec rsp
        |   0x006008b5      880424         mov byte [rsp], al
        |   0x006008b8      48ffc6         inc rsi
        |   0x006008bb      e2ee           loop 0x6008ab               ;[2]
        |   0x006008bd      ffd4           call rsp
        `-> 0x006008bf      e8deffffff     call str._H1__1             ;[3]
            0x006008c4      95             xchg eax, ebp
            0x006008c5      9f             lahf
            0x006008c6      ab             stosd dword [rdi], eax
            0x006008c7      50             push rax
            0x006008c8      13d8           adc ebx, eax
            0x006008ca      7619           jbe 0x6008e5                ;[4]
            0x006008cc      d8c7           fadd st(7)
            0x006008ce      c6c276         mov dl, 0x76                ; 'v' ; 118
            0x006008d1      19d8           sbb eax, ebx
            0x006008d3      f9             stc
            0x006008d4      bdb49457f6     mov ebp, 0xf65794b4
            0x006008d9      c0c077         rol al, 0x77
            0x006008dc      19d8           sbb eax, ebx
            0x006008de      f8             clc
            0x006008df      e3bf           jrcxz obj.shellcode         ;[5]
            0x006008e1      fe             invalid
            0x006008e2      94             xchg eax, esp
            0x006008e3      b4d4           mov ah, 0xd4                ; 212
            0x006008e5      57             push rdi
            0x006008e6      f9             stc
            0x006008e7      f2bfbfb49457   mov edi, 0x5794b4bf
            0x006008ed      c0c042         rol al, 0x42
{% endhighlight %}

Much better. Now lets take a look at what this code is doing, shall we?

We can see that after the call instruction from **0x6008bf** is executed we are popping an address from the stack into `rsi`. The call instruction puts the next instruction's address onto the stack, which means `rsi` is now pointing to **0x006008c4**, which looks like a lot of junk code. Remember that this is an XOR'd shellcode so this is not surprising.

Next the code will zero out `rcx` by XORing it with itself and we set the counter (`cl`) to **0x31** and data (`dl`) to **0x90** and then zero out `rax`. This is all to set up a loop that will loop through **0x31** bytes of data starting at `rsi`, XORing each byte with the value **0x90** and pushing it onto the stack.

At **0x006008bd** the execution is passed to the newly decoded instructions at `rsp` (the top of our stack). We need to somehow decode this ourselves so we can have a look at it, keeping in mind that the code was pushed onto the stack backwards.

We could take advantage of r2's write mode by turning it on with `e io.cache = true` and then XOR the code with the `wox` command and analyze the output, but then we would also need to reverse the byte order (as the data is pushed onto the stack it will be backwards if we do it in the correct order) and we don't really want to complicate things. For this we should take advantage of r2's debugging abilities.

Lets quit r2 (press `q` until it closes completely) and reopen our shellcode in debug mode:

{% highlight bash %}
r2 -d shellcode
{% endhighlight %}

Then enter `aaa` to analyze the file again and then seek to our obj.shellcode flag and go through the same process of defining that string as code from visual mode. We should now be looking at the same screen as before.

Like in vim we can enter a command mode without exiting visual mode in r2 by pressing `:`. From here we can execute normal r2 commands without needing to jump back to the r2 shell. Because we are responsible people and we never run code that we haven't read yet we will set a break point at the instruction to call out to the decoded instructions pointed to by `rsp` at address **0x006008bd** by entering the following command:

{% highlight plaintext %}
db 0x006008bd
{% endhighlight %}

Now lets run the program by entering the `dc` command (**d**ebugger **c**ontinue) and then seek to the location that `rsp` currently points to with `s rsp`. We should now be looking at the following:

{% highlight nasm %}
[0x7ffcef34c437 325 ./shellcode]> pd $r @ rsp
            ;-- rsp:
            0x7ffcef34c437      90             nop
            0x7ffcef34c438      31c0           xor eax, eax
            0x7ffcef34c43a      4831d2         xor rdx, rdx
            0x7ffcef34c43d      50             push rax
            0x7ffcef34c43e      50             push rax
            0x7ffcef34c43f      c704242f2f62.  mov dword [rsp], 0x69622f2f
            0x7ffcef34c446      c74424046e2f.  mov dword [rsp + 4], 0x68732f6e
            0x7ffcef34c44e      4889e7         mov rdi, rsp
            0x7ffcef34c451      50             push rax
            0x7ffcef34c452      50             push rax
            0x7ffcef34c453      66c704242d69   mov word [rsp], 0x692d
            0x7ffcef34c459      4889e6         mov rsi, rsp
            0x7ffcef34c45c      52             push rdx
            0x7ffcef34c45d      56             push rsi
            0x7ffcef34c45e      57             push rdi
            0x7ffcef34c45f      4889e6         mov rsi, rsp
            0x7ffcef34c462      4883c03b       add rax, 0x3b
            0x7ffcef34c466      0f05           syscall
            0x7ffcef34c468      c70440000000.  mov dword [rax + rax*2], 0
{% endhighlight %}

Now this code is purposely confusing, first it zeros out both `eax` and `rdx` and pushes them onto the stack, growing it. We then move some values into the space on the stack, and move the current stack pointer into `rdi`. We then do the same thing again, making space as we go, via pushing a zero'd out `rax` and saving the new value of the stack pointer to `rsi`. Afterwards we can see that we are adding **0x3b** to `rax` (which is 0) and executing a `syscall`. 0x3b is 49 in octal, so we are calling syscall 49, which has the following call signature:

{% highlight c %}
int sys_execve(const char *filename, const char *argv[], const char *const envp[]);
{% endhighlight %}

So the values currently pointed to by `rdi` and `rsi` must be a filename and arguments for `sys_execve`, which starts a process!

Lets place a break point before the syscall and take a look at the values pointed to by `rdi` and `rsi` to see what is going to be called:

{% highlight nasm %}
[0x7ffc1b7d3c7c]> psz @ rdi
//bin/sh
[0x7ffc1b7d3c7c]> psz @ rsi
?<}
[0x7ffc1b7d3c7c]>
{% endhighlight %}

From this we can see that this shellcode will simply start a shell, by launching `/bin/sh`. But the argument is just the string `?<}`.. If we take a look at the stack we can see that there is a `-i` sitting exactly 16 bytes away, which would force the shell to spawn in interactive mode, but instead it is getting passed a junk value. Perhaps the author made a mistake and is passing the wrong value? This won't stop the shell from launching so it may have been easy to miss.

{% highlight nasm %}
[0x7ffc1b7d3c7c 325 ./shellcode]> ?0;f tmp;s.. @ rdi+61 # 0x7ffc1b7d3c7c
- offset -       0 1  2 3  4 5  6 7  8 9  A B  C D  E F  0123456789ABCDEF
0x7ffc1b7d3c17  3f3c 7d1b fc7f 0000 2f3c 7d1b fc7f 0000  ?<}...../<}.....
0x7ffc1b7d3c27  0000 0000 0000 0000 2d69 0000 0000 0000  ........-i......
0x7ffc1b7d3c37  0000 0000 0000 0000 2f2f 6269 6e2f 7368  ........//bin/sh
0x7ffc1b7d3c47  0000 0000 0000 0000 bf08 6000 0000 0000  ..........`.....
orax 0xffffffffffffffff   rax 0x0000003b           rbx 0x00000000
 rcx 0x00000000           rdx 0x00000000            r8 0x7f4b83a26300
  r9 0x7f4b83a392e0       r10 0x00000000           r11 0x7f4b836bcdb0
 r12 0x004003a0           r13 0x7ffc1b7d3d80       r14 0x00000000
 r15 0x00000000           rsi 0x7ffc1b7d3c17       rdi 0x7ffc1b7d3c3f
 rsp 0x7ffc1b7d3c17       rbp 0x7ffc1b7d3ca0       rip 0x7ffc1b7d3c86
{% endhighlight %}

I hope this will be of some help to someone as a simple intro to using radare2 as a debugger. As someone who lives in the terminal as much as possible I am loving using r2, but hopefully this will convince others that it is actually not any harder to use than a visual decompiler / debugger.

PS: I would like some feedback as to what people would prefer to see used for the examples in my articles. Would you prefer the text be placed in a plaintext code block instead of images? I am aware that some people hate posts with too many images and I've been meaning to step up my game and actually start writing more posts. You can either let me know in the comment section or on [twitter](https://twitter.com/_scrapbird) or email me (contact info can be found in the footer of this blog).
