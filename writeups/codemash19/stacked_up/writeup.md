
## Stacked Up

"Stacked Up" was the only binary exploitation challenge at the CodeMash CTF this year:


```
A vulnerable service is running here:

nc whale.hacking-lab.com 5777

You have the binary of the service. Analyze it, find a vulnerability, and then exploit the server to get the flag!
```

So we have the address of a remote service and a copy of the binary it's running that we can download to disassemble and debug locally. When we unpack the zip file and check out the binary we can see that it's for 64-bit Linux. 
The binary seems to be just a very simple program that reads a line of input and echoes it back:

```
joseph@ubuntu:~/ctf$ ./stack
Input, please! >>---------->
hello
hello
```

For a program this simple, there aren't many possible bugs. The first thing that should jump to mind is a buffer overflow, and sure enough, if we type a very long input string the program segfaults:

```
joseph@ubuntu:~/ctf$ ./stack
Input, please! >>---------->
<a couple thousand a's>
<a couple thousand a's>
Segmentation fault (core dumped)
```

Next we can put the program into a disassembler to see exactly what's going on. Here's the disassembly
of main() in radare2:

<insert picture here>

main() is basically just a sequence of standard library function calls: puts(), fflush(), gets(), and puts() 
again. The other instructions are there to handle setting up and tearing down the stack 
frame for main(), and passing arguments to each function call. Notice how the argument to gets() is a memory
location relative to rbp - this indicates that the buffer overflow is happening in a local variable of main()
stored on the stack. At the top of radare2's disassembly, it shows the offsets for the local variables.
The argument to gets() (which radare2 has automatically named `char *s`) is at rbp - 0x400. This is 
a good clue that the buffer is 0x400 (1024 in decimal) bytes. 


The source code probably looked something like 
this:


```
int main() {
    char s[1024];

    puts("Input, please! >>----------> ");

    fflush(stdout);

    gets(s);
    puts(s);

    return 0;
}
```

Since the buffer is at [rbp-0x400], if we write more than 0x400 bytes we'll overflow the buffer
and start overwriting what's stored after it. Typically after the local variables we should
expect to see any registers that main() has to store, the return address, and then the stack frame for
the calling function. We're most interested in the return address because by overwriting it
we can cause the program to start executing code from an address of our choice. The disassembly
shows that the stored rbp (the value of rbp for the function that called main) is 
at [rbp+0x0]. The next doubleword, at [rbp+0x8], is the stored return address. 

So the bug here has a lot of features that make exploitation convenient: gets() only stops 
at a newline, so we don't have to worry about avoiding null bytes; the length is unlimited so we don't
have to worry about any length restrictions; the overflow is already on the stack, so there's
no need for a stack pivot or any other extra steps before kicking off a ROP chain; 
and lastly, there's no stack canary or anything at all between the buffer and the return address.
We have a clean overwrite of the stored rip plus control of the stack after for a ROP chain.

The program is using appropraite memory protections, so we won't be able to execute any
shellcode we write in memory. The pwnlib checksec tool is convenient for summarizing the protection
mechanisms we're dealing with:

```
joseph@ubuntu:~/ctf/codemash19$ pwn checksec ./stack
[*] '/home/joseph/ctf/codemash19/stack'
    Arch:     amd64-64-little
    RELRO:    Partial RELRO
    Stack:    No canary found
    NX:       NX enabled
    PIE:      No PIE (0x400000)
```

NX means that the writable memory segments in the process will not also be executable, so the 90s-style
"write shellcode, jump to shellcode" buffer overflow won't work. The exact state of ASLR will depend on
the operating system where the service is hosted, but it's probably safe to assume that ASLR will be
enabled in this challenge (otherwise, it would be a much easier challenge). However, we can see from the
checksec output that PIE is not enabled. This means that, at a minimum, the segments associated with the
`stack` binary will be loaded at a predictable location in memory (0x400000) in this case). These are
the only memory locations we can count on being at a predictable address. 

In order to get around the NX, we will have to jump into existing code (ROP attack).  
Of course, to jump to some existing code we will have to know the address of that
code - so this limits us to code in the `stack` binary itself, rather than the shared
libraries (at least at first). This is a pain because `stack` is such a simple program that it probably
won't have enough gadgets to do what we want. A quick look confirms that we don't have much to work
with here:

```
joseph@ubuntu:~/ctf/codemash19$ ROPgadget --binary ./stack 
...
Unique gadgets found: 62
```

62 gadgets is not a lot. Compares this to the libc shared library on my system:

```
joseph@ubuntu:~/ctf/codemash19$ ROPgadget --binary /lib/x86_64-linux-gnu/libc-2.23.so | tail -n 1
Unique gadgets found: 21212
```






