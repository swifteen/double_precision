[TOC]

# 一、浮点数计算误差分析

## 测试程序

### 例子 1

```c++
#include <stdio.h>
int main()
{
	int a = 33800;
	long long b = 13*sizeof(short)*(a);
	printf("[%lld]\n",b);
}
```

### 例子 2

```c++
#include <stdio.h>
int main()
{
	int a = 33800;
	long long b = 13*1000*sizeof(short)* (double)(a/1000.0);
	printf("[%lld]\n",b);
}
```

例子1输出878800
例子2输出878799

为何会产生差异呢，只能从汇编代码入手，一点一点分析浮点计算的过程

## 例子1汇编代码

例子1不涉及浮点数，因此汇编代码比较简单，先通过例子1的汇编代码进行了解。

将例子1生成汇编文件，把汇编代码分4个重要部分说明，如下

#### 片段1

https://bob.cs.sonoma.edu/IntroCompOrg-RPi/sec-stack-manage.html

The `push` instruction at the beginning of this function:

1. Pushes the return address, the value contained in the `lr` register, onto the stack.
2. Pushes the caller's frame pointer, the value contained in the `fp` register, onto the stack.
3. Updates the `sp` to show that two 32-bit values have been pushed onto the top of the stack.

The `add` instruction adds 4 to the value in the `sp` register and stores the sum in the `fp` register, thus setting the frame pointer for this function such that it points to the location on the stack where the frame pointer of the calling function is stored.

```assembly
	push	{fp, lr}				; 1、返回地址保存在lr (r13) 寄存器中，将lr寄存器中的值入栈;2、将fp寄存器中的值入栈
	.save {fp, lr}
	.setfp fp, sp, #4
	add	fp, sp, #4					; 将sp寄存器中的值加4之后的和，保存到fp寄存器中
	.pad #16
	sub	sp, sp, #16
```

.save .setfp .pad

#### 片段2

```assembly
	ldr	r3, .L3						; 将.L3处的值加载到r3寄存器中   代码中.L3处实际的值为33800，即将33800保存到r3寄存器中
	str	r3, [fp, #-16]				; 将r3寄存器中的值保存到栈fp-16的位置
	ldr	r3, [fp, #-16]				; 将栈fp-16位置的值加载到r3寄存器中，这里为何多此一举，担心其它进程覆盖r3吗？
```

#### 片段3

```assembly
	mov	r2, r3						; 将r3寄存器中的值拷贝到r2寄存器中
	mov	r3, r2						; 将r2寄存器中的值拷贝到r3寄存器中，这里为何多此一举
	lsl	r3, r3, #1					; Logical shift left逻辑左移1位，相当于乘以2
	add	r3, r3, r2					; 将r2寄存器和r3寄存器中的值相加得到的和，保存到r3寄存器中
	lsl	r3, r3, #2					; Logical shift left逻辑左移2位，相当于乘以4，这里把r3寄存器中值乘以4
	add	r3, r3, r2					; 将r2寄存器和r3寄存器中的值相加得到的和，保存到r3寄存器中
	lsl	r3, r3, #1					; Logical shift left逻辑左移1位，相当于乘以2
	mov	r2, r3						; 将r3寄存器中的值拷贝到r2寄存器中
	mov	r3, #0						; 将数值0拷贝到r3寄存器中
	strd	r2, [fp, #-12]			; 将r2寄存器中的值保存到栈fp-12位置
	ldrd	r2, [fp, #-12]			; 将栈fp-12位置的值保存到r2寄存器中
	ldr	r0, .L3+4					; 将.L3+4处的值保存到r0寄存器中，代码中.L3+4代表[%lld]\012\000，即为printf函数的输入参数
```

13 * 2 * 33800   =  [ (33800 * 2  + 33800) * 4  + 33800 ] * 2

可知数字13会被分解成  (2 + 1) * 4 + 1

#### 片段4

```assembly
	bl	printf						; 调用printf函数
	mov	r3, #0
	mov	r0, r3
	sub	sp, fp, #4
	@ sp needed
	pop	{fp, pc}
.L3:
	.word	33800
	.word	.LC0
```

解释1：A calling function uses the `bl` instruction to call a function, which places the return address in the `lr` (`r13`) register. 

解释2：So this instruction effectively pops the caller's frame pointer off the top of the stack, back into the fp register. Then the return address is popped into the pc, and the stack pointer, sp, is updated to the new top of the stack. This acts as an epilogue to clean up the stack after performing the algorithm that is the purpose of this function.

```assembly
pop     {fp, pc}
```

## 例子2汇编代码

```c++
#include <stdio.h>
int main()
{
	int a = 33800;
	long long b = 13*1000*sizeof(short)* (double)(a/1000.0);
	printf("[%lld]\n",b);
}
```

#### 汇编片段

