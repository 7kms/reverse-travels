# GCC内联汇编

## 基本内联

格式: `asm("汇编代码")`,其中`asm`和`__asm__`都是有效的

```c

asm("movl %ecx %eax"); /* 将ecx寄存器的内容送入eax */

__asm__("movb %bh (%eax)"); /* 将bh的一个字节数据 移至eax寄存器指向的内存 */

```
如果指令多余一条, 可以每个一行, 并用双引号标识. 同时为每条指令添加`\n\t`后缀(因为gcc将每一条当做字符串发给as(GAS即GNU汇编器), 并且通过使用换行符/制表符发送正确格式化后的行给汇编器).

```s
__asm__ ("movl %eax, %ebx/n/t"
         "movl $56, %esi/n/t"
         "movl %ecx, $label(%edx,%ebx,$4)/n/t"
         "movb %ah, (%ebx)");
```

## 扩展汇编

在基本内联汇编中, 只有指令. 在扩展汇编中, 我们可以同时指定操作数. 允许我们指定输入寄存器, 输出寄存器以及修饰寄存器列表.

格式:
```s
asm(汇编程序模板
  :输出操作数 /*可选*/
  :输入操作数 /*可选*/
  :修饰寄存器列表 /*可选*/
);
```
每一个操作数由一个操作数约束字符串所描述, 其后紧接一个括弧括着的C表达式. 冒号用于将汇编程序模板和第一个输出操作数分开, 另一个(冒号)用于将后一个输出操作数和第一个输入操作数分开(如果存在的话).逗号用于分离每一个组内的操作数. 总操作数的数目限制在10个, 或者机器描述中的任何指令格式中最大操作数数目, 以较大者为准.


如果没有输出操作数但是存在输入操作数, 则必须将两个连续的冒号放置于输出操作数原本会放置的地方

示例:
```s
asm("cld \n\t"
    "rep \n\t"
    "stosl"
    : /*无输出寄存器*/
    : "c" (count), "a" (fill_value), "D" (dest)
    : "%ecx", "%edi"
)
```
解释: 以上的内联汇编是将`fill_value`值连续`count`此拷贝到寄存器`edi`所指位置.(每执行`stosl`一次, 寄存器edi的值会递增或递减, 取决于是否设置了`direction`标志, 因此以上代码实则初始化一个内存块)

示例:
```c
int a =10, b;
asm("movl %1, %%eax;
     movl %%eax, %0;"
     :"=r"(b)   /* 输出 */
     :"r"(a)    /* 输入 */
     :"%eax"    /* 修饰寄存器 */
);

```
解释: 以上代码所做的是使用汇编指令使`b`变量的值等于`a`变量的值.
1. `b`为输出操作数, 用`%0`引用. 并且`a`为输入操作数,用`%1`引用.
2. `r`为操作数约束. `r`告诉GCC可以使用任一寄存器存储操作数.输出操作数的约束应该有一个约束修饰符`=`. 这修饰符表明它是一个制度的输出操作数
3. 寄存器名字以`2个%`为前缀. 有利于GCC区分操作数和寄存器. 操作数以一个%为前缀
4. 第3个冒号之后的修饰寄存器`%eax`用于告诉GCC`%eax`的值将会在`asm`内部被修改,所以GCC将不会使用此寄存器存储任何其他值.

当`asm`执行完毕, `b`变量会映射到更新的值, 因为它被指定为输出操作数. 或者说, `asm`内`b`变量的修改应该会被映射到`asm`外部.

## 每个域的解释
1. 汇编程序模板

- 汇编程序模板包含了被插入到c程序的汇编指令集.
- 格式: 每条指令用双引号圈起, 或者整个指令组用双引号圈起. 
- 格式: 每条指令应以分界符结尾,("\n")和分号(;), 其中"\n"可以紧随一个制表符"\t".

2. 操作数

c表达式用作`asm`内的汇编指令操作数. 每个操作数前面是以双引号圈起的操作数约束. 约定字符串主要用于决定操作数的寻址方式, 同时也用于指定使用的寄存器.如果使用的操作数多余一个, 每一个操作数用逗号隔开

