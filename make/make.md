# makefile

## 源文件编译过程

以c语言源文件、gcc编译器为例。



​				预处理					编译器					汇编器						 链接器

source.c   ---------> source.i   ---------> source.s   --------->  source.o   ---------> source 

gcc   	 		- E							-S							-c 



预处理阶段: 预处理器（cpp）将头文件展开、宏替换、去掉注释。

编译阶段：编译器（ccl） 将预处理后的c文件编译成汇编文件。

汇编阶段：汇编器（as）将汇编文件翻译成机器语言指令，把这些指令打包成一种叫做可重定位目标程序的格式，并将结果保存在```.o```结尾（windows下为```.obj```结尾）的文件中。

链接阶段：链接器（ld）将函数库中的指令代码组合到最终的可执行文件中。

## makefile结构

一个简单的makefile文件组织如下：

```makefile
target ...: prerequisistes ..
				recipe
				...
				...
```

**target**可以是目标文件的名称，可以是一个可执行文件，也可以是一个标签。

**prerequisites**是target依赖的文件或target。

**recipe**任意的shell命令。recipe所在的行（每一行）要以一个tab（制表符）开始。



makefile定义了文件之间的依赖关系，通俗的来说：

> ```
> prerequisites中如果有一个以上的文件比target文件要新的话，command所定义的命令就会被执行。
> ```

## make是如何工作的

一个makefile的简单示例：

```makefile
edit : main.o kbd.o command.o display.o \
        insert.o search.o files.o utils.o
    cc -o edit main.o kbd.o command.o display.o \
        insert.o search.o files.o utils.o

main.o : main.c defs.h
    cc -c main.c
kbd.o : kbd.c defs.h command.h
    cc -c kbd.c
command.o : command.c defs.h command.h
    cc -c command.c
display.o : display.c defs.h buffer.h
    cc -c display.c
insert.o : insert.c defs.h buffer.h
    cc -c insert.c
search.o : search.c defs.h buffer.h
    cc -c search.c
files.o : files.c defs.h buffer.h command.h
    cc -c files.c
utils.o : utils.c defs.h
    cc -c utils.c
clean :
    rm edit main.o kbd.o command.o display.o \
        insert.o search.o files.o utils.o
```



1. 在当前目录下查找名为makefile或Makefile的文件
2. 如果找到，make会寻找第一个target文件，并把它作为最终的目标文件。在上面的makefile定义中，edit就是最终的目标文件。
3. 如果第一个target文件不存在或者第一个target文件所依赖的prerequisites比target文件的修改时间更新，那么会执行定义的recipe命令来生成新的target文件。
4. 如果一个target文件依赖的prerequisites的prerequisites也不存在或者修改时间更新的话，make同样会执行对应的recipe。
5. 最终会生成最新的第一个target文件。

> make不关心recipe是否报错，make只关心makefile中定义的依赖关系是否能被满足。另外一点是与最终的目标文件没有直接或者间接关系的target内定义的recipe不会在执行默认参数的make时执行，除非在运行make时指定。

## makefile中使用变量

makefile中可以像编程语言一样定义变量。随着工程变复杂，某些表示一类文件的字符串可能会在makefile中多次出现，这时如果我们添加了一个新的此类文件就需要在makefile中多次修改，增加了makefile维护的难度。使用变量可以解决这个问题。

比如我们定义一个objects变量：

```makefile
objects = main.o kbd.o command.o display.o \
     insert.o search.o files.o utils.o
```

接下来，我们就可以很方便地在我们的makefile中以 `$(objects)` 的方式来使用这个变量了：

```makefile
objects = main.o kbd.o command.o display.o \
    insert.o search.o files.o utils.o

edit : $(objects)
    cc -o edit $(objects)
main.o : main.c defs.h
    cc -c main.c
kbd.o : kbd.c defs.h command.h
    cc -c kbd.c
command.o : command.c defs.h command.h
    cc -c command.c
display.o : display.c defs.h buffer.h
    cc -c display.c
insert.o : insert.c defs.h buffer.h
    cc -c insert.c
search.o : search.c defs.h buffer.h
    cc -c search.c
files.o : files.c defs.h buffer.h command.h
    cc -c files.c
utils.o : utils.c defs.h
    cc -c utils.c
clean :
    rm edit $(objects)
```

makefile支持一些运算符：

- `=`：赋值，使用`=`赋值，需要特别注意的是，使用`=`赋值时，如果引用了makefile中定义的变量，那么这个被引用的变量的值为整个makefile中最后被指定的值。

  ```makefile
  var_a = 1
  var_b = $(a)
  var_a = 2
  
  test : 
  		echo $(var_b)
  ```

  执行```make test```，输出为`2`。

