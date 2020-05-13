# CSAPP: Bomb Lab Note

## Intruction:
BombLab is the first lab in CS:APP. I were provided with a binary file, and the main function file.
When executing the binary, the program would request 6 string from user. If anyone is not correct, 
then the program would call explode_bomb and exit program.
User has to use reverse engineering assembly code to find out the input string and difuse the bomb.  

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;*"But isn't something... missing? Perhapssomething they overlooked?"*

## Before you start
You are suggested to have some basic understanding of Assembly langauge and memory allocation. Some suggestion:
* Read *Bryant & O’Hallaron, "Computer Systems: a Programmer’s Perspective", 3rd Edition
* Comfortable using [gdb](http://csapp.cs.cmu.edu/2e/docs/gdbnotes-x86-64.pdf)
* Comfortable reading [x86-64 assembl](https://cs.brown.edu/courses/cs033/docs/guides/x64_cheatsheet.pdf),
* Have knowledge of data structure: [BST](https://www.geeksforgeeks.org/binary-search-tree-data-structure/), and [Linklist](https://www.geeksforgeeks.org/data-structures/linked-list/)

## Basic idea
User use `objdump -d bomb > asb.txt` to generate assembly code from binary file. Check assembly code, you will find phase_1() to phase_6() in main(), each preceeded by a getline() and followed by phase_defuse().  
Therefore, the task is divided into 6 phases

## Target program
**bomb** the binary file that take in input from stdin or user command line.
Or you can store your answers in file and pass to **bomb**

## Method used for the lab
* `objdump -d <target>` to get the assembly code for binary file
* `gdb <target>`
* `(gdb) break` set break point
* `(gdb) si` and `(gdb) disass` to walk through code
* Write C script to bruteforce desired input to recursive function

## My notes
```
Phase1 is simple, compare the return value of readline with string stored in 0x4023f0
phase1: rdi = readline
mov $4023f0 esi @ # 
(gdb) x/32bs 0x4023f0
For NASA, space is still a high priority.

----------------------------------------------------------------
input should be 6 numbers, otherwise bomb would explode
There is a loop to generate geometric sequence and compare with input term by term, also very simple.
phase2: rdi = readline
rsp -= 0x28
rsi = rsp
rbx = rsp
*rbp = 32
1 2 4 8 16 32

----------------------------------------------------------------
phase3: rdi = readline
1 at rsp+c
2 at rsp+8
rsp start at 0x7fffffffe070
rsp -= 18
rcx = (rsp+8)
rdx = (rsp+12) unsign <=7 
esi = 0x402705
eax = 0   input must > 1
eax = rsp+12

----------------------------------------------------------------
This phase readline take in two int between 1 and 14 and pass to a recursive function func4.
compare the output of func4 with a preset value, pass if equal.
so I restored func4 in C and run a loop to find the input to get desired output
phase4: rdi = readline
rsp+8 = y = 1b = 27
rsp+12 = x = 9
rsp -= 18
rcs = rsp+8
rdx = rsp+12
esi = 402705
eax = 0 argument# = 2
(rsp+12)<=14 unsign
edx = 14
esi = 0
edi = x
func4(edi = x, esi = 0, edx = 14)
	eax = edx = 14
	eax = eax-esi = 14
	ebx = eax = 14
	ebx >> 31 = signof eax eg 0
	ebx += eax = 14 >> 1 = 7
int func4(int a, int b, int c)
{
	int x = (((c-b) + ((c-b)>>31)) >> 1) + b;
	if(x > a)
	{
		return x + func4(a, b, x - 1);
	}
	if(x < a)
	{
		return x + func(a, x + 1, c);
	}
	return x;
}
for(int i = 0; i <= 14; i++)
	{
		if(f(i, 0, 14)==27){
			printf("%d\n", i);
		}
	}

----------------------------------------------------------------
phase5: rdi = readline
rbx = rdi     arg is string of length 6
rax = rbx = readline
rdi = rbx+6
ecx = 0

	{J1 {rdx = ASCII value of first char e.g 0011 0001
		edx & 0000 1111 
		// now edx hold lower 4 bits of ASCII value of i-th position of input string.
		ecx += (4*rdx + 0x4024a0)
		rax += 1
		if (rdi != rax) jump J1
	} // 
	0x4024a0:
	0:	2  		1:	 10			2:	6		3:	1 
	4:	12		5:	 16			6:	9		7:	3
	8:	4		9:	 7			10:	14		11:	5
	12:	11		13:	 8			14: 15		15:	13
	
conclusion for loop above. excute 6 time, ecx = sum of the corresponding value of rdx in tabe 0x4024a0. rbx is ASCII of lower 4 bits of i-th input char
ecx need to be 0x32 = 50
one Solution is 111128

-------------------------------------------------------------
phase6: rdi = readlin
input is a vector of 6 int, with each entry between 1 and 6
There was an address passed to rdx, which, if check with x/128x $rdx, is an array of nodes.
The main part of phase 6 is link these nodes into linklist in the order of input vector
And check if the int stored in these nodes are in decending order
-------------------------------------------------------------
Its tricky to get into secret phase.
I find the call the secret_phase is inside phase_difused
to call secretphase, there is a comparion of __isoc99_sscanf@plt(0x6048b0, 0x40274f)
walk with gdb, I find the first address is answer of phase4 and return value is number of argument of answer4.
and what stored in second address is format %d %d %s. So we need to add an string to answer4. there are
several address store sting in phase_dufused, check the one that is compared with answer4, I got DrEvil.
there is also a check if you have passed 6 phases before you can get into secret phase

The secret_phase itself is simple if you are familiar with binary search tree.
Secret: rdi = readline      long(1 <= input <= 1001)
rax = strtol(rdi, esi, base=10)
rbx = rax
eax = rax -= 1 (0 <= eax <= 1000)
esi = ebx = rax = first 32 bit of input
edi = 0x604110 head of BST  {int left right}
SO there is a binary search tree laid in memory.
func7(edi, esi)
test   %eax,%eax
jne    401241 <secret_phase+0x4d> explode
we can see, to difuse bomb, we want func7 return 0;
func7 is a modified binary search, but make some operation on returned pointer
I then passed the value of head and defused secret phase
func7(edi,esi){
	if(edi==0){return -1} 			(find nothing)
	edx = *edi
	if(edx > esi) {						
		edi += 8			this->left
		return 2 * fun7(edi,esi) 
	} else if (edx < esi) {
		edi += 16			this->right
		return 2 * fun7(edi,esi) + 1
	} else if (edx == esi) {
		return this
	}
}

```

## Author
* **Chen Li** 


## Acknowledgments
* This project is based on *Bryant & O’Hallaron, "Computer Systems: a Programmer’s Perspective", 3rd Edition*