- 输入操作数: 在引号内有一个约束修饰符, 其后紧随一个用于表示操作数的C表达式. `操作数约束(C 表达式)`
- 输出操作数: 在输入操作数的格式基础上, 还有一个额外的修饰符. 输出操作数表达式必须为左值.
- 操作数编号方式: 在汇编程序模板中, 每个操作数用数字引用. 如果共有n个操作数(包括输入和输出操作数), 那么第一个输出操作数编号为0, 逐项递增, 并且最后一个输入操作数编号为n-1.

示例: 求一个数的5次方结果(使用lea指令)
```c
asm("leal (%1,%1,4), %0"
    : "=r"(five_times_x)
    : "r" (x)
);
```
上述代码输入为x, 没有指定使用的寄存器. GCC将会选择一些输入寄存器, 一个输出寄存器, 来做预期的工作.

如果想要输入和输出放在同一个寄存器里, 可以通过指定合适的约束来实现
```c
asm("leal (%0,%0,4), %0"
    : "=r"(five_times_x)
    : "0" (x)
);
```
上述代码输出和输出操作数位于同一个寄存器. 但是无法得知是哪个寄存器.

在前两个示例中,GCC决定了寄存器并且它知道发生了什么改变, 在最后一个示例我们不必将`ecx`添加到修饰寄存器列表, gcc知道它表示x(所以它就不被当做修饰寄存器了).
```c
asm("leal (%%ecx,%%ecx,4), %%ecx"
    : "=c"(x)
    : "c" (x)
);
```


3. 修饰寄存器列表

汇编函数内第三个`:`之后的域代表修饰寄存器列表. 这可以通知gcc我们将会自己使用和修改这些寄存器, 这样gcc就不会假设存入这些寄存器的值是有效的.不用在这个列表里列出输入, 输出寄存器. 因为gcc知道asm使用了它们(它们已经被显式的指定为约束). 如果指令隐式或显式地使用了任何其他寄存器(并且寄存器没有出现在输出或输出约束列表里), 那么就需要在修饰寄存器列表中指定这些寄存器.

如果汇编程序指令可以修改条件码寄存器(`cc`), 则必须将`cc`添加进修饰寄存器列表

如果汇编程序指令以不可预测的方式修改了内存, 则需要将`memory`添加进修饰寄存器列表.这可以使GCC不会在汇编指令间保持缓存于寄存器的内存值.如果被影响的内存不存在汇编的输入或输出列表中, 则必须添加`volatile`关键词.

模板内的多指令示例: 假设子例程_foo接受寄存器`eax`和`ecx`里的参数.

```c
asm("movl %0, %%eax;
    movl %1, %%ecx;
    call _foo"
    : /*no outputs*/
    : "g"(from), "g"(to)
    : "eax","ecx"
  );
```

4. Volatile

如果汇编语句必须在我们放置它的地方执行, 则将关键词`volatile`放置在`asm`和`()`之间.即`asm volatile (...:...:...:...);`可以防止汇编语句被移动,删除或者其他操作.

如果担心发生冲突, 可以使用`__volatile__`.如果汇编只是用于一些计算并且没有任何副作用, 不使用`volatile`会更好. 不使用`volatile`可以帮助gcc优化代码并使代码更漂亮.

## 关于约束

约束用于表明一个操作数是否可以位于寄存器和位于哪种寄存器;
操作数是否可以为一个内存引用和哪种地址;
操作数是否可以为一个立即数和它可能的取值范围(即值的范围).

### 常用约束

在许多的约束中, 只有少部分是常用的

1. 寄存器操作数约束
当使用这种约束指定操作数时, 他们存储在通用寄存器(GPR)中.

示例:
```c
asm("movl %%eax,%0 /n" :"=r"(myval));
```
变量`myval`保存在寄存器中, 寄存器`eax`的值被送入到该寄存器中, 并且`myval`的值从寄存器更新到了内存.当指定`r`约束是, gcc可以将变量保存在任何可用的GPR中. 要指定寄存器, 必须使用特定寄存器约束直接指定寄存器的名字.

```s
+---+--------------------+
| r |    Register(s)     |
+---+--------------------+
| a |   %eax, %ax, %al   |
| b |   %ebx, %bx, %bl   |
| c |   %ecx, %cx, %cl   |
| d |   %edx, %dx, %dl   |
| S |   %esi, %si        |
| D |   %edi, %di        |
+---+--------------------+

```