- `+=`：追加 append

  和`=`一样，如果在运算符右边引用变量，那么变量的值为最终值而不是当前值。 

- `:=`：直接赋值，与`=`的区别在于如果引用了变量，make会将变量展开为当前位置的值，而不是变量在文件中的最终值。

  ```makefile
  var_a = 1
  var_b := $(a)
  var_a = 2
  
  test : 
  		echo $(var_b)
  ```

  执行`make test`，输出为1

- `?=`：条件赋值，如果运算符左边的变量没有被赋值，那么就赋予它运算法右边的值。

  ```makefile
  var_a ?= 1
  
  test :
  		echo $(var_a)
  ```

  输出为1

  ```makefile
  var_a = 0
  var_a ?= 1
  
  test :
  		echo $(var_a)
  ```

  输出为0

## 让make自动推导

make找到一个`.o`文件，它就会自动的把源文件添加到依赖中，因此不需要在`.o`文件的依赖中手动的添加源文件，此外，make还会自动推导对应的命令。

>  GNU的make很强大，它可以自动推导文件以及文件依赖关系后面的命令，于是我们就没必要去在每一个 `.o` 文件后都写上类似的命令，因为，我们的make会自动识别，并自己推导命令。
>
> 只要make看到一个 `.o` 文件，它就会自动的把 `.c` 文件加在依赖关系中，如果make找到一个 `whatever.o` ，那么 `whatever.c` 就会是 `whatever.o` 的依赖文件。并且 `cc -c whatever.c` 也会被推导出来，于是，我们的makefile再也不用写得这么复杂。我们的新makefile又出炉了。

```makefile
objects = main.o kbd.o command.o display.o \
    insert.o search.o files.o utils.o

edit : $(objects)
    cc -o edit $(objects)

main.o : defs.h
kbd.o : defs.h command.h
command.o : defs.h command.h
display.o : defs.h buffer.h
insert.o : defs.h buffer.h
search.o : defs.h buffer.h
files.o : defs.h buffer.h command.h
utils.o : defs.h

.PHONY : clean
clean :
    rm edit $(objects)
```

`.PHONY`表示clean是个伪目标文件。

我们发现上面的makefile文件中有许多的`.o`所依赖的`.h`是重复的（除了它们自身所依赖的`.h`文件，这部分文件可以由make自行推导），借助make的自动推导功能，我们可以把这部分`.o`文件折叠到一起，之后makefile就变成了下面的样子：

```makefile
objects = main.o kbd.o command.o display.o \
    insert.o search.o files.o utils.o

edit : $(objects)
    cc -o edit $(objects)

$(objects) : defs.h
kbd.o command.o files.o : command.h
display.o insert.o search.o files.o : buffer.h

.PHONY : clean
clean :
    rm edit $(objects)
```

> 这种风格，让我们的makefile变得很简单，但我们的文件依赖关系就显得有点凌乱了。鱼和熊掌不可兼得。还看你的喜好了。我是不喜欢这种风格的，一是文件的依赖关系看不清楚，二是如果文件一多，要加入几个新的 `.o` 文件，那就理不清楚了。

## 清空目标文件

每个makefile中都应该有一个清空目标文件（`.o`和可执行文件）的规则，这不仅有利于重新编译，也有利于文件的清洁。

```makefile
clean:
    rm edit $(objects)
```

或：

```makefile
.PHONY : clean
clean:
    -rm edit $(objects)
```

> 前面说过， `.PHONY` 表示 `clean` 是一个“伪目标”。而在 `rm` 命令前面加了一个小减号的意思就是，也许某些文件出现问题，但不要管，继续做后面的事。当然， `clean` 的规则不要放在文件的开头，不然，这就会变成make的默认目标，相信谁也不愿意这样。不成文的规矩是——“clean从来都是放在文件的最后”。

## makefile里有什么

> Makefile里主要包含了五个东西：显式规则、隐晦规则、变量定义、文件指示和注释。

1. 显示规则：由makefile的书写者决定的如何生成一个或多个目标文件的规则。是由Makefile的书写者明显指出要生成的文件、文件的依赖文件和生成的命令。
2. 隐晦规则：由make自动推断的规则，隐晦规则可以让我们比较简略的书写makefile文件。
3. 变量定义：makefile中可以定义变量，有点类似c语言中的宏。定义变量的方式与一般的编程语言类似：```var = ···```，使用变量的方式为：```$(var)```。
4. 文件指示：可以将另一个makefile的规则导入，也可以决定makefile中的哪些部分是生效的，这类似于c语言中的`#include`和`#if`条件编译。
5. 注释：makefile中的注释行以`#`开始，如果想要在makefile中使用`#`字符可以使用转义的方式表示：`\#`。

