---
layout: post
title: Generic Unpacking with r2pipe
---

I've been playing around with `r2pipe` lately and thought I would do a bit of a write up on how I automated unpacking a Locky sample using r2 over `r2pipe`. The method I used was to break on calls to `VirtualAlloc` and then watch for a write to the allocated buffer and check for a PE header, the same method that [@struppigel](https://twitter.com/struppigel) describes very well in this youtube video: [https://www.youtube.com/watch?v=h9RiBJ06MAQ](https://www.youtube.com/watch?v=h9RiBJ06MAQ). I will use the same Locky sample that he used, which can be found [here](https://www.hybrid-analysis.com/sample/49a48d4ff1b7973e55d5838f20107620ed808851231256bb94c85f6c80b8ebfc?environmentId=100) if you would like to follow along.

The Algorithm
-
The basic algorithm I will be using is as follows:
- Set a breakpoint on `VirtualAlloc`
- Run until breakpoint is hit
- Step out of `VirtualAlloc` and record the address of the allocated buffer, which is the return value of `VirtualAlloc` and can be found in `eax`
- Step out of the calling function (which during my tests I found is usually also the function that writes to the allocated buffer, more on this later)
- Check the first 2 bytes of the buffer for the PE `MZ` header
- If PE header is found, dump the buffer to a file and quit, otherwise repeat the process

I chose to step out of the function calling `VirtualAlloc` instead of setting a hardware breakpoint to break when the buffer is written to because I found this to land me in system code somewhere which I would then have to step out of until I get back to user code to be sure the buffer is fully written. I am lazy and the simple solution of just stepping out of the calling function turned out to be more reliable in my tests.

The Setup
-
Because I dislike developing on Windows and much prefer a unix environment for running and testing my python `r2pipe` script I set up a KVM virtual machine running a 64 bit installation of Windows 7 Professional. To aid in testing I installed my favourite Windows debugger (x64dbg) and radare2. On the host OS I had a python virtual environment with `r2pipe` installed which I used to run the script, connecting over HTTP to an instance of r2 running on the windows VM. You can run the script directly on Windows or use one of the other remote connection methods r2 supports but this is the only setup I tested this in.

I started r2 with the following command:
{% highlight batch %}
radare2.exe -c "e http.sandbox = false; e http.bind = 0.0.0.0; =h& 1337" -d C:\Users\[REDACTED]\Desktop\49a48d4ff1b7973e55d5838f20107620ed808851231256bb94c85f6c80b8ebfc.bin
pause
{% endhighlight %}

This will start r2 in debug mode (`-d`) and instruct it to listen on all network adapters for an HTTP connection on port 1337. I also turned off the HTTP sandbox with `http.sandbox = false` as debugging features are disabled by default when starting an HTTP listener for security reasons.

The Code
-
Alright now we can get into the fun part, the actual code.

First we import `r2pipe`, open a connection to our VM and setup a few helper functions to simplify our lives:
{% highlight python %}
#!/usr/bin/env python

import r2pipe

r2 = r2pipe.open('http://192.168.100.64:1337')


def cont():
    print(r2.cmd('dc'))


def step_out():
    print(r2.cmd('dcr; ds'))


def breakpoint(addr):
    print(r2.cmd('db ' + addr))


def get_esp():
    return r2.cmdj('drj')['esp']


def get_eip():
    return r2.cmdj('drj')['eip']
{% endhighlight %}

Next we will reopen the file in debug mode with the r2 command `doo`, this ensures we can run the script multiple times and it will restart the debugged program (Locky) each time. After that we continue execution to allow some standard libraries to load, set a breakpoint on `entry0` and continue until we hit it.

{% highlight python %}
print('Starting debugging')
print(r2.cmd('doo'))
cont()
breakpoint('entry0')
print('Breakpoints:\n%s' % r2.cmd('db'))
cont()
print('Hit breakpoint at: %s' % r2.cmd('s'))
{% endhighlight %}

We can now analyze the program with `aaa` as all of the imports have been loaded.

{% highlight python %}
r2.cmd('aaa')
{% endhighlight %}

This will name our functions for us so that if we wish we can have a look around. We don't actually need this level of analysis for this script but again, I'm lazy and this file is small so it doesn't take very long.

Next on the list of things to do is get the address of `VirtualAlloc` and set a breakpoint on it. I also print out some debug messages throughout the code to help me see what is going on.

{% highlight python %}
VirtualAlloc = int(r2.cmd('? [sym.imp.KERNEL32.dll_VirtualAlloc]~[0]'))
print('VirtualAlloc is at: ' + str(hex(VirtualAlloc)))
print('Current addr: ' + str(hex(get_eip())))