2. 内存操作约束

当操作数位于内存时, 任何对它们的操作将直接发生在内存位置, 这与寄存器相反(首先将值存储在要修改的寄存器中, 然后将它写回到内存位置). 寄存器约束通常用于一个指令必须使用它们或者它们可以大大提高处理速度的地方.当需要在`asm`内更新一个c变量, 而又不想使用寄存器去保存它的值, 使用内存最为有效.

示例: IDTR寄存器的值存储于内存位置loc处

```c
asm("sidt %0 \n" ::"m"(loc));
```

3. 匹配(数字)约束

在某些情况下, 一个变量可能既充当输入操作数, 也充当输出操作数.

```c
asm("incl %0" :"=a"(var) :"0"(var));
```
在此示例中, 寄存器`%eax`即作为输入变量, 也作输出变量. `var`输入被读进`%eax`, 并且等递增后更新的`%eax`再次被存储进`var`.这里的`0`用于指定与`第0个输出变量相同的约束`(它指定var输出实例只被存储在`%eax`中)

使用匹配约束的最重要的意义在于它们可以有效地使用可用寄存器.

其他约束

| 约束 | 作用 |
| ---- | ---- |
| m | 允许一个内存操作数, 可以使用基础普遍支持的任一种地址 |
| o | 允许一个内存操作数, 但只有当地址是可偏移的. 即该地址加上一个小的偏移量可以得到一个有效地址 |
| V | 一个不允许偏移的内存操作数. 即任何适合`m`约束而不适合`o`的约束 |
| i | 允许一个(带有常量)的立即整型操作数. 这包括其值仅在汇编时期知道的符号常量 |
| n | 允许一个带有已知数字的立即整型操作数. 许多系统不支持汇编时期的常量, 因为操作数少于一个字宽.对于此种操作数, 约束应该使用`n`而不是`i`|
| g | 允许任一寄存器, 内存或者立即整型操作数, 不包括通用寄存器之外的寄存器 |

以下约束为x86特有

| 约束 | 作用 |
| ---- | ---- |
| r | 寄存器操作数约束, 具体对应哪一个寄存器可以查看上文中的表格 |
| q | 寄存器a,b,c或者d |
| I | 范围从0-31的常量(对于32位位移) |
| J | 范围从0-63的常量(对于64位位移) |
| K | 0xff |
| L | 0xffff |
| M | 0,1,2或3(lea指令的位移) |
| N | 范围从0-255的常量(对于out指令) |
| f | 浮点寄存器 |
| t | 第一个(栈顶)浮点寄存器 |
| u | 第二个浮点寄存器 |
| A | 指定a或d寄存器.这主要用于想要返回64位整型数, 使用`d`寄存器保存最高有效位和`a`寄存器保存最低有效位 |

## 约束修饰符

常用的约束修饰符:

1. `=`: 表示对于这条指令, 操作数为只写的; 旧值会被忽略并被输出数据所替换
2. `&`: 表示这个操作数为一个早期改动的操作数, 其在该指令完成前通过使用输入操作数被修改了. 因此这个操作数不可以位于一个被用作输出操作数或任何内存地址部分的寄存器. 如果在旧值被写入之前它仅用作输入而已, 一个输入操作数可以为一个早期改动操作数


示例: 两个数相加

```c
#include <stdio.h>

int main(void){
  int foo = 10, bar = 15;
  __asm__ __volatile__("addl %%ebx ,%%eax"
                        :"=a"(foo)
                        :"a"(foo),"b"(bar)
                        );
  printf("foo+bar=%d\n", foo);
  return 0;
}
```
gcc将foo存放于`%eax`, 将bar存放于`%ebx`, 同时也在`%eax`中存放结果.`=`符号表示它是一个输出寄存器.

示例: 将一个整数加到一个变量

```c
__asm__ __volatile__(
  "lock         ;\n"
  "add %1,%0    ;\n"
  : "=m"(my_var)
  : "ir"(my_int),"m"(my_var)
  : /* 无修饰寄存器列表 */
);
```
这是一个原子加法. 为了移出原子性, 我们可以移出指令`lock`. 在输出域中,`"=m"`表明myvar是一个输出且位于内存. `ir`表明myint是一个整型, 并应该存在于其他寄存器. 没有寄存器位于修饰寄存器列表中.