## makefile的文件名

`GNU make`支持”GNUmakefile“、“makefile”、“Makefile”三种默认命名，当需要使用别的名称的makefile时，可以添加`-f`或者`--file`选项来指定makefile：

```makefile
make - f ···
```

## 引用其它Makefile

在makefile中使用`include`关键字可以把其它makefile包含进来。

```makefile
include <filename>
```

`filename`可以是当前操作系统Shell的文件模式（可以包含路径和通配符）。

`include`前可以有空字符，但不能以`tab`开始。

make命令开始时，会查找`include`关键字指出的其它makefile，并把它的内容插入到当前位置。如果文件没有指明绝对路径或者相对路径的话，make首先会在当前目录下寻找，如果当前目录下没有找到，会在下面几个目录查找：

1. 如果make执行时，使用了`-I`或者`--include-dir`选项，那么make会在此目录下再进行查找
2. 如果目录`<prefix>/include`（一般是： `/usr/local/bin` 或 `/usr/include`）存在的话，make也会去寻找。

如果有文件没找到，make会生成一条警告⚠️消息，但不会立即停止运行。它会继续载入其它文件，一旦完成makefile的读取，make会再次重试，如果还不能找到，make会出现一条致命信息。如果想让make不理会无法读取的文件而继续执行，可以在`include`前添加一个减号`-`：

```makefile
-include <filename>
```

> 其表示，无论include过程中出现什么错误，都不要报错继续执行。和其它版本make兼容的相关命令是sinclude，其作用和这一个是一样的。

## GNU make工作方式

1. 读入所有的Makefile。
2. 读入被include的其它Makefile。
3. 初始化文件中的变量。
4. 推导隐晦规则，并分析所有规则。
5. 为所有的目标文件创建依赖关系链。
6. 根据依赖关系，决定哪些目标要重新生成。
7. 执行生成命令。

1-5步为第一个阶段，6-7为第二个阶段。第一个阶段中，如果定义的变量被使用了，那么，make会把其展开在使用的位置。但make并不会完全马上展开，make使用的是拖延战术，如果变量出现在依赖关系的规则中，那么仅当这条依赖被决定要使用了，变量才会在其内部展开。

# 书写规则

规则的语法如下：

```makefile
targets : prerequisites
    command
    ...
```

或

```makefile
targets : prerequisites ; command
    command
    ...
```

如果command与“targets : prerequisites”不在同一行则必须以`Tab`开始；如果在同一行则需要以分号分隔。

## 通配符

make支持Unix风格的通配符，例如`~`表示当前用户的`$HOME`目录，`~jhq`表示用户jhq的宿主目录；`*`可以匹配任意字符，想要匹配`*`可以使用转义。

| 通配符 |                                |
| ------ | ------------------------------ |
| *      | 匹配0个或者是任意个字符        |
| ？     | 匹配任意一个字符               |
| []     | 可以将指定匹配的字符放在"[]"中 |
| %      |                                |

除了表示文件的地方可以使用通配符外，变量也可以使用通配符：

```makefile
objects = *.o
```

> `*.o`并不会展开成文件名的集合，在makefile中变量就是c/c++中的宏，它们只会进行简单的直接替换。

但是如果我们的通配符使用在依赖的规则中的话一定要注意这个问题：不能通过引用变量的方式来使用，如下所示。

```makefile
OBJ=*.c
test:$(OBJ)
    gcc -o $@ $^
```

我们去执行这个命令的时候会出现错误，提示我们没有 "*.c" 文件

如果想要变量的内容`*.o`展开为所有`.o`文件名的集合，可以使用下面的方式：

```makefile
objects := $(wildcard *.o)
```

`wildcard`为扩展通配符，除了`wildcard`外还有`notdir`（去除路径）和`patsubst`（替换通配符）。

## 文件搜索

在一些大的工程中，有大量的源文件，我们通常的做法是把这许多的源文件分类，并存放在不同的目录中。所以，当make需要去找寻文件的依赖关系时，你可以在文件前加上路径，但最好的方法是把一个路径告诉make，让make在自动去找。

Makefile文件中的特殊变量 `VPATH` 就是完成这个功能的，如果没有指明这个变量，make只会在当前的目录中去找寻依赖文件和目标文件。如果定义了这个变量，那么，make就会在当前目录找不到的情况下，到所指定的目录中去找寻文件了。

```makefile
VPATH = src:../headers
```