```assembly
.LC0:
	.ascii	"[%lld]\012\000"
	.text
	.align	2
	.global	main
	.arch armv6
	.syntax unified
	.arm
	.fpu vfp
	.type	main, %function    
.LFB0:    
    ......							;省略了部分
	sub	sp, sp, #16
	ldr	r3, .L3+16					; 将.L3+16处的值加载到r3寄存器中   代码中.L3+16处实际的值为33800，即将33800保存到r3寄存器中
	str	r3, [fp, #-8]				; 将r3寄存器中的值保存到栈fp-8的位置		对应代码int a = 33800;将局布变量入栈
	ldr	r3, [fp, #-8]				; 将栈fp-8位置的值加载到r3寄存器中
	vmov	s15, r3	@ int			; 将r3寄存器的值(33800)拷贝到s15单精度浮点寄存器中
	vcvt.f64.s32	d6, s15			; 将s15浮点寄存器中32位单精度浮点数转换为64位双精度浮点保存到d6双精度浮点寄存器中(33800.0)
	vldr.64	d5, .L3				    ; 将.L3处的值加载到d5寄存器中 .L3处前32位值为0存为double的低32位，后32位值为1083129856存为double的高32位
	vdiv.f64	d7, d6, d5			; 将d6保存的double值(33800.0)除以d5保存的double值(1000.0)，得到值保存到d7双精度浮点寄存器中
	vldr.64	d6, .L3+8				; 将.L3+8处的值加载到d6双精度浮点寄存器中  代码中.L3+8处前32位的值为0，后32位的值为1087988736
	vmul.f64	d7, d7, d6			; 将d7保存的double值乘以d6保存的double值，得到值保存到d7双精度浮点寄存器中
	vmov	r0, r1, d7				; 将d7双精度浮点寄存器中的低32位的值保存到r0中，将d7双精度浮点寄存器中的高32位的值保存到r1中
	bl	__aeabi_d2lz				; double to long long C-style conversion ,此时r0, r1为输入参数
	mov	r2, r0
	mov	r3, r1
	strd	r2, [fp, #-20]
	ldrd	r2, [fp, #-20]
	ldr	r0, .L3+20
	bl	printf
	mov	r3, #0
	mov	r0, r3
	sub	sp, fp, #4
	@ sp needed
	pop	{fp, pc}
.L3:
	.word	0						;double类型1000.0的低32位
	.word	1083129856				;double类型1000.0的高32位
	.word	0						;double类型26000.0的低32位   编译时就进行了计算得到26000
	.word	1087988736				;double类型26000.0的高32位	编译时就进行了计算得到26000
	.word	33800
	.word	.LC0
```

#### 运算步骤

1. 将32位整型33800赋值到单精度浮点寄存器s15中
2. 将单精度浮点寄存器s15中的值转换为双精度浮点数，保存到双精度浮点寄存器d6中(33800.0)
3. 加载1000.0这个双精度浮点数(编译时就转换为双精度了)到双精度浮点寄存器d5中
4. 将d6保存的double值(33800.0)除以d5保存的double值(1000.0)，得到值保存到d7双精度浮点寄存器中
5. 加载26000.0这个双精度浮点数(编译时就进行了计算得到26000，并转换为双精度了)到双精度浮点寄存器d6中
6. 将d7保存的double值乘以d6保存的double值(26000.0)，得到值保存到d7双精度浮点寄存器中
7. 将d7双精度浮点寄存器中的高32位保存到r0寄存器中，低32位保存到r1寄存器中
8. 调用__aeabi_d2lz指令，以r0和r1中保存的浮点数为参数，将浮点数转换成long long类型

#### 代码中的浮点数常量

根据国际标准IEEE 754，任意一个二进制浮点数V可以表示成下面的形式：