breakpoint('[sym.imp.KERNEL32.dll_VirtualAlloc]')  # Break on VirtualAlloc
print('Breakpoints:\n%s' % r2.cmd('db'))
{% endhighlight %}

The final part of the script is the main loop which carries out the algorithm we described above. It looks like this:

{% highlight python %}
while True:
    cont()
    print('Hit breakpoint at: %s' % r2.cmd('s'))
    eip = get_eip()
    if eip == VirtualAlloc:
        print('Break in VirtualAlloc')
        esp = get_esp()
        allocated_size = int(r2.cmd('pv @ 0x%x' % (esp + 8)), 16)
        step_out()  # step out of VirtualAlloc
        last_allocated_memory = r2.cmdj('drj')['eax']
        print('VirtualAlloc allocated 0x%x bytes of memory at 0x%x' % (allocated_size, last_allocated_memory))

        step_out()
        if r2.cmd('p8 2 @ 0x%x' % last_allocated_memory) == '4d5a':
            print('Found PE header at 0x%x' % last_allocated_memory)
            print(r2.cmd('wt dump.bin 0x%x @ 0x%x' % (allocated_size, last_allocated_memory)))
            print('PE dumped to dump.bin in %s' % r2.cmd('pwd'))
            quit()
{% endhighlight %}

The code works as follows:
- We continue execution until we hit a breakpoint
- Double check we are actually stopped at `VirtualAlloc`
- If so, print out some messages and get the address `esp` currently points to
- Get the size of the buffer (`esp+8`) being allocated, which is the second argument to `VirtualAlloc`. Values can be retrieved in r2 with the `pv` command (**p**rint **v**alue)
- Step out of `VirtualAlloc`
- Read the address of the allocated buffer from `eax` using `drj`, which returns a json representation of all register values.
- We can now read 2 bytes from the start of the buffer using `p8 2` and compare them with the `MZ` string at the start of all PE headers, in the hex string form `4d5a`.
- If it matches, dump the allocated buffer to file with r2's handy `wt` (**w**rite **t**o) command.

Putting this all together we get the following:

{% highlight python %}
#!/usr/bin/env python

import r2pipe

r2 = r2pipe.open('http://192.168.100.64:1337')


def cont():
    print(r2.cmd('dc'))


def step_out():
    print(r2.cmd('dcr; ds'))


def breakpoint(addr):
    print(r2.cmd('db ' + addr))


def get_esp():
    return r2.cmdj('drj')['esp']


def get_eip():
    return r2.cmdj('drj')['eip']


print('Starting debugging')
print(r2.cmd('doo'))
cont()
breakpoint('entry0')
print('Breakpoints:\n%s' % r2.cmd('db'))
cont()
print('Hit breakpoint at: %s' % r2.cmd('s'))

r2.cmd('aaa')

VirtualAlloc = int(r2.cmd('? [sym.imp.KERNEL32.dll_VirtualAlloc]~[0]'))
print('VirtualAlloc is at: ' + str(hex(VirtualAlloc)))
print('Current addr: ' + str(hex(get_eip())))

breakpoint('[sym.imp.KERNEL32.dll_VirtualAlloc]')  # Break on VirtualAlloc
print('Breakpoints:\n%s' % r2.cmd('db'))

while True:
    cont()
    print('Hit breakpoint at: %s' % r2.cmd('s'))
    eip = get_eip()
    if eip == VirtualAlloc:
        print('Break in VirtualAlloc')
        esp = get_esp()
        allocated_size = int(r2.cmd('pv @ 0x%x' % (esp + 8)), 16)
        step_out()  # step out of VirtualAlloc
        last_allocated_memory = r2.cmdj('drj')['eax']
        print('VirtualAlloc allocated 0x%x bytes of memory at 0x%x' % (allocated_size, last_allocated_memory))

        step_out()
        if r2.cmd('p8 2 @ 0x%x' % last_allocated_memory) == '4d5a':
            print('Found PE header at 0x%x' % last_allocated_memory)
            print(r2.cmd('wt dump.bin 0x%x @ 0x%x' % (allocated_size, last_allocated_memory)))
            print('PE dumped to dump.bin in %s' % r2.cmd('pwd'))
            quit()
{% endhighlight %}

The script takes about 2 minutes to run and yields an unpacked version of Locky named `dump.bin` in the working directory of where ever you launched radare2.exe from. I'm sure there are many improvements I could make to this script but this is my first time playing around with r2pipe so please send me suggestions, I would love to hear them.

This script can also be found as a [gist](https://gist.github.com/scrapbird/d215ec29d9cc67f9e671631cbe700391).