上面的定义指定两个目录，“src”和“../headers”，make会按照这个顺序进行搜索。目录由“冒号”分隔。（当然，当前目录永远是最高优先搜索的地方）。

另一个设置文件搜索路径的方法是使用make的“vpath”关键字（注意，它是全小写的），这不是变量，这是一个make的关键字，这和上面提到的那个VPATH变量很类似，但是它更为灵活。它可以指定不同的文件在不同的搜索目录中。这是一个很灵活的功能。它的使用方法有三种：

- ```vpath <pattern> <directories> ```：为符合模式<pattern>的文件指定搜索目录<directories>。
- ```vpath <pattern>```：清除符合模式<pattern>的文件的搜索目录。
- ```vpath```：除所有已被设置好了的文件搜索目录。

> 其实就是只有一种用法即第一种用法，可以理解为<pattern>的默认值为`*`也就是匹配所有文件，而<directories>的默认值为空，也就是把搜索目录设置为空。

另外在使用 vpath 的时候，搜索的条件中可以包含模式字符“%”，这个符号的作用是匹配一个或者是多个字符，例如“%.c”表示搜索路径下所有的 .c 结尾的文件。如果搜索条件中没有包含“%" ，那么搜索的文件就是具体的文件名称。

使用什么样的搜索方法，主要是基于编译器的执行效率。使用 VPATH 的情况是前路径下的文件较少，或者是搜索的文件不能使用通配符表示，这些情况下使用VPATH最好。如果存在某个路径的文件特别的多或者是可以使用通配符表示的时候，就不建议使用 VPATH 这种方法，为什么呢？因为 VPATH 在去搜索文件的时没有限制条件，所以它回去检索这个目录下的所有文件，每一个文件都会进行对比，搜索和我们目录名相同的文件，不仅速度会很慢，而且效率会很低。我们在这种情况下就可以使用 vpath 搜索，它包含搜索条件的限制，搜索的时候只会从我们规定的条件中搜索目标，过滤掉不符合条件的文件，当然查找的时候也会比较的快。

## 伪目标

可以用标记`.PHONY`来显式的指明一个目标是伪目标。

```makefile
.PHONY : clean
```

伪目标一般没有依赖的文件。但是，我们也可以为伪目标指定所依赖的文件。伪目标同样可以作为“默认目标”，只要将其放在第一个。一个示例就是，如果你的Makefile需要一口气生成若干个可执行文件，但你只想简单地敲一个make完事，并且，所有的目标文件都写在一个Makefile中，那么你可以使用“伪目标”这个特性：

```makefile
all : prog1 prog2 prog3
.PHONY : all

prog1 : prog1.o utils.o
    cc -o prog1 prog1.o utils.o

prog2 : prog2.o
    cc -o prog2 prog2.o

prog3 : prog3.o sort.o utils.o
    cc -o prog3 prog3.o sort.o utils.o
```

## 多目标

| 自动化变量 | 描述                                                         |
| ---------- | ------------------------------------------------------------ |
| `$@`       | 代表目标文件名(target)                                       |
| `$%`       | 仅当目标是函数库文件时，表示规则中的目标成员名。（函数库文件：Unix下`.a`，WIndows下`.lib`） |
| `$<`       | 代表第一个依赖文件                                           |
| `$?`       | 所有比目标新的依赖目标的集合，以空格分隔。                   |
| `$^`       | 代表依赖文件列表(prerequisites)                              |
| `$+`       | 类似`$^`，区别在于它不会去除重复的依赖目标。                 |
| `$*`       | 代表目标模式中`%`及其之前的部分。如果目标是`DIR/test.c`，目标的模式为`%.c`，那么`$*`的值就为`DIR/test`|如果目标文件的后缀是make识别的，那么`$*`就是除了后缀的那一部分 |



## 条件判断

条件语句的作用：条件语句可以根据一个变量的值来控制 make 执行或者时忽略 Makefile 的特定部分，条件语句可以是两个不同的变量或者是常量和变量之间的比较。

| 关键字 | 描述 |
| ------ | ---- |
| ifeq   |      |
| ifneq  |      |
| ifdef  |      |
| ifndef |      |

> 条件判断关键字和之后的条件判断语句之间要有空格，并且条件关键字所在的行不能以`Tab`开始。

```makefile
.PHONY : test

a = 1
b = 1
c = 2

test :
    ifeq ($(a), $(b))
	    @echo "a == b"
    endif

    ifneq ($(a), $(c))
	    @echo "a != c"
    endif

    ifdef a
	    @echo "def a"
    endif

    ifndef 9527
	    @echo "ndef 9527"
    endif
```

