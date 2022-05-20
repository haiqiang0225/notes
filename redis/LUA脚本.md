[toc]

# LUA脚本

> [官方文档](https://www.lua.org/manual/5.0/manual.html#2.1)
>
> [菜鸟教程](https://www.runoob.com/lua/lua-data-types.html)

## 基本数据类型

| 数据类型 | 描述                                                         |
| -------- | ------------------------------------------------------------ |
| nil      | 表示无效值，条件表达式中判定为false                          |
| boolean  | true，false                                                  |
| number   | 双精度类型的实浮点数                                         |
| string   | 字符串由一对双引号或单引号表示                               |
| function | 有c或者Lua编写的函数                                         |
| table    | Lua中的表（table）其实是一个“关联数组”（associative arrays），数组的索引可以是数字、字符串或表类型。在Lua里，table的创建是通过“构造表达式”来完成，最简单的构造表达式是{}，用来创建一个空表。下标从1开始。可以用key访问。 |

可以利用`type`函数测试给定的变量名或者值的类型：

```bash
type("Hello World!")
```

输出：`string`

**声明变量**

Lua声明变量时，不需要指定数据类型：

```lua
local str = "hello"
local num = 21
local flag = true
local arr = {'java', 'python', 'lua'}
-- 可以这样访问name对应的值：
--     1. map[name]
--     2. map.name
local map = {name = 'Jack', age = 21}
-- 访问第一个元素：arr[1]
local arr = {'hello', 'world'}
-- 声明一个全局变量，删除时只需要将值置为nil即可
global_var = 'global'
```

## 语法

### 注释

- 单行注释

    lua中使用`--`作为单行注释的开始。

    ```lua
    -- 一行注释
    ```

- 多行注释

    使用`--[[注释]]--`标识多行注释

    ```lua
    --[[
        注释1
        注释2
        ...
    ]]--
    ```

### 标识符

Lua 标示符用于定义一个变量，函数获取其他用户定义的项。标示符以一个字母 A 到 Z 或 a 到 z 或下划线 `_` 开头后加上 0 个或多个字母，下划线，数字（0 到 9）。

作为约定，以下划线`_`和大写字母组成的标识符作为lua脚本的内部保留关键字，比如`_VERSION`。

### 保留关键字

```lua
       and       break     do        else      elseif
       end       false     for       function  if
       in        local     nil       not       or
       repeat    return    then      true      until     while
```

### 循环

- 循环语法

    ```lua
    while (condition)
    do
        statements
    end
    ```

    ```lua
    -- var 从exp1变化到exp2，每次增长步长为exp3。exp3可选，不指定则默认1
    for var=exp1,exp2,exp3 do
        statements
    end
    ```

    ```lua
    -- 类似 do-while循环
    repeat
        statements
    until (condition)
    ```

    循环控制关键字只有`break`和`goto`，没有`continue`。

    **goto语句**

    ```lua
    -- 语法
    goto label
    
    ::label::
    ```

- 遍历数组

    方法1：普通循环

    ```lua
    local arr = {'hello', 'world', 'lua'}
    for i = 1, #arr do
        print(arr[i])
    end
    ```

    其中`#arr`用于获取arr的长度。

    方法2：

    ```lua
    local arr = {'hello', 'world', 'lua'}
    for index, value in ipairs(arr) do
        print(index, value)
    end
    ```

- 遍历table：

    ```lua
    local map = {name = 'Jack', age = 21}
    for key, value in pairs(map) do
        print(key, value)
    end
    ```

    

### 流程控制

**if**

```lua
if (bool_condition)
then
    statements true  -- 表达式为true时执行
else
    statements false -- 表达式为false时执行
end
```

## lua函数

函数定义：

```lua
[local] function fun_name (arg1, arg2, ..., argn)
    statements
    return res1, res2, ...
end
```

lua中函数可以当成参数传到另一个函数中：

```lua
function myFun(arg)
    print("myfun.print:", arg)
end

function test(arg1, arg2, func)
    func(arg1)
    func(arg2)
end
```

函数也可以赋给变量。

**可变参数**

在函数参数列表中使用`...`表示函数具有可变的参数。可以在可变参数之前加上固定参数，但是可变参数必须位于参数列表的最后位置。

获取所有可变参数可以使用`local args = {...}`

```lua
function test(...)
    local args = {...}
    for i, v in ipairs(args) do
        print(i, v)
    end
end

test("Hello", "World", "lua!")
```

但是这种遍历方式存在问题，问题在于`ipairs`函数在遇到`nil`时会停止，因此如果参数列表中如果存在`nil`的话，就会提前停止遍历。

此外还可以使用`select`函数：

- `select('#', ...)` 返回可变参数的长度
- `select(n, ...)` 返回从n开始到结束位置的所有参数列表。

## Lua运算符

### 算术运算符

lua支持常规的算术运算符：+加法，-减法，*乘法，/除法，%取余，^幂（不是异或），-负号，//整数除法（lua5.3+）。

### 关系运算符

- `==` 等于，检查两侧的值是否相等，相等返回true，否则false
- `~=`不等于，相等false，否则true
- `>`   大于，不能比较`nil`。
- `<`   小于
- `>=` 大于等于
- `<=` 小于等于

### 逻辑运算符

- `and` 逻辑与
- `or`   逻辑或
- `not` 逻辑非

### 其它运算符

- `..` 连接两个字符串
- `#`   返回字符串或表的长度

## Lua stirng

Lua中字符串的表示方式：

- `''` 或者`""`之间的一串字符；
- `[[ str ]]`中的str 