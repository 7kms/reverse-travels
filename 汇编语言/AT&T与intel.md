# AT&T与intel
> https://www.linuxprobe.com/gcc-how-to.html

1. 源操作数和目的操作数(AT&T语法的操作数方向和Intel语法的刚好相反).
    - intel: "Op-code dst src". 第一操作数为目的操作数, 第二操作数为源操作数
    - AT&T: "Op-code src dst".

2. 寄存器命名
    - AT&T: 寄存器名称有"%"前缀, 即如果使用"eax"应该写作"%eax"

3. 立即数
    - intel: 十六进制常量以"h"为后缀
    - AT&T: 立即数以"$"为前缀, 十六进制常量前缀"0x". 对于十六进制会记为"$0x111"

4. 操作数大小

    - intel: 通过给存储器操作添加`"byte ptr", word ptr, dword ptr`前缀. 如`mov al,byte ptr foo` 
    - AT&T: 存储器操作数的大小取决于操作码名字的最后一个字符. 操作码后缀`b,w,l`分别指明了字节`字节(8位), 字(16位), 长型(32位)`存储器引用. 如`movb foo,%al`

5. 存储器操作数
    - intel: 基址寄存器包含在`[`和`]`中. 间接内存引用为`section:[base + index*scale +disp]`
    - AT&T: 基址寄存器包含在`(`和`)`中. 间接内存引用为`section:disp(base,index, scale)`. 当一个常量用于`disp或scale`时,不能添加`$`前缀


```s
+------------------------------+------------------------------------+
|       Intel Code             |      AT&T Code                     |
+------------------------------+------------------------------------+
| mov     eax,1                |  movl    $1,%eax                   |   
| mov     ebx,0ffh             |  movl    $0xff,%ebx                |   
| int     80h                  |  int     $0x80                     |   
| mov     ebx, eax             |  movl    %eax, %ebx                |
| mov     eax,[ecx]            |  movl    (%ecx),%eax               |
| mov     eax,[ebx+3]          |  movl    3(%ebx),%eax              | 
| mov     eax,[ebx+20h]        |  movl    0x20(%ebx),%eax           |
| add     eax,[ebx+ecx*2h]     |  addl    (%ebx,%ecx,0x2),%eax      |
| lea     eax,[ebx+ecx]        |  leal    (%ebx,%ecx),%eax          |
| sub     eax,[ebx+ecx*4h-20h] |  subl    -0x20(%ebx,%ecx,0x4),%eax |
+------------------------------+------------------------------------+

```