输出：

```bash
a == b
a != c
def a
ndef 9527
```

# 函数

makefile函数调用格式：

```makefile
$(<function> <arguments>)
或
${<function> <arguments>}
```

参数之间使用逗号分隔，参数和函数名之间用空格分隔。

## 字符串处理函数

### 模式字符替换函数$(patsubst)

usage:

```makefile
$(patsubst <pattern>,<replacement>,<text>)
```

说明：

函数的功能是查找text中的单词是否符合模式pattern，如果匹配的话，则用replacement替换。返回值为替换后的新字符串。

Example:

```makefile
OBJ = $(patsubst %.c,%.o,1.c 2.c 3.c)

all:
	@echo $(OBJ)
```

output:

```makefile
1.o 2.o 3.o
```

### 字符串替换函数$(subst)

usage:

```makefile
$(subst <from>,<to>,<text>)
```

函数说明：函数的功能是把字符串中的 form 替换成 to，返回值为替换后的新字符串。实例:

```makefile
OBJ=$(subst ee,EE,feet on the street)
all:
    @echo $(OBJ)
```

不支持正则。

### 去空格函数$(strip)

usage:

```makefile
$(strip <string>)
```

函数说明：函数的功能是去掉字符串的开头和结尾的字符串，并且将其中的多个连续的空格合并成为一个空格。返回值为去掉空格后的字符串。实例：

```makefile
OBJ=$(strip    a       b c)
all:
    @echo $(OBJ)
```

### 查找字符串$(findstring <find>, <in>)

usage:

```makefile
$(findstring <find>, <in>)
```

函数说明：函数的功能是查找 in 中的 find ,如果我们查找的目标字符串存在。返回值为目标字符串，如果不存在就返回空。实例：

```makefile
OBJ=$(findstring a,a b c)
all:
    @echo $(OBJ)
```

### 过滤函数$(filter)

usage:

```makefile
$(filter <pattern>,<text>)
```

函数说明：函数的功能是过滤出 text 中符合模式 pattern 的字符串，可以有多个 pattern 。返回值为过滤后的字符串。实例：

```makefile
OBJ=$(filter %.c %.o,1.c 2.o 3.s)
all:
    @echo $(OBJ)
```

### 反过滤函数$(filter-out)

usage:

```makefile
$(filter-out <pattern>,<text>)
```

函数说明：函数的功能是功能和 filter 函数正好相反，但是用法相同。去除符合模式 pattern 的字符串，保留符合的字符串。返回值是保留的字符串。实例：

```makefile
OBJ=$(filter-out 1.c 2.o ,1.o 2.c 3.s)
all：
    @echo $(OBJ)
```

### 排序函数$(sort <list>)

usage：

```makefile
$(sort <list>)
```

按字典序排列（升序），返回值为排序后的字符串。实例：

```makefile
OBJ=$(sort 5 2 3 4 1 b a c)

all:
	@echo $(OBJ)
```

output:

```bash
1 2 3 4 5 a b c
```

### 取单词函数$(word)

usage:

```makefile
$(word <n>,<text>)
```

函数说明：函数的功能是取出函数`<text>`中的第n个单词。返回值为我们取出的第 n 个单词（从1开始计数）。实例：

```makefile
OBJ=$(word 2,1.c 2.c 3.c)
all:
		@echo $(OBJ)
```

output:

```bash
2.c
```

## 常用文件名操作函数

### 取目录函数$(dir)

usage:

```makefile
$(dir <names>)
```

函数说明：函数的功能是从文件名序列 names 中取出目录部分，如果没有 names 中没有 "/" ，取出的值为 "./" 。返回值为目录部分，指的是最后一个反斜杠之前的部分。如果没有反斜杠将返回“./”。实例：

```makefile
OBJ=$(dir src/foo.c hacks)
all:
    @echo $(OBJ)
```

output:

```bash
src/ ./
```

### 取文件函数$(notdir)

usage:

```makefile
$(notdir <names>)
```

函数说明：函数的功能是从文件名序列 names 中取出非目录的部分。非目录的部分是最后一个反斜杠之后的部分。返回值为文件非目录的部分。实例：

```makefile
OBJ=$(notdir src/foo.c hacks)
all:
    @echo $(OBJ)
```

output:

```bash
foo.c hacks
```

### 取后缀名函数($suffix)

usage:

```makefile
$(suffix <names>)
```

函数说明：函数的功能是从文件名序列中 names 中取出各个文件的后缀名。返回值为文件名序列 names 中的后缀序列，如果文件没有后缀名，则返回空字符串。实例：