![img](https:////upload-images.jianshu.io/upload_images/3229842-dc513ebfe7b96346.jpg?imageMogr2/auto-orient/strip|imageView2/2/w/295/format/webp)

（1）(-1)^s表示符号位，当s=0，V为正数；当s=1，V为负数。 
（2）M表示有效数字，大于等于1，小于2。 
（3）2^E表示指数位。

对于**double**双精度浮点数，用 1 位表示符号，用 11 位表示指数，52 位表示尾数，其中指数域称为阶码

| 高32位十进制 | 2位二进制表示                             | S    | e的二进制       | e的十进制 | E(E = e - 1023) | M            | V     |
| ------------ | ----------------------------------------- | ---- | --------------- | --------- | --------------- | ------------ | ----- |
| 1083129856   | `0100 0000 1000 1111 0100 0000 0000 0000` | 0    | `100 0000 1000` | 1032      | 1032-1023=9     | 1.111101     | 1000  |
| 1087988736   | `0100 0000 1101 1001 0110 0100 0000 0000` | 0    | `100 0000 1101` | 1037      | 1037-1023=14    | 1.1001011001 | 26000 |

由上可知26000和1000.0在编译时就已经提前转换为双精度浮点数保存到.L3指定的常量区域了

## gdb调试跟踪每条指令

#### gdb基本命令

先从简单的gdb命令入手，参考https://bob.cs.sonoma.edu/IntroCompOrg-RPi/sec-gdb1.html#p-342

```shell
(gdb) li			#打印行号
(gdb) br 8			#在第8行打一个断点
(gdb) s			   	#执行到下一个断点
(gdb) print b      	#打印变量b
```

```shell
Starting program: /home/pi/shared/boox/test_double1_gdb 

Breakpoint 1, main () at test_double1_gdb.cpp:6
6               double  b = 13*1000*sizeof(short)* (double)(a/1000.0);
(gdb) li
1       #include <stdio.h>
2       int main()
3       {
4               int a = 33800;
5               //long long b = 13*1000*sizeof(short)* (double)(a/1000.0);
6               double  b = 13*1000*sizeof(short)* (double)(a/1000.0);
7               long long c = b;
8               printf("[%lf][%lld]\n",b,c);
9       }
(gdb) br 8
Note: breakpoint 3 also set at pc 0x10508.
Breakpoint 4 at 0x10508: file test_double1_gdb.cpp, line 8.
(gdb) r
The program being debugged has been started already.
Start it from the beginning? (y or n) y
Starting program: /home/pi/shared/boox/test_double1_gdb 

Breakpoint 1, main () at test_double1_gdb.cpp:6
6               double  b = 13*1000*sizeof(short)* (double)(a/1000.0);
(gdb) s

Breakpoint 2, main () at test_double1_gdb.cpp:7
7               long long c = b;
(gdb) s

Breakpoint 3, main () at test_double1_gdb.cpp:8
8               printf("[%lf][%lld]\n",b,c);
(gdb) print b
$6 = 878799.99999999988
(gdb) print c
$7 = 878799
```

从上面可知浮点计算得到878799.99999999988后，再强制转换为long long丢掉了小数部分，导致最后结果为878799

#### 对比添加-g选项生成汇编文件

由下图可知，编译时添加-g选项后，并不影响真正有用的汇编代码，也不影响寄存器的使用，只是插入了一些辅助的指令而已

![image-20210405235244490](https://github.com/swifteen/double_precision/blob/main/images/image-20210405235244490.png?raw=true)

#### gdb打印所有寄存器中的值

通过info all-registers命令可以打印通用寄存器、双精度浮点寄存器d0~d31、单精度浮点寄存器s0~s31的值。使用寄存器的值时，要注意字节序

```shell
(gdb) info all-registers
r0             0xd68cf             878799
r1             0x0                 0
r2             0xd68cf             878799
r3             0x0                 0
r4             0x0                 0
r5             0x105a8             66984
r6             0x103d0             66512
r7             0x0                 0
r8             0x0                 0
r9             0x0                 0
r10            0x76fff000          1996484608
r11            0x7efff494          2130703508
r12            0x7efff510          2130703632
sp             0x7efff480          0x7efff480
lr             0x104f8             66808
pc             0x10504             0x10504 <main()+68>
cpsr           0x20000010          536870928

//以下只列出了使用到的几个双精度浮点寄存器
d5             { u32 = {0x0, 0x41f00000}, u64 = 0x41f0000000000000, f32 = {0x0, 0x1e}, f64 = 0x100000000}
d6             { u32 = {0xffffffff, 0x412ad19f}, u64 = 0x412ad19fffffffff, f32 = {0x0, 0xa}, f64 = 0xd68cf}
d7             { u32 = {0x0, 0xd68cf}, u64 = 0xd68cf00000000, f32 = {0x0, 0x0}, f64 = 0x0}

//以下只列出了使用到的几个单精度浮点寄存器
s11            30                  (raw 0x41f00000)
s12            -nan(0x7fffff)      (raw 0xffffffff)
s13            10.676177           (raw 0x412ad19f)
s14            0                   (raw 0x00000000)
s15            1.23145969e-39      (raw 0x000d68cf)
```

#### 汇编分解

将例子2中的汇编指令进行指令级分解，观察每次指令操作后寄存器中值的变化，进而了解浮点运算的每一步动作。

需要使用的gdb命令：

参考https://stackoverflow.com/questions/2420813/using-gdb-to-single-step-assembly-code-outside-specified-executable-causes-error

```gdb
(gdb) display/i $pc			#告诉gdb显示下一个汇编指令
(gdb) si					#执行下一条汇编指令 step instruction
(gdb) info all-registers	#打印通用寄存器、双精度浮点寄存器d0~d31、单精度浮点寄存器s0~s31的值
```

 在关键位置打好断点后，先输入s，进入打断点的位置，然后重复输入si，查看每次的汇编指令，同时输入info all-registers查看每次执行一条汇编指令后，寄存器中的值。



下面逐步分解上面例子2中对应的汇编指令执行后的寄存器状态，只分析最关键的浮点操作部分汇编

```assembly
	ldr	r3, .L3+16					; 将.L3+16处的值加载到r3寄存器中   代码中.L3+16处实际的值为33800，即将33800保存到r3寄存器中
	str	r3, [fp, #-8]				; 将r3寄存器中的值保存到栈fp-8的位置		对应代码int a = 33800;将局布变量入栈
	ldr	r3, [fp, #-8]				; 将栈fp-8位置的值加载到r3寄存器中
	vmov	s15, r3	@ int			; 将r3寄存器的值(33800)拷贝到s15单精度浮点寄存器中
	vcvt.f64.s32	d6, s15			; 将s15浮点寄存器中32位单精度浮点数转换为64位双精度浮点保存到d6双精度浮点寄存器中(33800.0)
	vldr.64	d5, .L3				    ; 将.L3处的值加载到d5寄存器中 .L3处前32位值为0存为double的低32位，后32位值为1083129856存为double的高32位
	vdiv.f64	d7, d6, d5			; 将d6保存的double值(33800.0)除以d5保存的double值(1000.0)，得到值保存到d7双精度浮点寄存器中
	vldr.64	d6, .L3+8				; 将.L3+8处的值加载到d6双精度浮点寄存器中  代码中.L3+8处前32位的值为0，后32位的值为1087988736
	vmul.f64	d7, d7, d6			; 将d7保存的double值乘以d6保存的double值，得到值保存到d7双精度浮点寄存器中
	vmov	r0, r1, d7				; 将d7双精度浮点寄存器中的低32位的值保存到r0中，将d7双精度浮点寄存器中的高32位的值保存到r1中
	bl	__aeabi_d2lz				; double to long long C-style conversion ,此时r0, r1为输入参数
```

##### 1.初始寄存器状态，r3中保存的值为局布变量a的值33800

   此时所有双精度浮点寄存器和单精度浮点寄存器的值都为0，下面故意没有展示出来，没必要展示。

```assembly
   	ldr	r3, .L3+16					; 将.L3+16处的值加载到r3寄存器中   代码中.L3+16处实际的值为33800，即将33800保存到r3寄存器中
   	str	r3, [fp, #-8]				; 将r3寄存器中的值保存到栈fp-8的位置		对应代码int a = 33800;将局布变量入栈
   	ldr	r3, [fp, #-8]				; 将栈fp-8位置的值加载到r3寄存器中
```

```shell
r0             0x1                 1
r1             0x7efff5e4          2130703844
r2             0x7efff5ec          2130703852
r3             0x8408              33800
r4             0x0                 0
r5             0x105a8             66984
r6             0x103d0             66512
r7             0x0                 0
r8             0x0                 0
r9             0x0                 0
r10            0x76fff000          1996484608
r11            0x7efff494          2130703508
r12            0x7efff510          2130703632
sp             0x7efff480          0x7efff480
lr             0x76e38718          1994622744
pc             0x104d8             0x104d8 <main()+24>
cpsr           0x60000010          1610612752
```
##### 2.将r3寄存器的值(33800)拷贝到s15单精度浮点寄存器中
```assembly
0x104d8 <main()+24>: vmov    s15, r3		
```
```shell
r0             0x1                 1
r1             0x7efff5e4          2130703844
r2             0x7efff5ec          2130703852
r3             0x8408              33800
r4             0x0                 0
r5             0x105a8             66984
r6             0x103d0             66512
r7             0x0                 0
r8             0x0                 0
r9             0x0                 0
r10            0x76fff000          1996484608
r11            0x7efff494          2130703508
r12            0x7efff510          2130703632
sp             0x7efff480          0x7efff480
lr             0x76e38718          1994622744
pc             0x104dc             0x104dc <main()+28>
cpsr           0x60000010          1610612752

d7             { u32 = {0x0, 0x8408}, u64 = 0x840800000000, f32 = {0x0, 0x0}, f64 = 0x0}
s15            4.73638881e-41      (raw 0x00008408)
```
##### 3.将s15浮点寄存器中32位单精度浮点数转换为64位双精度浮点保存到d6双精度浮点寄存器中(33800.0)

```assembly
0x104dc <main()+28>: vcvt.f64.s32    d6, s15 
```
```shell
(gdb) info all-registers
r0             0x1                 1
r1             0x7efff5e4          2130703844
r2             0x7efff5ec          2130703852
r3             0x8408              33800
r4             0x0                 0
r5             0x105a8             66984
r6             0x103d0             66512
r7             0x0                 0
r8             0x0                 0
r9             0x0                 0
r10            0x76fff000          1996484608
r11            0x7efff494          2130703508
r12            0x7efff510          2130703632
sp             0x7efff480          0x7efff480
lr             0x76e38718          1994622744
pc             0x104e0             0x104e0 <main()+32>
cpsr           0x60000010          1610612752

d6             { u32 = {0x0, 0x40e08100}, u64 = 0x40e0810000000000, f32 = {0x0, 0x7}, f64 = 0x8408}
d7             { u32 = {0x0, 0x8408}, u64 = 0x840800000000, f32 = {0x0, 0x0}, f64 = 0x0}

s13            7.01574707          (raw 0x40e08100)
s14            0                   (raw 0x00000000)
s15            4.73638881e-41      (raw 0x00008408)
```
可以看出此时d6寄存器中f64值为0x8408，代表浮点数33800.0
##### 4.将.L3处的值加载到d5寄存器中 .L3处前32位值为0存为double的低32位，后32位值为1083129856存为double的高32位

```assembly
=> 0x104e0 <main()+32>: vldr    d5, [pc, #56]   ; 0x10520 <main()+96> 
```
```shell
(gdb) info all-registers
r0             0x1                 1
r1             0x7efff5e4          2130703844
r2             0x7efff5ec          2130703852
r3             0x8408              33800
r4             0x0                 0
r5             0x105a8             66984
r6             0x103d0             66512
r7             0x0                 0
r8             0x0                 0
r9             0x0                 0
r10            0x76fff000          1996484608
r11            0x7efff494          2130703508
r12            0x7efff510          2130703632
sp             0x7efff480          0x7efff480
lr             0x76e38718          1994622744
pc             0x104e4             0x104e4 <main()+36>
cpsr           0x60000010          1610612752

d5             { u32 = {0x0, 0x408f4000}, u64 = 0x408f400000000000, f32 = {0x0, 0x4}, f64 = 0x3e8}
d6             { u32 = {0x0, 0x40e08100}, u64 = 0x40e0810000000000, f32 = {0x0, 0x7}, f64 = 0x8408}
d7             { u32 = {0x0, 0x8408}, u64 = 0x840800000000, f32 = {0x0, 0x0}, f64 = 0x0}

s11            4.4765625           (raw 0x408f4000)
s12            0                   (raw 0x00000000)
s13            7.01574707          (raw 0x40e08100)
s14            0                   (raw 0x00000000)
s15            4.73638881e-41      (raw 0x00008408)
```
可以看出此时d5寄存器中f64值为0x3e8，代表浮点数1000.0
##### 5.将d6保存的double值(33800.0)除以d5保存的double值(1000.0)，得到值保存到d7双精度浮点寄存器中

```assembly
=> 0x104e4 <main()+36>: vdiv.f64        d7, d6, d5
```

```shell
(gdb) info all-registers
r0             0x1                 1
r1             0x7efff5e4          2130703844
r2             0x7efff5ec          2130703852
r3             0x8408              33800
r4             0x0                 0
r5             0x105a8             66984
r6             0x103d0             66512
r7             0x0                 0
r8             0x0                 0
r9             0x0                 0
r10            0x76fff000          1996484608
r11            0x7efff494          2130703508
r12            0x7efff510          2130703632
sp             0x7efff480          0x7efff480
lr             0x76e38718          1994622744
pc             0x104e8             0x104e8 <main()+40>
cpsr           0x60000010          1610612752

d5             { u32 = {0x0, 0x408f4000}, u64 = 0x408f400000000000, f32 = {0x0, 0x4}, f64 = 0x3e8}
d6             { u32 = {0x0, 0x40e08100}, u64 = 0x40e0810000000000, f32 = {0x0, 0x7}, f64 = 0x8408}
d7             { u32 = {0x66666666, 0x4040e666}, u64 = 0x4040e66666666666, f32 = {0xffffffff, 0x3}, f64 = 0x21}

s11            4.4765625           (raw 0x408f4000)
s12            0                   (raw 0x00000000)
s13            7.01574707          (raw 0x40e08100)
s14            2.72008302e+23      (raw 0x66666666)
s15            3.0140624           (raw 0x4040e666)
```
可以看出此时d7寄存器中f64值为0x21，代表浮点数33.0
##### 6.将.L3+8处的值加载到d6双精度浮点寄存器中  代码中.L3+8处前32位的值为0，后32位的值为1087988736

```assembly
=> 0x104e8 <main()+40>: vldr    d6, [pc, #56]   ; 0x10528 <main()+104>
```

```shell
(gdb) info all-registers
r0             0x1                 1
r1             0x7efff5e4          2130703844
r2             0x7efff5ec          2130703852
r3             0x8408              33800
r4             0x0                 0
r5             0x105a8             66984
r6             0x103d0             66512
r7             0x0                 0
r8             0x0                 0
r9             0x0                 0
r10            0x76fff000          1996484608
r11            0x7efff494          2130703508
r12            0x7efff510          2130703632
sp             0x7efff480          0x7efff480
lr             0x76e38718          1994622744
pc             0x104ec             0x104ec <main()+44>
cpsr           0x60000010          1610612752

d5             { u32 = {0x0, 0x408f4000}, u64 = 0x408f400000000000, f32 = {0x0, 0x4}, f64 = 0x3e8}
d6             { u32 = {0x0, 0x40d96400}, u64 = 0x40d9640000000000, f32 = {0x0, 0x6}, f64 = 0x6590}
d7             { u32 = {0x66666666, 0x4040e666}, u64 = 0x4040e66666666666, f32 = {0xffffffff, 0x3}, f64 = 0x21}

s11            4.4765625           (raw 0x408f4000)
s12            0                   (raw 0x00000000)
s13            6.79345703          (raw 0x40d96400)
s14            2.72008302e+23      (raw 0x66666666)
s15            3.0140624           (raw 0x4040e666)
```
可以看出此时d6寄存器中f64值为0x6590，代表浮点数26000.0
##### 7.d7保存的double值乘以d6保存的double值，得到值保存到d7双精度浮点寄存器中

```assembly
=> 0x104ec <main()+44>: vmul.f64        d7, d7, d6
```

```shell
(gdb) info all-registers
r0             0x1                 1
r1             0x7efff5e4          2130703844
r2             0x7efff5ec          2130703852
r3             0x8408              33800
r4             0x0                 0
r5             0x105a8             66984
r6             0x103d0             66512
r7             0x0                 0
r8             0x0                 0
r9             0x0                 0
r10            0x76fff000          1996484608
r11            0x7efff494          2130703508
r12            0x7efff510          2130703632
sp             0x7efff480          0x7efff480
lr             0x76e38718          1994622744
pc             0x104f0             0x104f0 <main()+48>
cpsr           0x60000010          1610612752

d5             { u32 = {0x0, 0x408f4000}, u64 = 0x408f400000000000, f32 = {0x0, 0x4}, f64 = 0x3e8}
d6             { u32 = {0x0, 0x40d96400}, u64 = 0x40d9640000000000, f32 = {0x0, 0x6}, f64 = 0x6590}
d7             { u32 = {0xffffffff, 0x412ad19f}, u64 = 0x412ad19fffffffff, f32 = {0x0, 0xa}, f64 = 0xd68cf}

s11            4.4765625           (raw 0x408f4000)
s12            0                   (raw 0x00000000)
s13            6.79345703          (raw 0x40d96400)
s14            -nan(0x7fffff)      (raw 0xffffffff)
s15            10.676177           (raw 0x412ad19f)
```
可以看出此时d7寄存器中f64值为0xd68cf，代表浮点数878799.0

其中高32位为0x412ad19f，低32位为0xffffffff，见下图分解

![image-20210412223825458](https://github.com/swifteen/double_precision/blob/main/images/image-20210412223825458.png?raw=true)

##### 8.将d7双精度浮点寄存器中的低32位的值保存到r0中，将d7双精度浮点寄存器中的高32位的值保存到r1中

```assembly
=> 0x104f0 <main()+48>: vmov    r0, r1, d7
```

```shell
(gdb) info all-registers
r0             0xffffffff          4294967295
r1             0x412ad19f          1093325215
r2             0x7efff5ec          2130703852
r3             0x8408              33800
r4             0x0                 0
r5             0x105a8             66984
r6             0x103d0             66512
r7             0x0                 0
r8             0x0                 0
r9             0x0                 0
r10            0x76fff000          1996484608
r11            0x7efff494          2130703508
r12            0x7efff510          2130703632
sp             0x7efff480          0x7efff480
lr             0x76e38718          1994622744
pc             0x104f4             0x104f4 <main()+52>
cpsr           0x60000010          1610612752

d5             { u32 = {0x0, 0x408f4000}, u64 = 0x408f400000000000, f32 = {0x0, 0x4}, f64 = 0x3e8}
d6             { u32 = {0x0, 0x40d96400}, u64 = 0x40d9640000000000, f32 = {0x0, 0x6}, f64 = 0x6590}
d7             { u32 = {0xffffffff, 0x412ad19f}, u64 = 0x412ad19fffffffff, f32 = {0x0, 0xa}, f64 = 0xd68cf}

s11            4.4765625           (raw 0x408f4000)
s12            0                   (raw 0x00000000)
s13            6.79345703          (raw 0x40d96400)
s14            -nan(0x7fffff)      (raw 0xffffffff)
s15            10.676177           (raw 0x412ad19f)
```
其中r0为浮点数d7寄存器中的低32位0xffffffff，r1为浮点数d7寄存器中的高32位0x412ad19f
##### 9.double to long long C-style conversion ,此时r0, r1为输入参数

```assembly
=> 0x104f4 <main()+52>: bl      0x10538 <__fixdfdi>
```

```shell
(gdb) info all-registers
r0             0xffffffff          4294967295
r1             0x412ad19f          1093325215
r2             0x7efff5ec          2130703852
r3             0x8408              33800
r4             0x0                 0
r5             0x105a8             66984
r6             0x103d0             66512
r7             0x0                 0
r8             0x0                 0
r9             0x0                 0
r10            0x76fff000          1996484608
r11            0x7efff494          2130703508
r12            0x7efff510          2130703632
sp             0x7efff480          0x7efff480
lr             0x104f8             66808
pc             0x10538             0x10538 <__fixdfdi>
cpsr           0x60000010          1610612752

d5             { u32 = {0x0, 0x408f4000}, u64 = 0x408f400000000000, f32 = {0x0, 0x4}, f64 = 0x3e8}
d6             { u32 = {0x0, 0x40d96400}, u64 = 0x40d9640000000000, f32 = {0x0, 0x6}, f64 = 0x6590}
d7             { u32 = {0xffffffff, 0x412ad19f}, u64 = 0x412ad19fffffffff, f32 = {0x0, 0xa}, f64 = 0xd68cf}

s11            4.4765625           (raw 0x408f4000)
s12            0                   (raw 0x00000000)
s13            6.79345703          (raw 0x40d96400)
s14            -nan(0x7fffff)      (raw 0xffffffff)
s15            10.676177           (raw 0x412ad19f)
```

##### 10.将d7寄存器的高32位保存到r0寄存器中，将d7寄存器的低32位保存到r1寄存器中

    从以下寄存器中的值可以看出，__fixdfdi函数执行后，将转换后的long long 值保存在了d7寄存器中，然后d7寄存器中的值转移到r0和r1，从而得到double转换为long long 的结果

```shell
=> 0x10538 <__fixdfdi>: vmov    d7, r0, r1
```

```shell
(gdb) info all-registers
r0             0xd68cf             878799
r1             0x0                 0
r2             0x7efff5ec          2130703852
r3             0x8408              33800
r4             0x0                 0
r5             0x105a8             66984
r6             0x103d0             66512
r7             0x0                 0
r8             0x0                 0
r9             0x0                 0
r10            0x76fff000          1996484608
r11            0x7efff494          2130703508
r12            0x7efff510          2130703632
sp             0x7efff480          0x7efff480
lr             0x104f8             66808
pc             0x104f8             0x104f8 <main()+56>
cpsr           0x20000010          536870928

d5             { u32 = {0x0, 0x41f00000}, u64 = 0x41f0000000000000, f32 = {0x0, 0x1e}, f64 = 0x100000000}
d6             { u32 = {0xffffffff, 0x412ad19f}, u64 = 0x412ad19fffffffff, f32 = {0x0, 0xa}, f64 = 0xd68cf}
d7             { u32 = {0x0, 0xd68cf}, u64 = 0xd68cf00000000, f32 = {0x0, 0x0}, f64 = 0x0}

fpscr          0x20000010          536870928
s0             0                   (raw 0x00000000)
s1             0                   (raw 0x00000000)
s2             0                   (raw 0x00000000)
s3             0                   (raw 0x00000000)
s4             0                   (raw 0x00000000)
s5             0                   (raw 0x00000000)
s6             0                   (raw 0x00000000)
s7             0                   (raw 0x00000000)
s8             0                   (raw 0x00000000)
s9             0                   (raw 0x00000000)
s10            0                   (raw 0x00000000)
s11            30                  (raw 0x41f00000)
s12            -nan(0x7fffff)      (raw 0xffffffff)
s13            10.676177           (raw 0x412ad19f)
s14            0                   (raw 0x00000000)
s15            1.23145969e-39      (raw 0x000d68cf)
```


# 二、总结

1、不要将浮点运算得到的值，用来做精确的比较后去做某事

2、计算机中的浮点数也是用二进制表示的，不是所有浮点数都能用二进制精确的表示。

比如0.2，二进制小数0.00110011，对应的十进制小数0.19921875

- 使用二进制表达十进制的小数时，某些数字无法被有限位的二进制小数表示；
- 单精度和双精度的浮点数只包括 7 位或者 15 位的有效小数位，存储需要无限位表示的小数时只能存储近似值；

# 三、参考知识

## gcc参数

### 使用`-save-temps`参数产生所有的中间步骤的文件

`-save-temps`可以做4,5,6步骤的工作。通过这个参数，所有中间阶段的文件都会存储在当前文件夹中，注意它也会产生可执行文件。

```
$ gcc -save-temps main.c

$ ls
a.out  main.c  main.i  main.o  main.s
```

从例子中我们可以看到各个中间文件以及可执行文件。

### -Wl

> 用来向链接器指定参数
>  `gcc -Wl, --wrap,malloc -Wl,--wrap,free -o int1 A.o B.o`
>  则链接器的参数会有两个 `--wrap malloc --wrap free` 可以看到`--wrap,free` 变成了`--wrap free` 也就是`,` 变成了空格

## objdump反汇编

```shell
pi@raspberrypi:~/shared/boox $ objdump -d test_double2.o

test_double2.o:     file format elf32-littlearm


Disassembly of section .text:

00000000 <main>:
   0:   e92d4800        push    {fp, lr}
   4:   e28db004        add     fp, sp, #4
   8:   e24dd010        sub     sp, sp, #16
   c:   e59f3048        ldr     r3, [pc, #72]   ; 5c <main+0x5c>
  10:   e50b3010        str     r3, [fp, #-16]
  14:   e51b3010        ldr     r3, [fp, #-16]
  18:   e1a02003        mov     r2, r3
  1c:   e1a03002        mov     r3, r2
  20:   e1a03083        lsl     r3, r3, #1
  24:   e0833002        add     r3, r3, r2
  28:   e1a03103        lsl     r3, r3, #2
  2c:   e0833002        add     r3, r3, r2
  30:   e1a03083        lsl     r3, r3, #1
  34:   e1a02003        mov     r2, r3
  38:   e3a03000        mov     r3, #0
  3c:   e14b20fc        strd    r2, [fp, #-12]
  40:   e14b20dc        ldrd    r2, [fp, #-12]
  44:   e59f0014        ldr     r0, [pc, #20]   ; 60 <main+0x60>
  48:   ebfffffe        bl      0 <printf>
  4c:   e3a03000        mov     r3, #0
  50:   e1a00003        mov     r0, r3
  54:   e24bd004        sub     sp, fp, #4
  58:   e8bd8800        pop     {fp, pc}
  5c:   00008408        .word   0x00008408
  60:   00000000        .word   0x00000000
```

## nm查看符号表

```bash
pi@raspberrypi:~/shared/boox $ nm test_double2.o        
         U __aeabi_unwind_cpp_pr1
00000000 T main
         U printf
```

## fp寄存器解释

The frame pointer ($fp) points to the start of the stack frame and does not move for the duration of the subroutine call. This points to the base of the stack frame, and the parameters that are passed in to the subroutine remain at a constant spot relative to the frame pointer.

![img](http://www.linuxidc.com/upload/2013_03/130320150468341.png)

## 寄存器用途

The column labeled “Restore Contents?” shows whether the function needs to ensure that the value in the register is the same when it returns to the calling function as it contained when the this function was called.

| Register | Synonym（相同） | Restore Contents | Purpose                      |
| -------- | --------------- | ---------------- | ---------------------------- |
| `r0`     |                 | N                | argument/results             |
| `r1`     |                 | N                | argument/results             |
| `r2`     |                 | N                | argument/results             |
| `r3`     |                 | N                | argument/results             |
| `r4`     |                 | Y                | local variable               |
| `r5`     |                 | Y                | local variable               |
| `r6`     |                 | Y                | local variable               |
| `r7`     |                 | Y                | local variable               |
| `r8`     |                 | Y                | local variable               |
| `r9`     |                 | Y                | depends on platform standard |
| `r10`    |                 | Y                | local variable               |
| `r11`    | `fp`            | Y                | frame pointer/local variable |
| `r12`    | `ip`            | N                | intra-procedure-call scratch |
| `r13`    | `sp`            | Y                | stack pointer                |
| `r14`    | `lr`            | N                | link register                |
| `r15`    | `pc`            | N                | program counter              |

## 汇编指令

### BL

![image-20210406230606785](https://github.com/swifteen/double_precision/blob/main/images/image-20210406230606785.png?raw=true)

### LDR

![image-20210406230652114](https://github.com/swifteen/double_precision/blob/main/images/image-20210406230652114.png?raw=true)

### STR

![image-20210406230723619](https://github.com/swifteen/double_precision/blob/main/images/image-20210406230723619.png?raw=true)

### LSL、LSR

![image-20210406230817687](https://github.com/swifteen/double_precision/blob/main/images/image-20210406230817687.png?raw=true)

### VMOV

![image-20210404233915113](https://github.com/swifteen/double_precision/blob/main/images/image-20210404233915113.png?raw=true)

### VCVT

![image-20210404235040384](https://github.com/swifteen/double_precision/blob/main/images/image-20210404235040384.png?raw=true)

### VLDR

![image-20210404235559606](https://github.com/swifteen/double_precision/blob/main/images/image-20210404235559606.png?raw=true)

参考https://www.codenong.com/cs106577545/

https://bob.cs.sonoma.edu/IntroCompOrg-RPi/sec-instrs-1.html#instr-ldr

# 四、参考链接

## 浮点数转换

https://blog.csdn.net/weixin_43955216/article/details/107385732?utm_medium=distribute.pc_relevant.none-task-blog-baidujs_title-0&spm=1001.2101.3001.4242

## 汇编指令

https://bob.cs.sonoma.edu/IntroCompOrg-RPi/sec-stack-manage.html

## gdb调试

https://bob.cs.sonoma.edu/IntroCompOrg-RPi/sec-gdb1.html#p-342

https://bob.cs.sonoma.edu/IntroCompOrg-RPi/sec-gdb2.html

You can use `stepi` or `nexti` (which can be abbreviated to `si` or `ni`) to step through your machine code.

The most useful thing you can do here is `display/i $pc`, before using `stepi` as already suggested in R Samuel Klatchko's answer. This tells gdb to disassemble the current instruction just before printing the prompt each time; then you can just keep hitting Enter to repeat the `stepi` command.

https://stackoverflow.com/questions/2420813/using-gdb-to-single-step-assembly-code-outside-specified-executable-causes-error