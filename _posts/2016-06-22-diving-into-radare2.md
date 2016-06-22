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

{% highlight bash %}
➜  ~ gcc --version
gcc (Debian 4.7.2-5) 4.7.2
Copyright (C) 2012 Free Software Foundation, Inc.
This is free software; see the source for copying conditions.  There is NO
warranty; not even for MERCHANTABILITY or FITNESS FOR A PARTICULAR PURPOSE.

➜  ~ gcc shellcode.c -fno-stack-protector -z execstack -o shellcode
➜  ~
{% endhighlight %}

Then open it with r2:

<img src="/images/2016-06-22/561CySN.png" width="340" alt="radare 2's prompt" />

As can be seen by the r2 prompt, we at currently positioned at offset **0x004003a0**. Lets analyze the whole file and seek to the main subroutine:

![main subroutine](/images/2016-06-22/9A4n0rn.png)

We can see that r2 has analyzed our bin and named the shellcode subroutine for us and that the program is putting the address of the shellcode into `edx`, then calling it. Lets take a look at the shellcode now.

<img src="/images/2016-06-22/vP7Mps8.png" width="300" alt="shellcode subroutine" />

Hmm.. looks like r2 hasn't recognized this part of the code as a function, lets jump into visual disassembly mode by executing `V` to get into visual mode and pressing `p` to cycle to the disassembly view.

![visual disassembly of shellcode](/images/2016-06-22/jhNnMjb.png)

After a quick look at this we can see that the shellcode jumps down to **0x6008bf** which is an instruction to call.. a string? Looks like r2 has mistaken that line of code for data, but that's okay because we can fix it. Lets scroll down to that section by pressing `j` once and mark it as code by pressing `dc` (a good mnemonic for this is **d**efine **c**ode). After doing that we get the following:

![visual disassembly of shellcode fixed](/images/2016-06-22/zwrPBhd.png)

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

{% highlight bash %}
db 0x006008bd
{% endhighlight %}

Now lets run the program by entering the `dc` command (**d**ebugger **c**ontinue) and then seek to the location that `rsp` currently points to with `s rsp`. We should now be looking at the following:

![final shellcode decompiled in debug mode](/images/2016-06-22/f5BbRqK.png)

Now this code is purposely confusing, first it zeros out both `eax` and `rdx` and pushes them onto the stack, growing it. We then move some values into the space on the stack, and move the current stack pointer into `rdi`. We then do the same thing again, making space as we go, via pushing a zero'd out `rax` and saving the new value of the stack pointer to `rsi`. Afterwards we can see that we are adding **0x3b** to `rax` (which is 0) and executing a `syscall`. 0x3b is 49 in octal, so we are calling syscall 49, which has the following call signature:

{% highlight c %}
int sys_execve(const char *filename, const char *argv[], const char *const envp[]);
{% endhighlight %}

So the values currently pointed to by `rdi` and `rsi` must be a filename and arguments for `sys_execve`, which starts a process!

Lets place a break point before the syscall and take a look at the values pointed to by `rdi` and `rsi` to see what is going to be called:

<img src="/images/2016-06-22/3Tfkit3.png" width="340" alt="value of rdi and rsi before the syscall" />

From this we can see that this shellcode will simply start a shell, by launching `/bin/sh`. But the argument is just the string `Q`, if we take a look at the stack we can see that there is a `-i` sitting there exactly 16 bytes away, which would force the shell to spawn in interactive mode. Perhaps the author made a mistake and is passing the wrong value? This won't affect the shell from launching so it may have been easy to miss.

![values of stack and registers](/images/2016-06-22/LHmfBE4.png)

I hope this will be of some help to someone as a simple intro to using radare2 as a debugger. As someone who lives in the terminal as much as possible I am loving using r2, but hopefully this will convince others that it is actually not any harder to use than a visual decompiler / debugger.
