# register-8086

**AX，BX，CX，DX，SP，BP，SI，DI，IP，FLAG，CS，DS，SS，ES**

## 通用寄存器

包括：**<font color='#ed1941'>AX，BX，CX，DX,SP，BP，SI，DI，共8个</font>**

<font color='#00CACA'>**AX，BX，CX，DX称作数据寄存器**</font>

**AX**：accumulator，累加器

**BX**：base ,基<font color='Magenta'>*地址*</font>寄存器

**CX**：count,计数寄存器，与loop类指令配合使用

**DX**：data，数据寄存器

<font color='#00CACA'>**SP，BP称作指针寄存器**</font>

**SP**：stack pointer，堆栈指针寄存器

**BP**：base pointer，<font color='Magenta'>*基址*</font>指针寄存器

<font color='#00CACA'>**SI，DI称作变址寄存器**</font>

**SI**：source index，源<font color='Magenta'>*变址*</font>寄存器

**DI**：destination index，目的<font color='Magenta'>*变址*</font>寄存器

## 控制寄存器

**IP**：instruction pointer，指令指针寄存器

**FLAG**： 标志寄存器

## 段寄存器

**CS**：code segment，代码段寄存器

**DS**：data segment，数据段寄存器

**SS**：stack segment，堆栈段寄存器

**ES**：extra segment，附加段寄存器



## 详细

其中，通用寄存器除了有自己专用的功能外，还可以用来传送数据和暂存数据。比如AX寄存器可以用作累加器；CX可以用作寄存器，在使用loop指令时指定loop的次数，BX可以用来内存寻址，即[BX]。另外，除BX外，<font color='#00CACA'>SI，DI，BP</font>也可以当做间接寄存器使用。

---------------------

### 数据寄存器

<font color='#00CACA'>**AX，BX，CX，DX**</font>**均可以分为独立的两个寄存器来使用，其中xH为高8位，xL为低8位，x∈(A,B,C,D)**，这样做的目的是兼容以前的8位程序，一样的，在32位CPU中，**<font color='#4B0082'>EAX，EBX，ECX，EDX</font>**也可以分为两个16位寄存器；64位CPU中，**<font color='1E90FF'>RAX，RBX，RCX，RDX</font>**也可以分为两个独立的32位寄存器使用。每个寄存器分开使用时，彼此之间都是<font color='red'>**独立**</font>的，比如AH和AL，当AL中存储的数据溢出时，是不会影响到AH中的数值的，虽然物理上来说，AH和AL是同一寄存器的高8位和低8位。但是修改AH或AL均会影响AX的值。

#### AX

**<font color='#00CACA'>AX</font>**：在使用div、mul指令时会使用到。

**<font color="Cornislk">div</font>**：除数可以在寄存器或者内存中，而被除数默认存放在AX或AX和DX中。除数有8位和16位两种情况

- 当除数是8位时，被除数为16位，默认在AX中存放。AL存放商，AH存放余数。

- 当除数是16位时，被除数为32位，默认在AX和DX中存放，DX存放高16位，AX存放低16位。AX存放商，DX存放余数。

- 当除数是32位时，被除数为64位，默认在EAX和EDX中存放，EDX存放高16位，EAX存放低16位。EAX存放商，EDX存放余数。

  | 被除数          | 除数            | 商   | 余数 |
  | --------------- | --------------- | ---- | ---- |
  | 16位 \| AX      | 8位 \| reg/mem  | AL   | AH   |
  | 32位 \| DX:AX   | 16位 \| reg/mem | AX   | DX   |
  | 64位 \| EDX:EAX | 32位 \| reg/men | EAX  | EDX  |

  

**<font color=" Cornislk">mul</font>**：乘数和被乘数的大小必须保持一致，乘积的大小则是它们的一倍。这三种类型都可以使用寄存器和内存操作数，但不能使用立即数

- 都是8位，则一个默认存放在AL中，另一个可以在寄存器或者内存中，不能为立即数。

- 都是16位，一个默认在AX中

- 都是32位，一个默认在EAX中

  | 被乘数 | 乘数         | 乘积    |
  | ------ | ------------ | ------- |
  | AL     | 8 \| reg/mem | AX      |
  | AX     | 16\|reg/mem  | DX:AX   |
  | EAX    | 32\|reg/mem  | EDX:EAX |
  
  

#### BX

**<font color='#00CACA'>BX</font>**：基址寄存器，除了可以暂存数据外还可以用于内存寻址

#### CX

**<font color='#00CACA'>CX</font>**：计数寄存器，除了可以暂存数据外还可以与loop指令配合使用，用于指定loop的次数。

cpu在执行loop指令时会做两件事：

- 1）CX=CX-1
- 2）如果CX中的值为0，则跳出循环

因此CX寄存器中的值可以看做是循环代码的执行次数。

#### DX

**<font color='#00CACA'>DX</font>**：数据寄存器，除了暂存数据外还可以与div和mul指令配合使用，见AX中[div](#AX)、[mul](#AX)指令相关部分。

### 指针寄存器

#### BP

**<font color='#585eaa'>BP</font>**：base pointer，基指针寄存器，可以用于内存寻址，同样可用于内存寻址的还有**<font color="#f58220">BX、SI、DI</font>**寄存器。

#### SP

**<font color='#585eaa'>BP</font>**：stack pointer，堆栈指针寄存器，需要与SS段寄存器一起使用。

### 变址寄存器

#### SI

**<font color="#5c7a29">SI</font>**：source index，源变址寄存器，用于存放某个存储单元地址的偏移地址。同样也可以暂存数据。

#### DI

**<font color="#5c7a29">DI</font>**：destination index，目的变址寄存器，类似SI。

### 其他

#### CS：IP

**<font color="#f26522">CS：IP</font>**：任意时刻，CS：IP指向CPU下一条要执行的指令的地址。CS为代码段寄存器（code segment），IP为指令指针寄存器（instruction pointer）。

#### SS：SP

**<font color="#f26522">SS：SP</font>**：在任何时刻，SS:SP 都是指向栈顶元素 。SS，stack segment，堆栈段寄存器；SP，stack pointer，堆栈指针寄存器。

