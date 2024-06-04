# 位置无关代码分析报告
题目描述：
1. 依据“7.12 位置无关代码” 的内容

2. 在希冀平台上编译main.c和sum.c（在课件下载“csapp3e-code-all ”  tar 文件解压缩之后的 code/link/ 目录下），分析main.o和prog中对array和sum的引用是如何被重定位的 

3. 在希冀平台上编译 libvector. so，分析动态链接库中的函数是如何被重定位的 
    code/13-linking/libvector.so
    提示：readelf；objdump -dx libvector. so 

4. 完成一份分析报告
<br>

# 目录
**一、分析main.o和prog中对array和sum的引用是如何被重定位的
二、分析动态链接库中的函数是如何被重定位的**
<br>
<br>

# Part 1 <br> **分析main.o和prog中对array和sum的引用是如何被重定位的**

main.c与sum.c文件代码如下图所示![](https://md.0x7f.app/uploads/8572b508-551b-4c11-9e6c-c25a2bba4a62.png)


①分别编译main.c和sum.c生成相应的可重定位目标文件，通过ld指令对相应目标文件进行链接。

```
linux> gcc -c -fno-pic main.c
linux> gcc -c -fno-pic sum.c
linux> ld -o prog main.o sum.o
```
其中指令
```
ld -o prog main.o sum.o
```
代表着链接时main.o依赖于sum.o生成定位后的目标文件，进而生成可执行文件prog。

②使用objdump -dx指令分别反汇编main.o以及prog，查看生成的反汇编代码。
![](https://md.0x7f.app/uploads/a780a376-bc05-460a-8bae-ad08a025cc14.png)


由上图可以看出在main.o中可以看出对于array数组目标文件中使用了绝对寻址的方式，而对于sum函数则使用了PC相对寻址方式。

对于array数组的绝对寻址根据公式\begin{equation}*refptr=(unsigned)(ADDR(r.symbol)+r.addend)\end{equation}

进行验证。

查看.symtab条目
![](https://md.0x7f.app/uploads/f32fc610-ddfd-4865-9441-1fb090ddf85a.png)


其中array地址为0x601000，由main.o文件可知addend为0，相加得到0x601000，即为绝对地址与prog文件中汇编代码bf 00 10 60相符。
##
<br>

![](https://md.0x7f.app/uploads/25eee2e7-e863-4a94-b56a-efafd34ddbc9.png)


对于sum函数的PC相对寻址方式可以通过\begin{equation}*refptr=(unsigned)(ADDR(r.symbol)+r.addend -refaddr)
\end{equation}进行验证，其中$refaddr=ADDE(s)+r.offset$ 。
<br>


根据公式得到
***refptr=0x400107-0x4-(0x4000e8+0x13)=0x8**。可以看到汇编代码改为e8 08，说明计算正确，同时注意到 **0x400107=0x4000ff+0x8**，即绝对地址等于调用时RIP寄存器指向的地址加上相对地址，与预期结论相同。

# Part 2 <br> **分析动态链接库中的变量与函数是如何被重定位的**

书中将addvec.c与multivec.c两个文件生成动态链接文件libvector. so，链接时让main.c依赖于该动态链接库生成可执行文件，其中addvec.c中定义了addcnt全局变量。我们的目标是分析addcnt变量以及addvec函数在prog2运行时是如何被重定位的。


## 一、分析全局变量addcnt是如何被重定位的


```
linux> gcc -shared -fpic -Og -o libvector.so addvec.c multvec.c
linux> gcc -Og -o prog2 main2.c ./libvector.so
```
首先根据addvec.c与multvec.c函数生成动态链接库libvector. so
再让main函数与动态链接库链接生成prog2可执行文件
![](https://md.0x7f.app/uploads/784e4013-9082-4b09-b1f5-067a1e5e9728.png)

查看addvec函数的汇编代码，注意到语句
``` =
mov     0x2009cf(%rip)，%r8      #0x7ffff7dd1fe0
mov     (%r8)，%eax
add     $0x1，%eax
mov     %eax，(%r8)
```

根据所学到的知识，动态链接在程序运行时进行，修改GOT表（特别说明：这个GOT表是libvector.so的GOT表）对应位置的值（这个值即&addcnt），再通过汇编语言修改寄存器的值对addcnt进行修改。

回到汇编语句，分析每一条语句的功能：

**1、语句1：mov     0x2009cf(%rip)，%r8      #0x7ffff7dd1fe0 将GOT表对应位置的值取出。**

分析：输入语句
```
linux> objdump -dx libvector.so
```
查看libvector.so的EIF表以及反汇编代码
![](https://md.0x7f.app/uploads/e441e649-5894-43b9-adfd-74dedf6078c5.png)
由于.so文件为可重定位目标文件，ELF表中alloc的地址均为相对地址。
注意到.got对应的相对地址是0x200fd0

![](https://md.0x7f.app/uploads/bedbd1d4-95ae-42e1-808b-3d9063c23827.png)
语句1将相对地址为0x200fe0的值取出，即GOT[2]。

![](https://md.0x7f.app/uploads/0642ed17-80ac-4748-9df4-860062f95dfa.png)
再回到运行时的程序，相对地址为0x200fe0对应运行时的绝对地址0x7ffff7dd1fe0，其存储的值为addcnt的地址，即0x7ffff7dd2024。

<br>

**2、语句2：mov  (%r8)，%eax 将addvec的值取出给eax寄存器。**

分析：由于GOT表地址和addvec地址接近，故共用一张截图，可以看到0x7ffff7dd2024对应的值为0，说明此时addcnt的值为0。
![](https://md.0x7f.app/uploads/ac6de4f3-a47f-4b8f-b1ab-8c1930c44a43.png)


<br>

**3、语句3、4：add     $0x1，%eax；mov     %eax，(%r8)修改eax寄存器的值并改变addcnt的值。**

分析：（%r8)即addcnt的值，eax寄存器的值等于addcnt，修改eax的值再通过mov指令修改（%r8）即可改变addcnt的值。

![](https://md.0x7f.app/uploads/4eeec677-f3cd-44c7-897b-11e35777fb8e.png)

如上图所示，可以看到0x7ffff7dd2024处的值已经变为1。即addcnt的值加1

<br>

## 二、分析全局函数addvec是如何被重定位的
由于默认-fpic选项，汇编代码的地址均为相对地址，在运行时才会加载为绝对地址，addvec作为libvector.so这个动态链接库的函数，在可执行文件中应该通过PIC函数调用的方式间接调用。


为了体现两次调用的不同过程，修改main2.c代码
```c=
/* main2.c */
/* $begin main2 */
#include <stdio.h>
#include "vector.h"

int x[2] = {1, 2};
int y[2] = {3, 4};
int z[2];

int main()
{
    addvec(x, y, z, 2);//第一次调用
    addvec(x, y, z, 2);//第二次调用
    printf("z = [%d %d]\n", z[0], z[1]);
    return 0;
}
/* $end main2 */
```
然后重新编译main2.c源文件，由于unbuntu默认在加载程序时对可能调用的函数进行提前解析，因此我们需要加上-z lazy编译选项阻止这种行为的发生。

输入指令
```
linux> gcc -Og -z lazy main2.c ./libvector.so -o prog2
```
这样就顺利生成了可执行文件prog2

![](https://md.0x7f.app/uploads/0bae7215-cb4f-46c1-889c-3c0fea217982.png)

根据所学内容，如上图所示，当程序运行并第一次调用addvec时，进入该程序对应的plt表，但是此时并不会直接调用函数因为此时函数还没有加载到内存中，所以程序会调用dynamic linker将addvec代码段加载到内存中再进行调用。而当程序第二次调用addvec函数时，此时GOT表内对应的值（即*GOT[i]）已经修改为addvec的地址，因此会直接调用addvec函数。带着预期结论接下来进行分析。

**具体分析过程如下：**
首先确定一下prog2中addvec@plt函数对应plt表的位置。

![](https://md.0x7f.app/uploads/acc9d91d-9de2-41ff-92b6-5d67c5c1b323.png)
可以看到addvec@plt对应PLT[1]

再确定一下addvec函数在加载完成后对应GOT表的位置。
![](https://md.0x7f.app/uploads/91085afa-0471-4670-beab-5aba09c7063c.png)



![](https://md.0x7f.app/uploads/9452e487-6968-4958-a19a-df3dc22c47f1.png)
GOT表起始位置为0x200fd8，addvec函数位置为0x201018，对应GOT[8]。

## 
<br>

**接下来单步运行程序进行具体分析**

![](https://md.0x7f.app/uploads/52292f48-56c7-48eb-ac33-d58ba27f1df2.png)
通过main函数汇编代码可以看到addvec@plt函数被调用了两遍。


第一遍进入addvec@plt时，打印GOT表内容。

![](https://md.0x7f.app/uploads/cdeb1945-2309-4164-9751-91b234cf66d9.png)



![](https://md.0x7f.app/uploads/8332d92e-7013-4a93-b8d4-ab89d3100a56.png)
刚刚计算得到了addvec函数对应GOT[8]可以看到此时GOT[8]=0x555555554646，就是rip的下一条指令，程序继续运行然后即将调用动态链接器。

![](https://md.0x7f.app/uploads/4b9f6ec3-8735-4e87-a83b-b076fb67172f.png)

![](https://md.0x7f.app/uploads/9b2dfc9f-c509-448c-9fa3-ffe1e841f459.png)
如上图所示，addvec@plt函数第三条指令jmpq使程序跳转到PLT[0]，PLT[0]中jmpq *0x2009d4(%rip)跳转到GOT[7]对应的值，即动态链接器的地址。


接下来动态链接器进行动态链接（执行了一长串汇编代码），将addvec函数加载入内存并调用了addvec函数，这下便完成了第一次调用。
![](https://md.0x7f.app/uploads/86ba11c8-3fe9-45d4-8eb8-64c88a21c849.png)

![](https://md.0x7f.app/uploads/566562b5-a5e4-4068-9b2d-2f35806b437a.png)
此时打印GOT表的内容可以发现**GOT[8]已经修改为刚刚加载进来的addvec函数对应的地址即0x7ffff7bd160a**。
## 
<br>

第二次调用就很简单了，程序跳转到PLT[1]，而第一条jmp指令立即使程序跳转到addvec函数,**可以看到addvec函数对应的地址确实是0x7ffff7bd160a**。

![](https://md.0x7f.app/uploads/c30ae821-1aec-4ce3-aa14-5e5993beb751.png)

![](https://md.0x7f.app/uploads/496c2569-b6bb-4db6-9847-3ce9481e9302.png)



可以看出整个分析过程与书中内容基本相符，只是对应@plt代码段在PLT表中的下标以及addvec函数地址在GOT表中的下标有一定出入。