示例:
```c
__asm__ __volatile__(
  "decl %0; sete %1"
  : "=m"(my_var), "=q"(cond)
  : "m" (my_var)
  : "memory"
);
```
my_var的值减1, 并且如果结果的值为0, 则变量cond置1. 可以通过将指令`"locl;\n\t"`添加为汇编目标的第一条指令以增加原子性.

以类似的方式, 为了增加`my_var`, 可以使用`incl %0`.

注意:
- my_var是一个存储于内存的变量
- cond位于寄存器eax,ebx,ecx,edx中的任何一个, 约束"=q"保证了这一点
- memory位于修饰寄存器列表中, 也就是说, 代码将改变内存中的内容

示例: 如何置1或清0寄存器中的一个比特位

```c
__asm__ __volatile__(
  "btsl %1,%0"
  :"=m"(ADDR)
  :"Ir"(pos)
  :"cc"
);
```
ADDR变量(一个内存变量)的`pos`位置上的比特被设置为1. 可以使用`btrl`来清除由`btsl`设置的比特位.pos的约束"Ir"表明pos位于寄存器, 并且他的值为0-31(x86相关约束).也就是说, 我们可以设置/清除ADDR变量上第0到31位的任一比特位. 因为条件码会改变, 所以将`cc`添加进修饰寄存器列表

示例: 字符串拷贝
```c
static inline char *strcpy(char *dest, const char *src)
{
  int d0, d1,d2;
  __asm__ __valotile__(
    "1:/tlodsb \n\t"
    "stosb \n\t"
    "testb %%al, %%al \n\t"
    "jne 1b"
    : "=&S"(d0), "=&D"(d1), "=&a"(d2)
    : "0"(src),"1"(dest)
    : "memory"
  );
  return dest;
}
```
源地址存放于`esi`, 目标地址存放于`edi`, 同时开始拷贝, 当到达0时, 拷贝完成. 约束`"&S","&D","&a"`表明寄存器`esi,edi和eax`早期修饰寄存器, 也就是说, 他们的内容在函数完成前会被改变. 代码会改变`dest`中类容,所以memory位于修饰寄存器列表中.

示例: 移动双字块数据(注意: 函数被声明为一个宏)

```c
#define mov_blk(src, dest, numwords) /
__asm__ __volatile__ (                                          /
                       "cld/n/t"                                /
                       "rep/n/t"                                /
                       "movsl"                                  /
                       :                                        /
                       : "S" (src), "D" (dest), "c" (numwords)  /
                       : "%ecx", "%esi", "%edi"                 /
                       )

```
这里没有输出, 寄存器ecx,esi和edi的内容发生了改变, 这是块移动的副作用. 因此必须将他们添加进修饰寄存器列表.

示例: 系统调用使用gcc内联汇编实现.(所有的系统调用被写成宏`linux/unistd.h`). 带有3个参数的系统调用被定义为如下所示的宏

```c
type name(type1 arg1,type2 arg2,type3 arg3) /
{ /
long __res; /
__asm__ volatile (  "int $0x80" /
                  : "=a" (__res) /
                  : "0" (__NR_##name),"b" ((long)(arg1)),"c" ((long)(arg2)), /
                    "d" ((long)(arg3))); /
__syscall_return(type,__res); /
}
```
无论何时调用带有三个参数的系统调用,以上展示的宏就会用于执行调用.系统调用号位于 eax 中,每个参数位于 `ebx,ecx,edx `中. 最后 `"int 0x80"` 是一条用于执行系统调用的指令. 返回值被存储于 `eax` 中

每个系统调用都以类似的方式实现. `Exit `是一个单一参数的系统调用, 让我们看看它的代码看起来会是怎样:

示例: `Exit`

```c
  asm("movl $1,%%eax;         /* SYS_exit is 1 */
             xorl %%ebx,%%ebx;      /* Argument is in ebx, it is 0 */
             int  $0x80"            /* Enter kernel mode */
            );
```
Exit的系统调用号为1, 同时他的参数是0. 因此我们分配eax包含1, ebx包含0. 同时通过`int $0x80`执行`exit(0)`. 以上即是exit的工作原理.