```makefile
OBJ=$(suffix src/foo.c hacks)
all:
    @echo $(OBJ)
```

output:

```bash
.c
```

### 取前缀函数$(basename)

usage:

```makefile
$(basename <names>)
```

函数说明：函数的功能是从文件名序列 names 中取出各个文件名的前缀部分。返回值为被取出来的文件的前缀名，如果文件没有前缀名则返回空的字符串。实例：

```makefile
OBJ=$(basename src/foo.c hacks)

all:
	@echo $(OBJ)
```

output:

```bash
src/foo hacks
```

### 添加后缀名函数$(addsuffix)

usage:

```makefile
$(addsuffix <suffix>,<names>)
```

函数说明：函数的功能是把后缀 suffix 加到 names 中的每个单词后面。返回值为添加上后缀的文件名序列。实例：

```makefile
OBJ=$(addsuffix .c,src/foo.c hacks)
all:
    @echo $(OBJ)
```

output:

```bash
src/foo.c.c hacks.c
```

### 添加前缀名函数$(addprefix)

usage:

```makefile
$(addprefix <prefix>,<names>)
```

函数说明：函数的功能是把前缀 prefix 加到 names 中的每个单词的前面。返回值为添加上前缀的文件名序列。实例：

```makefile
OBJ=$(addprefix src/,foo.c hacks)

all:
	@echo $(OBJ)
```

output:

```bash
src/foo.c src/hacks
```

### 链接函数$(join)

```makefile
$(join <list1>,<list2>)
```

函数说明：函数功能是把 list2 中的单词**_对应_**的拼接到 list1 的后面。如果 list1 的单词要比 list2的多，那么，list1 中多出来的单词将保持原样，如果 list1 中的单词要比 list2 中的单词少，那么 list2 中多出来的单词将保持原样。返回值为拼接好的字符串。实例：

```makefile
OBJ=$(join src car,abc zxc qwe)

all:
	@echo $(OBJ)
```

output

```bash
srcabc carzxc qwe
```

### 获取匹配模式文件名函数$(wildcard)

```makefile
$(wildcard <pattern>)
```

函数说明：函数的功能是列出当前目录下所有符合模式的 PATTERN 格式的文件名。返回值为空格分隔并且存在当前目录下的所有符合模式 PATTERN 的文件名。实例：

```makefile
OBJ=$(wildcard *.c  *.h)
all:
    @echo $(OBJ)
```

执行 make 命令，可以得到当前函数下所有的 ".c " 和 ".h" 结尾的文件。这个函数通常跟的通配符 "*" 连用，使用在依赖规则的描述的时候被展开（在这里我们的例子如果没有 wildcard 函数，我们的运行结果也是这样，"echo" 属于 shell 命令，在使用通配符的时通配符自动展开，我们这里只是相要说明一下这个函数在使用时，如果通过引用变量出现在规则中要被使用）。

## 其它常用函数

### $(foreach)

```makefile
$(foreach <var>,<list>,<text>)
```

函数的功能是：把参数`<list>`中的单词逐一取出放到参数`<var>`所指定的变量中，然后再执行`<text>`所包含的表达式。每一次`<text>`会返回一个字符串，循环过程中，`<text>`的返所返回的每个字符串会以空格分割，最后当整个循环结束的时候，`<text>`所返回的每个字符串所组成的整个字符串（以空格分隔）将会是 foreach 函数的返回值。所以`<var>`最好是一个变量名，`<list>`可以是一个表达式，而`<text>`中一般会只用`<var>`这个参数来一次枚举`<list>`中的单词。

```makefile
names = a b c d
files = $(foreach n,$(names),$(n).o)

all:
	@echo $(files)
```

output:

```makefile
a.o b.o c.o d.o
```

> 注意，foreach 中的 `<var>` 参数是一个临时的局部变量，foreach 函数执行完后，参数`<var>`的变量将不再作用，其作用域只在 foreach 函数当中。

### $(if)

```makefile
$(if <condition>,<then-part>)
或者：
$(if <condition>,<then-part>,<else-part>)
```

如果<condition>返回的是非空的字符串，那么这个表达式就相当于返回真，<then-part>会被计算，否则执行<else-part>（如果有）。

```makefile
#OBJ := foo.c
OBJ := $(if $(OBJ),$(OBJ),main.c)

all:
	@echo $(OBJ)
```

输出：

```bash
main.c
```

如果取消第一行的注释，则输出：

```bash
foo.c
```

### $(call)

call 函数是唯一一个可以用来创建新的参数化的函数。我们可以用来写一个非常复杂的表达式，这个表达式中，我们可以定义很多的参数，然后你可以用 call 函数来向这个表达式传递参数。

```makefile
$(call <expression>,<param1>,<param2>,<param3>,...)
```

当 make 执行这个函数的时候，`expression`参数中的变量$(1)、$(2)、$(3)等，会被参数`parm1`，`parm2`，`parm3`依次取代。而`expression`的返回值就是 call 函数的返回值。

```makefile
reverse = $(1) $(2)
foo = $(call reverse,a,b)
all:
	@echo $(foo)
```

输出：```a b```

```makefile
reverse = $(2) $(1)
foo = $(call reverse,a,b)
all:
      @echo $(foo)
```

输出：```b a```

### $(origin)

```makefile
$(origin <variable>)
```

origin 函数不像其他的函数，它并不操作变量的值，它只是告诉你这个变量是哪里来的。

下面是origin函数返回值：

- “undefined”：如果<variable>从来没有定义过，函数将返回这个值。
- “default”：如果<variable>是一个默认的定义，比如说“CC”这个变量。
- “environment”：如果<variable>是一个环境变量并且当Makefile被执行的时候，“-e”参数没有被打开。
- “file”：如果<variable>这个变量被定义在Makefile中，将会返回这个值。
- “command line”：如果<variable>这个变量是被命令执行的，将会被返回。
- “override”：如果<variable>是被override指示符重新定义的。
- “automatic”：如果<variable>是一个命令运行中的自动化变量。

这些信息对于我们编写 Makefile 是非常有用的，例如假设我们有一个 Makefile ，其包含了一个定义文件`Make.def`，在`Make.def`中定义了一个变量`bletch`，而我们的环境变量中也有一个环境变量`bletch`，我们想去判断一下这个变量是不是环境变量，如果是我们就把它重定义了。如果是非环境变量，那么我们就不重新定义它。于是，我们在 Makefile 中，可以这样写：

```makefile
ifdef bletch
ifeq "$(origin bletch)" "environment"
bletch = barf,gag,etc
endif
endif
```

当然，使用`override`关键字不就可以重新定义环境中的变量了吗，为什么需要使用这样的步骤？是的，我们用`override`是可以达到这样的效果的，可是`override`会把从命令行定义的变量也覆盖了，而我们只想重新定义环境传来的，而不是重新定义命令行传来的。

# makefile命令

## 命令回显

通常 make 在执行命令行之前会把要是执行的命令行输出到标准输出设备。我们称之为 "回显"，就好像我们在 shell 环境下输入命令执行时一样。如果规则的命令行以字符“@”开始，则 make 在执行的时候就不会显示这个将要被执行的命令。典型的用法是在使用`echo`命令输出一些信息时。

我们在执行 make 时添加上一些参数，可以控制命令行是否输出。当使用 make 的时候机加上参数`-n`或者是`--just-print` ，执行时只显示所要执行的命令，但不会真正的执行这个命令。只有在这种情况下 make 才会打印出所有的 make 需要执行的命令，其中包括了使用的“@”字符开始的命令。这个选项对于我们调试 Makefile 非常的有用，使用这个选项就可以按执行顺序打印出 Makefile 中所需要执行的所有命令。而 make 参数`-s`或者是`--slient`则是禁止所有的执行命令的显示。就好像所有的命令行都使用“@”开始一样。

## 命令执行

当规则中的目标需要被重建的时候，此规则所定义的命令将会被执行，如果是多行的命令，那么每一行命令将是在一个独立的子 shell 进程中被执行。因此，多命令行之间的执行命令时是相互独立的，相互之间不存在依赖。

在 Makefile 中**书写在同一行中的多个命令属于一个完整的 shell 命令行**，书写在独立行的一条命令是一个独立的 shell 命令行。因此：在一个规则的命令中命令行 “cd”改变目录不会对其后面的命令的执行产生影响。就是说之后的命令执行的工作目录不会是之前使用“cd”进入的那个目录。如果达到这个目的，就不能把“cd”和其后面的命令放在两行来书写。而应该**把这两个命令放在一行上用分号隔开**。这样才是一个完整的 shell 命令行。



make 对所有规则的命令的解析使用环境变量“SHELL”所指定的那个程序。在 GNU make 中，默认的程序时 “/bin/sh”。不像其他的绝大多数变量，他们的只可以直接从同名的系统环境变量那里获得。make 的环境变量 “SHELL”没有使用环境变量的定义。因为系统环境变量“SHELL”指定的那个程序被用来作为用户和系统交互的接口程序，他对于不存在直接交互过程的 make 显然不合适。在 make 环境变量中“SHELL”会被重新赋值；他作为一个变量我们也可以在 Makefile 中明确的给它赋值，变量“SHELL“的默认值时“/bin/sh”（macos下为/bin/zsh）。

## 并发执行命令

GNU make 支持同时执行多条命令。通常情况下，同一时刻只有一个命令在执行，下一个命令只有在当前命令结束之后才能够开始执行。不过可以通过 make 命令行选项 "-j" 或者 "--jobs" 来告诉 make 在同一时刻可以允许多条命令同时执行。

如果选项 "-j" 之后存在一个整数，其含义是告诉 make 在同一时刻可以允许同时执行的命令行的数目。这个数字被称为`job slots`。当 "-j" 选项中没有出现数字的时候，那么同一时间执行的命令数目没有要求。使用默认的`job solts`，值为1，表示make将串行的执行规则的命令（同一时刻只能由一条命令被执行）。

并行执行命令所带来的问题是显而易见的：

- 多个同时执行的命令的输出信息将同时被输出到终端。当出现错误时很难根据一大堆凌乱的信息来区分那条命令执行错误。
- 在同一时刻可能会存在多个命令执行的进程同时读取到标准输入，但是对于白哦准输入设备来说，在同一时刻只能存在一个进程访问它。就是说在某个时间点，make只能保证此刻正在执行的进程中的一个进程读取标准输入流。而其他的进程键的标准输入流将设置为无效。因此在此一时刻多个执行命令的进程中只有一个进程获得标准输入，而其他的需要读取标准输入流的进程由于输入流无效而导致致命的错误。

# Makefile嵌套执行make

## makefile嵌套

我们都知道在一个大的工程文件中，不同的文件按照功能被划分到不同的模块中，也就说很多的源文件被放置在了不同的目录下。每个模块可能都会有自己的编译顺序和规则，如果在一个 Makefile 文件中描述所有模块的编译规则，就会很乱，执行时也会不方便，所以就需要在不同的模块中分别对它们的规则进行描述，也就是每一个模块都编写一个 Makefile 文件，这样不仅方便管理，而且可以迅速发现模块中的问题。这样我们只需要控制其他模块中的 Makefile 就可以实现总体的控制，这就是 make 的嵌套执行。

如何来使用呢？举例说明如下：

```makefile
subsystem:
    cd subdir && $(MAKE)
```

在当前目录下有一个subdir子文件夹，包含一个内容为上述的Makefile，在当前目录执行make subsystem即可按subdir/Makefile中定义的规则对`subdir`下的文件进行编译。

上述的规则也可以换成另外一种写法：

```makefile
subsystem:
		$(MAKE) -C subdir
```

在 make 的嵌套执行中，我们需要了解一个变量 "CURDIR"，此变量代表 make 的工作目录。当使用 make 的选项 "-C" 的时候，命令就会进入指定的目录中，然后此变量就会被重新赋值。总之，如果在 Makefile 中没有对此变量进行显式的赋值操作，那么它就表示 make 的工作目录。我们也可以在 Makefile 中为这个变量赋一个新的值，当然重新赋值后这个变量将不再代表 make 的工作目录。

## export

当嵌套执行make时，如果想要传递变量可以使用`export`关键字。

```makefile
export <variable>
```

如果不需要传递：

```makefile
unexport <variable>
```

> <variable>是变量的名字，不需要使用$字符，如果所有的变量都需要传递，单独使用`export`关键字不加任何参数即可。另外需要注意的是要将make的`export`关键字与shell中的`export`区分开来。

Makefile 中还有两个变量不管是不是使用关键字 "export" 声明，它们总会传递到下层的 Makefile 中。这两个变量分别是 SHELL 和 MAKEFLAGS，特别是 MAKEFLAGS 变量，包含了 make 的参数信息。如果执行总控 Makefile 时，make 命令带有参数或者在上层的 Makefile 中定义了这个变量，那么 MAKEFLAGS 变量的值将会是 make 命令传递的参数，并且会传递到下层的 Makefile 中，这是一个系统级别的环境变量。

make 命令中有几个参数选项并不传递，它们是:"-C"、"-f"、"-o"、"-h" 和 "-W"。如果我们不想传递 MAKEFLAGS 变量的值，在 Makefile 中可以这样来写：

```makefile
subsystem:
		cd subdir && $(MAKE) MAKEFLAGS=
```

## 嵌套执行make的案例

[嵌套执行make的案例](http://c.biancheng.net/view/7157.html)

 # 其他

## .c.o	.s.o

```makefile
.c.o 等价于 %.o:%.c
.s.o 等价于 %.o:%.s
```

