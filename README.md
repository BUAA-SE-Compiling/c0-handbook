# C0 实验指导书

## 1. 概述

本指导书为 C0 指导书，阅读本指导书前，请确保你已经顺利完成了 mini 实验并且对编译器的结构和编译过程有了基本认识。

由于本指导书还只是一个 beta 版本，因此很可能存在一些错误，如果你发现本书中有以下问题

- 逻辑性/知识性错误
- 表意模糊/错误
- 前后矛盾
- 代码不对应/错误
- 示例输出过时
- ...

欢迎积极联系助教，也可以直接提 issue 甚至 pr，勘误或者单纯的建议都会有一定的加分。

## 2. 编译过程概述

见 [miniplc0 指导书](https://github.com/BUAA-SE-Compiling/miniplc0-handbook#2-%E7%BC%96%E8%AF%91%E8%BF%87%E7%A8%8B%E6%A6%82%E8%BF%B0)

## 3. C0 编译系统

C0 是简化了 C 语言语法与编译过程得到的小型编程语言。

通过本章节的实验，你将：

- 阅读 C0 编译器的要求
- 阅读 C0 文法及其语义规则
- 了解一些可用的编译器实现思路
- 了解对于实验提交的要求

### 3.1 编译系统整体结构

C0 编译器的输入是后缀为`.c0`的源代码文件，输出是后缀为`.o0`的二进制目标文件或后缀为`.s0`的文本目标文件。

完成 C0 编译系统，你至少需要实现一个具有[指定接口](#311-编译器接口)的编译器，这个编译器能够完成从C0源代码到二进制[目标文件](#312-目标虚拟机)的翻译。

为了便于自己查看编译结果，你可以选择性地提供一个反汇编器，用以反汇编你的二进制目标文件。

助教之后会提供一个能够解析并运行目标文件的虚拟机。目标文件的结构参见[目标文件](#312-目标虚拟机)的说明。

我们不限制你实现编译器的语言，并且允许复用 mini 实验的代码。

你可以自行设计指令集和虚拟机并给出实现，这意味着需要单独评测，因此不会有额外奖励。

#### 3.1.1 编译器接口

无论你的编译器是什么形态，其至少要能够通过命令行这样使用：

```
Usage:
  cc0 [options] input [-o file]
or 
  cc0 [-h]
Options:
  -s        将输入的 c0 源代码翻译为文本汇编文件
  -c        将输入的 c0 源代码翻译为二进制目标文件
  -h        显示关于编译器使用的帮助
  -o file   输出到指定的文件 file

不提供任何参数时，默认为 -h
提供 input 不提供 -o file 时，默认为 -o out
```

比如：
`cc0 -c ./in.c0 -o abc` 将从运行目录下的`in.c0`作为输入文件，输出二进制目标文件`abc`。
`cc0 -s ./in.c0` 将从运行目录下的`in.c0`作为输入文件，输出文本汇编文件`out`。

#### 3.1.2 目标虚拟机

请先阅读虚拟机说明书中的 [虚拟机结构部分](https://github.com/BUAA-SE-Compiling/c0-vm-standards#%E8%99%9A%E6%8B%9F%E6%9C%BA%E7%BB%93%E6%9E%84)。

你可以不采用这个虚拟机，而选择其他自己设计的或者现成的运行时或架构（比如JVM、x86、LLVM等）。对应的，你就需要提供运行环境的设计和实现。具体参见后文的[提交要求](#35-提交方式和内容要求)部分。

简单来说，你需要编译出的文本汇编具有如下格式（`#`开头的是不必要的行注释，`.o`的格式参见虚拟机说明书）：


```assembly
# 常量表，记录int、double、字符串常量的信息
.constants:
    # 下标  常量的类型 常量的值
    {index} {type}    {value} 
    ...
# 启动代码，负责执行全局变量的初始化
.start:
    #  下标  指令名   操作数
    {index} {opcode} {operands}
    ...
# 函数表，记录函数的基本信息
.functions:
    # 下标   函数名在.constants中的下标 参数占用的slot数 函数嵌套的层级
    {index} {name_index}              {params_size}   {level} 
    ...
# 函数体
.F0:
    # 下标   指令名   操作数
    {index} {opcode} {operands}
    ...
.F1:
    {index} {opcode} {operands}
    ...
...
.F{functions_count-1}:
    {index} {opcode} {operands}
    ...
```

比如编译`.c0`：

```c++
int g0 = 42;
double g1 = 1.0;

int fun(int num) {
    return -num;
}

int main() {
    return fun(-123456);
}
```

得到`.s0`：

```assembly
.constants:
0 S "fun"
1 S "main"
2 I -123456            # 0xFFFE1DC0
3 D 0x3FF0000000000000 # 1.000000
.start:
0    bipush 42
1    loadc  3          # 1.000000
.functions:
0 0 1 1                # .F0 fun
1 1 0 1                # .F1 main
.F0: #fun
0    loada 0, 0
1    iload
2    ineg
3    iret
.F1: #main
0    loadc 2           # -123456
1    call 0            # fun
2    iret
```

或优化`.s0`：

```assembly
.constants:
0 S "main"
1 I 123456
.start:
.functions:
0 0 0 1         # .F0 main
.F0: #main
0    loadc 1    # 123456
1    iret
```

### 3.2 C0 文法与约束

本节内容将对 C0 的各种文法以及约束进行解读。如果想要直接获取完整版本的文法，请移步[附录B](#附录b-c0文法)或github仓库。

#### 3.2.1 可接受字符集

C0的源代码只在ASCII范围内接受如下任意字符：

- 4种空白符：空格（`0x20, ' '`）、水平制表符（`0x09, '\t'`）、换行符（`0x0A, '\n'`）、回车符（`0x0D, '\r'`）
- 10种数字：从`0`到`9`
- 52种英文字母：从`a`到`z`，从`A`到`Z`
- 32种标点字符：`` _ ( ) [ ] { } < = > . , : ; ! ? + - * / % ^ & | ~ \ " ' ` $ # @ ``

字节值不属于上述98种字符的输入，目前都视为非法输入。

在**基础文法**中，`` _ [ ] . : ? % ^ & | ~ \ " ' ` $ # @ `` 不组成任何语法成分，也是不合法的输入。

#### 3.2.2 基础C0

这一部分列出基础C0的各种标准。如果对文法阅读存在理解困难，请先行参阅[附录A](#附录a-ebnf)。

##### 3.2.2.1 保留字

基础C0的保留字有下列这几种：

```c++
<reserved-word> ::= 
     'const'
    |'void'   |'int'    |'char'   |'double'
    |'struct'
    |'if'     |'else'   
    |'switch' |'case'   |'default'
    |'while'  |'for'    |'do'
    |'return' |'break'  |'continue' 
    |'print'  |'scan'
```

保留字不能被词法分析器视为标识符。它们可能是当前语法的关键组成部分；也可能是为了以后的新语法预留的内容，因此暂时不会作为关键字出现在语法中。保留字区分大小写。

##### 3.2.2.2 标识符

基础C0的标识符文法为：

```c++
<nondigit> ::=    'a'|'b'|'c'|'d'|'e'|'f'|'g'|'h'|'i'|'j'|'k'|'l'|'m'|'n'|'o'|'p'|'q'|'r'|'s'|'t'|'u'|'v'|'w'|'x'|'y'|'z'|'A'|'B'|'C'|'D'|'E'|'F'|'G'|'H'|'I'|'J'|'K'|'L'|'M'|'N'|'O'|'P'|'Q'|'R'|'S'|'T'|'U'|'V'|'W'|'X'|'Y'|'Z'
<identifier> ::= 
    <nondigit>{<nondigit>|<digit>}
```

**根据`<identifier>`的文法规则**可以看出：C0的标识符只支持数字和英文字母，且不能以数字开头。标识符区分大小写。

##### 3.2.2.3 类型系统

基础C0只支持两种基础类指示符和一个常类型修饰符：

```c
<type-specifier>         ::= <simple-type-specifier>
<simple-type-specifier>  ::= 'void'|'int'
<const-qualifier>        ::= 'const'
```

这些关键字主要被显式地使用于[变量声明]()、[函数定义]()和[类型转换]()。

`void`是没有值语义的类型，不能参与变量声明（见[变量](#3227-变量)的语义规则）、不能参与求值（见[表达式](#3225-运算符与表达式)的语义规则）。

`int`是32位的有符号整数，以二进制补码的形式存储。

`const`通常与类型一起使用，语义是immutable，说明其修饰的类型**值不可被改变**。

##### 3.2.2.4 整数字面量

基础C0只支持两种类型的整数字面量：十进制整数字面量和十六进制整数字面量。

```c++
<nonzero-digit> ::= 
    '1'|'2'|'3'|'4'|'5'|'6'|'7'|'8'|'9'
<digit> ::= 
    '0'|<nonzero-digit>
<hexadecimal-digit> ::=
    <digit>|'a'|'b'|'c'|'d'|'e'|'f'|'A'|'B'|'C'|'D'|'E'|'F'
<integer-literal> ::= 
    <decimal-literal>|<hexadecimal-literal>
<decimal-literal> ::= 
    '0'|<nonzero-digit>{<digit>}
<hexadecimal-literal> ::= 
    ('0x'|'0X')<hexadecimal-digit>{<hexadecimal-digit>}
```

整数字面量不包含符号，但是其类型是有符号的`int`，这一点在有其他数据类型的情况下是需要注意的一点，后文会提到。

> 注意：**根据`<decimal-literal>`的文法规则**可以看出：非0的十进制整数字面量不能有任何前导0。而 mini 的中`<无符号整数>`的文法规则是允许的，这里的不同之处请留意。

> UB: 虽然字面量有类型意味着其有着受限的值域，但是C0并不要求对溢出进行报错，使用过大的字面量是未定义行为。你可以报出编译错误，也可以截断高位让其自然溢出，甚至选择给出一个善意的warning之后当作无事发生。

##### 3.2.2.5 运算符与表达式

基础C0支持的运算符有如下几种：

```c
<unary-operator>          ::= '+' | '-'
<additive-operator>       ::= '+' | '-'
<multiplicative-operator> ::= '*' | '/'
<relational-operator>     ::= '<' | '<=' | '>' | '>=' | '!=' | '=='
<assignment-operator>     ::= '='
```

运算符在表达式语法中出现的语境是：

```c
<assignment-expression> ::= 
    <identifier><assignment-operator><expression>
    
<condition> ::= 
    <expression>[<relational-operator><expression>]
    
<expression> ::= 
    <additive-expression>
<additive-expression> ::= 
    <multiplicative-expression>{<additive-operator><multiplicative-expression>}
<multiplicative-expression> ::= 
    <unary-expression>{<multiplicative-operator><unary-expression>}
<unary-expression> ::=
    [<unary-operator>]<primary-expression>
<primary-expression> ::=  
     '('<expression>')' 
    |<identifier>
    |<integer-literal>
    |<function-call>
```

根据文法解释运算符优先级和运算符结合性，将运算符优先级从高到低排列为：

- 单目运算符：`+`和`-`，从右到左结合
- 乘法型运算符：`*`和`/`，从左到右结合
- 加法型运算符：`+`和`-`，从左到右结合
- 关系运算符：`<`、`<=`、`>`、`>=`、`==`、`!=`，从左到右结合
- 赋值运算符：`=`，从右到左结合

> 关系运算符在`<condition>`的规则中至多允许出现一次，因此其结合性可能用不到。赋值语句的右侧不接受关系表达式，因此赋值运算符和关系运算符的优先级关系也可能用不到。

从语义上，要求：

- 表达式的值和类型取决于运算结果，且运算数都必须是有值的（不能是`void`类型、不能是函数名）：
  - `<primary-expression>` 的类型和值，与其推导出的语法成分完全相同
  - 对于单目的算术表达式（`<unary-expression>`），其类型和操作数（`<primary-expression>`）相同，值等同于对操作数进行算术取负的结果
  - 对于双目的算术表达式（`<additive-expression>`、`<multiplicative-expression>`），值等同于对操作数进行对应算术运算的结果
  - 顶级表达式`<expression>`的类型和值，与其推导出的`<additive-expression>`完全相同
- `<condition>`只有`true`和`false`的逻辑语义，其运算数都必须是有值的（不能是`void`类型、不能是函数名）
  - `<condition> ::= <expression>`时，如果`<expression>`是（或可以转换为）`int`类型，且转换得到的值为`0`，那么视为`false`；否则均视为`true`。
  - `<condition> ::= <expression><relational-operator><expression>`时，根据对应关系运算符的语义，决定是`true`还是`false`
- `<assignment-expression>`没有值语义
- `<assignment-expression>`左侧的标识符的值类型必须是可修改的变量（不能是`const T`、不能是函数名）
- `<assignment-expression>`右侧的表达式必须是有值的（不能是`void`类型、不能是函数名）

> 注意： 基础C0只有`int`这一种有值类型，因此不涉及类型转换的问题

> UB:  关系表达式`<condition>`在这里没有规定实际类型，只说明了什么样的情况应该视为true或false，但是从未说明其本身的值应该是多少（即没有说必须是0或1）
>
> （因为它不出现在赋值语句右侧以及函数传参，因此这里不进行强制约束了）

##### 3.2.2.6 程序结构

C0的源代码结构如下：

```c++
<C0-program> ::= 
    {<variable-declaration>}{<function-definition>}
```

即最外层作用域（全局作用域）中只有变量声明和函数定义，且变量声明一定先于所有函数定义。

从语义上，要求C0源代码中必须定义`main`函数。

有关作用域、变量声明和函数定义详见后文内容。

##### 3.2.2.7 变量

C0的变量声明规则如下：

```c
<variable-declaration> ::= 
    [<const-qualifier>]<type-specifier><init-declarator-list>';'
<init-declarator-list> ::= 
    <init-declarator>{','<init-declarator>}
<init-declarator> ::= 
    <identifier>[<initializer>]
<initializer> ::= 
    '='<expression>
```

一个正确的变量声明必须要有类型和标识符，而初始化值和`const`修饰符均是可选的。

从语义上，有如下要求：

- 变量的类型，不能是`void`或`const void`

- `const`修饰的变量必须被显式初始化：

  ```c++
  const int a = 1;
  const int b; // error
  ```

- `const`修饰的变量不能被修改：

  ```c++
  const int a = 1;
  int main() 
  {
      a = 2; // error
      scan(a); // error
  	return 0;
  }
  ```

- 声明为变量的标识符，在同级作用域中只能被声明一次

有关作用域详见后文内容。

> UB: 未初始化的非`const`变量，C0不要求对其提供默认值，因此使用它们是未定义行为。你可以为他们指定一个默认值，也可以选择报出编译错误，也可以给出一个warning之后当作无事发生。

##### 3.2.2.8 函数

C0与mini的一大区别，在于其支持函数声明与调用：

```c++
<function-definition> ::= 
    <type-specifier><identifier><parameter-clause><compound-statement>

<parameter-clause> ::= 
    '(' [<parameter-declaration-list>] ')'
<parameter-declaration-list> ::= 
    <parameter-declaration>{','<parameter-declaration>}
<parameter-declaration> ::= 
    [<const-qualifier>]<type-specifier><identifier>
    
<function-call> ::= 
    <identifier> '(' [<expression-list>] ')'
<expression-list> ::= 
    <expression>{','<expression>}
```

参数声明比较类似变量声明，等价于在函数体内进行的局部变量声明，只不过一定会通过函数调用被初始化。

函数体的组成单位是语句：

```c
<compound-statement> ::= 
    '{' {<variable-declaration>} <statement-seq> '}'
<statement-seq> ::= 
	{<statement>}
<statement> ::= 
     '{' <statement-seq> '}'
    |<condition-statement>
    |<loop-statement>
    |<jump-statement>
    |<print-statement>
    |<scan-statement>
    |<assignment-expression>';'
    |<function-call>';'
    |';'

<jump-statement> ::= <return-statement>
```

从函数返回需要使用返回语句：

```c
<return-statement> ::= 'return' [<expression>] ';'
```

从语义上，有如下要求：

- 函数名不能被修改，也不可读（因此不参与计算）
- 参数的类型，不能是`void`或`const void`
- `const`修饰的参数不能被修改
- 声明为函数的标识符，在同级作用域中只能被声明一次
- 声明为参数的标识符，在同级作用域中只能被声明一次
- 函数调用的标识符，必须是当前作用域中可见的，被声明为函数的标识符
- 函数传参均为拷贝传值，在被调用者中对参数值的修改，不应该导致调用者也被修改
- 函数调用的传参数量以及每一个参数的数据类型（不考虑`const`），都必须和函数声明中的完全一致
- 不能在返回类型为`void`的函数中使用有值的返回语句，也不能在非`void`函数中使用无值的返回语句。

有关作用域详见后文内容。

> UB: 函数中的控制流，如果存在没有返回语句的分支，这些分支的具体返回值是未定义行为。**但是你的汇编必须保证每一种控制流分支都能够返回。**

##### 3.2.2.9 条件语句

C0支持最简单的if-else条件语句：

```c
<condition-statement> ::= 
    'if' '(' <condition> ')' <statement> ['else' <statement>]
```

由于`<statement>`也能够推导出`<condition-statement>`，因此这里存在if-if-else这个经典的二义性问题。我们在这里遵循的原则是：else总是匹配前文中距离其最近的尚且没有和else匹配的if，例：

```c
if(i<0) // if1
	if (i>0) // if2
	    i = i+1;
	else // match if2
	    i = 1;
```

`if`语句的流程是：

- 求值`<condition>`
  - 如果`<condition>`是`true`，控制进入`if`的代码块
  - 如果`<condition>`是`false`，控制进入`else`的代码块

##### 3.2.2.10 循环语句

C0支持最简单的while循环语句：

```c++
<loop-statement> ::= 
    'while' '(' <condition> ')' <statement>
```

`while`语句的流程是：

1. 求值`<condition>`
2. 如果`<condition>`是`false`，跳转到步骤5
3. 控制进入`while`的代码块并顺序执行
4. 控制达到`while`代码块的尾部时，跳转到步骤1
5. 控制跳过`while`结构，执行之后的代码块

##### 3.2.2.11 输入输出语句

由于C0本身支持的内容过少（系统相关的IO、变长参数、字符串、指针），不足以支撑我们熟悉的`scanf`函数和`printf`函数。C0选择了一种暴力直接的思路：将`scan`/`print`视为关键字，作为平台无关的语言内置函数使用。

```c
<scan-statement>  ::= 'scan' '(' <identifier> ')' ';'
<print-statement> ::= 'print' '(' [<printable-list>] ')' ';'
<printable-list>  ::= <printable> {',' <printable>}
<printable> ::= <expression>
```

语义上，要求：

- `scan`的`<identifer>`必须是非`const`的变量，必须是可修改的
- `print`的`<expression>`求值后的类型不能是`void`
- `print`的`<expression>`求值后的类型决定了`print`输出的类型
- `print`最后会输出一个换行
- 一个`print`有多个`<printable>`时，`<printable>`之间输出一个空格(`bipush 32` + `cprint`)

> UB: scan和print是目标机有关的内容，当I/O流出现问题时，它们的表现是未定义的

##### 3.2.2.12 符号 作用域 生命周期

C0中每一层大括号都是一级作用域，最外层是全局作用域（层级记为0）。一个符号的生存周期和其被声明的作用域相同。

```c
// level 0
int a;

int fun() {
    // level 1
    int a;    
    
    {
        // level 2
    }
    
    if (1) {
        // level 2
        while (0) {
            // level 3
        }
    }
    return 0;
}
```

与作用域相关的语义规则如下：

- 在同一作用域中，一个标识符只能声明一次（无论已经被声明为变量还是函数）
  - C语言不支持函数重载，基础C0也是如此
  - 函数的参数，等同于在该函数体作用域中声明的局部变量，因此不能在函数体内声明和他同名的其他局部变量
- 如果在当前作用域声明了外层作用域已经声明过的标识符，那么在当前作用域中，外层作用域的定义暂时失效（外层符号的定义被暂时覆盖），无论外层同名的是函数还是变量
  - 由于函数名的作用域是其被声明的作用域，而函数的参数名或局部变量名作用域是函数体内部，相对函数名的作用域是内层，因此函数内部如果存在与其同名的变量或参数，将导致无法在此函数内调用自身（无法递归）。
- 如果引用的符号在当前作用域内没有声明过，就从内到外地逐级查找外层作用域中的定义
- 不能使用当前作用域和外层作用域都没有声明过的标识符
  - 使用当前或外层作用域**之后**会声明的标识符，是未定义行为。由于基础文法限定只能在每个作用域的开头进行变量声明，因此不会发生这种问题。

> **特别注意**：虚拟机指令`loada level_diff, offset`的`level_diff`指的不是这里源代码层面的作用域层次差，而是**函数嵌套的层数差**。由于C0不允许函数嵌套，因此整个实验中`level_diff`的值只可能有`0`（函数中加载局部变量/参数、全局中加载全局变量）和`1`（函数中加载全局变量）。

#### 3.2.3 扩展C0

扩展 C0 是对C0语法的扩充，以一些可选的附加项出现在本实验中，本节后续内容给出这些附加项的解释。

在每个附加项的最后会给出分值系数（如果有）。

##### 3.2.3.1 注释

```c++
<single-line-comment> ::=
    '//' {<any-char>} (<LF>|<CR>)
<multi-line-comment> ::= 
    '/*' {<any-char>} '*/'
```

其中：
- `<any-char>`是任意值的字节（包括可接受字符集之外的）
- `<LF>`是ascii值为0x0A的字符
- `<CR>`是ascii值为0x0D的字符

严格地讲，注释并不属于语法成份，不应该被词法分析输出。

注释的分析**不遵循**最大吞噬规则：

- 单行注释内容的分析遇到第一个 0x0A 或 0x0D 字节就立即结束
- 多行注释内容的分析遇到第一个`*/`序列就立即结束

系数：1

##### 3.2.3.2 字符字面量与字符串字面量

```c
<char-liter> ::= 
    "'" (<c-char>|<escape-seq>) "'" 
<string-literal> ::= 
    '"' {<s-char>|<escape-seq>} '"'
<escape-seq> ::=  
      '\\' | "\'" | '\"' | '\n' | '\r' | '\t'
    | '\x'<hexadecimal-digit><hexadecimal-digit>

<printable> ::=
    <expression> | <string-literal> | <char-literal>
```

其中：

- `<s-char>`可以是[可接受字符集](#3.2.1-可接受字符集)中，除了双引号`"`、反斜线`\`、换行符（`0x0A, \n`）、回车符（`0x0D, \r`）这四种字符的其他任意单字节字符。
- `<c-char>`可以是[可接受字符集](#3.2.1-可接受字符集)中，除了单引号`'`、反斜线`\`、换行符（`0x0A, \n`）、回车符（`0x0D, \r`）这四种字符的其他任意单字节字符。
- 上述两种单字节字符，以及转义字符中的`\x??`十六进制字符，值域均为0到0xff(255)

下列示例输出 `hello world!`：

```c
print("hello\x20world!");
```

> UB: 这里并不要求char字面量可以参与表达式运算

系数：2

##### 3.2.3.3 循环语句与跳转语句

```c++
<statement> ::= 
     '{' <statement-seq> '}'
    |<condition-statement>
    |<loop-statement>
    |<jump-statement>
    |<print-statement>
    |<scan-statement>
    |<assignment-expression>';'
    |<function-call>';'
    |';'
    
<jump-statement> ::= 
     'break' ';'
    |'continue' ';'
    |<return-statement>
    
<loop-statement> ::= 
    'while' '(' <condition> ')' <statement>
   |'do' <statement> 'while' '(' <condition> ')' ';'
   |'for' '('<for-init-statement> [<condition>]';' [<for-update-expression>]')' <statement>

<for-init-statement> ::= 
    [<assignment-expression>{','<assignment-expression>}]';'
<for-update-expression> ::=
    (<assignment-expression>|<function-call>){','(<assignment-expression>|<function-call>)}
```

`do-while`语句的流程是：

1. 控制进入`do-while`的代码块并顺序执行
2. 控制到达`do-while`代码块的尾部时，求值`<condition>`
3. 如果`<condition>`是true，跳转到步骤1
4. 控制离开`while`结构，执行之后的代码块

`for`语句的流程是：

1. 控制顺序进入执行`<for-init-statement>`
2. 求值`<condition>`
3. 如果`<condition>`是false，跳转到步骤1
4. 控制进入`for`代码块并顺序执行
5. 控制到达`for`代码块的尾部时，执行`<for-update-expression>`
6. 跳转到步骤2

语义规则：

- `break`和`continue`可以在循环体内使用，在其他地方使用是编译错误
- `break`代表跳出循环体，控制转移到循环外的下一条语句
- `continue`代表跳过本次循环体的代码，控制转移到循环体的最后一条语句
- `for`不提供`<condition>`时，默认是`1`

`break`和`continue`的解释例：

```c
while (i) {
    i = i+1;
    // continue跳转到这里
}
// break跳转到这里

for (;i;i=i+1) {
    i = i+1;
    // continue跳转到这里
}
// break跳转到这里

do {
    i = i+1;
    // continue跳转到这里
} while(i);
// break跳转到这里
```

基础C0不要求循环结构能够支持`break`或`continue`，但是一旦你选择了这条，你就**必须**要为你所有支持的循环提供`break`和`continue`。

系数：3(do) / 5(for)

##### 3.2.3.4 switch 与 break

```c
<statement> ::= 
     '{' <statement-seq> '}'
    |<condition-statement>
    |<loop-statement>
    |<jump-statement>
    |<print-statement>
    |<scan-statement>
    |<assignment-expression>';'
    |<function-call>';'
    |';'
    
<jump-statement> ::= 
     'break' ';'
    |<return-statement>

<condition-statement> ::= 
     'if' '(' <condition> ')' <statement> ['else' <statement>]
    |'switch' '(' <expression> ')' '{' {<labeled-statement>} '}'
<labeled-statement> ::= 
     'case' (<integer-literal>|<char-literal>) ':' <statement>
    |'default' ':' <statement>
```

语义规则：

- `switch`的`<expression>`必须是整数类型的（`int`或`char`），不允许类型转换
- 每一种标签（`default`、特定值的`case`）只能出现一次
- `break;` 只能出现在 `switch` 的语句块中
- `break`代表跳出`switch`语句块，控制转移到`switch`外的下一条语句
- 如果一个标签的最后没有`break;`，那么控制会继续流入下一个标签

基础C0不要求循环结构能够支持`break`，即使你选择了这条，也**不需要**你为所有支持的循环提供`break`。

> UB: 如果`default`标签不是`switch`中的最后一个标签，那么这是未定义行为

系数： 5

##### 3.2.2.5 作用域与生命周期

```c++
<compound-statement> ::= 
    '{' {<variable-declaration>} <statement-seq> '}'
<statement-seq> ::= 
	{<statement>}
<statement> ::= 
     <compound-statement>
    |<condition-statement>
    |<loop-statement>
    |<return-statement>
    |<print-statement>
    |<scan-statement>
    |<assignment-expression>';'
    |<function-call>';'
    |';'
```

简而言之，可以在任何一个作用域的最前端声明变量。

语义要求：
- 定义在每个作用域内部的变量，只在该作用域内可见，生命周期和该作用域相同
- 定义在循环体内的变量，生命周期只有一次迭代，每次迭代的值不保留

系数： 3

##### 3.2.2.6 类型转换

类型转换是将一个**有值类型**的值转换为**某个**指定**有值类型**的值的操作，C0的有值基础类型都是可以互相转换的。

类型转换分为显示类型转换和隐式类型转换。

隐式类型转换主要发生在求值和赋值的场合：

- 对于双目的算术表达式（`<additive-expression>`、`<multiplicative-expression>`），如果两操作数类型不同，应隐式地将较小的类型转换至较大的类型，最终得到的结果类型和类型较大的操作数一致，值等同于对操作数进行对应算术运算的结果
- 对于双目的关系表达式（`<condition> ::= <expression><relational-operator><expression>`），如果两操作数类型不同，应隐式地将较小的类型转换至较大的类型，根据对应关系运算符的语义，决定是`true`还是`false`
- 对于赋值表达式`<assignment-expression>`以及带有初始化的变量声明`<init-declarator>`，如果`=`运算符两侧的类型不同，应该将右侧表达式隐式转换为左侧标识符的类型
- 如果函数调用的传参数量和声明所需的数量一致，但是参数的类型不匹配，应当对传入参数进行隐式转换
- 如果函数`return`语句的表达式类型和函数声明的返回值类型不一致，应当对该表达式进行隐式类型转换后再返回

显式类型转换：

```c
<multiplicative-expression> ::= 
     <cast-expression>{<multiplicative-operator><cast-expression>}
<cast-expression> ::=
    {'('<type-specifier>')'}<unary-expression>
<unary-expression> ::=
    [<unary-operator>]<primary-expression>
```

根据类型转换的定义以及[表达式](3225-运算符与表达式)“任何运算数的类型不能是`void`”的语义要求：

- 无论`<cast-expression>`的目标类型是什么，只要操作数`<unary-expression>`的类型是`void`，都**是**语义错误
- 只要`<cast-expression>`的目标类型是`void`，无论操作数`<unary-expression>`的类型是什么，都**是**语义错误

##### 3.2.2.7 char

```c
<type-specifier>         ::= <simple-type-specifier>
<simple-type-specifier>  ::= 'void'|'int'|'char'
<const-qualifier>        ::= 'const'

<primary-expression> ::=  
     '('<expression>')' 
    |<identifier>
    |<integer-literal>
    |<char-literal>
    |<function-call>
```

`char`是8位的无符号整数，以二进制原码的形式存储。

字符字面量的默认类型是`char`。

`char`类型参与**任何运算**之前，都先被隐式地转换为`int`，运算结果的类型也是`int`。

> 注意： 为了实现 char，你**必须**先实现[类型转换](#3226-类型转换)和[字符字面量与字符串字面量](#3232-字符字面量与字符串字面量)。

系数：9 (字面量+类型转换+char)

##### 3.2.2.8 double

```c
<sign> ::= 
    '+'|'-'
<digit-seq> ::=
    <digit>{<digit>}
<floating-literal> ::= 
     [<digit-seq>]'.'<digit-seq>[<exponent>]
    |<digit-seq>'.'[<exponent>]
    |<digit-seq><exponent>
<exponent> ::= 
    ('e'|'E')[<sign>]<digit-seq>

<type-specifier>         ::= <simple-type-specifier>
<simple-type-specifier>  ::= 'void'|'int'|'double'
<const-qualifier>        ::= 'const'

<multiplicative-expression> ::= 
     <cast-expression>{<multiplicative-operator><cast-expression>}
<cast-expression> ::=
    {'('<type-specifier>')'}<unary-expression>
<unary-expression> ::=
    [<unary-operator>]<primary-expression>
<primary-expression> ::=  
     '('<expression>')' 
    |<identifier>
    |<integer-literal>
    |<floating-literal>
    |<function-call>
```

`double`是64位的有符号浮点数，存储遵循IEEE754标准。

浮点字面量的默认类型是`double`。

> 注意： 为了实现 double，你**必须**先实现[类型转换](#3226-类型转换)。

系数：10 (字面量+类型转换+double)

### 3.3 实现指引

本部分将大致介绍亲手实现一个编译器的思路。

#### 3.3.1 词法分析

词法分析的任务是一个将源代码分割成 token 的序列。过程中还要对于不可接受的输入、不合法的token形态上报错误。

当 token 的形态比较简单时，我们只根据第一个字符就可以确认这个 token 应该是什么类型。比如我们的 mini 实验中，只有标识符（和关键字）、无符号整数、各种标点符号，它们的 FIRST 集合是完全不相交的。

这种情况下最简单的实现思路就是预读第一个字符，根据其所属的FIRST集合，分发到对应的处理子程序：

```c++
Token token;
char ch = nextChar();
if (isalpha(ch)) {
    token = lex_an_identifier(ch);
    // 判断并返回token
}
else if (isdigit(ch)) {
    token = lex_an_integer(ch);
    // 判断并返回token
}
/* 其他FIRST集合 */
```

当 token 的 FIRST 集合有相交部分时，这种简单的写法往往不能直接适应。

比如 C0 同时有十进制整数、十六进制整数、浮点数等字面量，词法分析器读取到`0`并不能知道立刻知道它是哪一种形态。你当然可以让词法分析器对每一种可能都进行尝试，这样的代价是每次发生处理错误都需要进行回溯，而回溯往往是我们最不希望发生的事情，因为没有意义的尝试会增加运行开销，也会给debug带来难度。类似条件语句的“短路”，尝试每一种可能对于代码的修改也很敏感。

回到`0`的问题，除非我们继续读下一个字符：如果是`x`或`X`，可以确定它是十六进制表示；如果是`0123456789.eE`中的一个，就可以确定它是浮点数：

```c++
if (ch == '0') {
    char next = nextChar();
    if (/* next in "xX" */) {
        // 十六进制
    }
    else if (/* next in "0123456789.eE" */) {
        // 浮点数
    }
    else {
        // 十进制整数0
    }
}
else if (/* ch in "123456789" */) {
    // 整数? 浮点数?
}
```

如果 FIRST 集合相似的程度继续增大，会有更多层的if-else嵌套，对于可维护性来说是很大的打击。

在 mini 实验的时候，我们采用的是维护一个临时的自动机。自动机每次根据当前的状态和输入的下一个字符进行状态转移，在不能继续转移时输出 token。它在处理 FIRST 集合相交的问题时，则是：

```c++
case INITIAL_STATE: {
    if (ch == '0') {
        state = ZERO_STATE;
        /* */
    }
    else {/* */}
}; break;

case ZERO_STATE: {
    if (/* ch in "xX" */) {
        state = HEX_X_STATE;
        /* */
    }
    else if (ch == '.') {
        state = FLOATING_DOT_STATE;
        /* */
    }
    else if (/* ch in "Ee" */) {
        state = FLOATING_E_STATE;
        /* */
    }
    else {/* */}
}; break;
```

另外，基于自动机的方法只要得到了识别 token 的状态图，就可以直观地对着图构造出词法分析器。而且比较适合用于自动化生成词法分析器。

如果说最开始的写法的维护单位是每一个语句块，我们需要根据语境来分配处理子函数。那么自动机的维护单位则是每一个状态，状态自带语境，不需要考虑它的上一个状态是什么，只需要根据输入填上对应的转移和操作即可。状态自带语境这一点，也非常有利于发现错误时精准报错。当发现需要对分析器进行调整时，if-else需要调整嵌套的层间关系、层内顺序和调用的子程序，而自动机则是对节点和边进行增删改。

自动机可以通过主动降低可维护性的方式换取简洁和性能：比如用一个二维数组描述状态转移矩阵（`state2 = f[state1][input]`），值不值得就见仁见智了。当然，对于较为简单的自动机完全可以删除状态节点并退化为调用处理子程序：
```c++
case INITIAL_STATE: {
    if (isalpha(ch)) {
        lex_an_identifier(ch);
        /* */
    }
    else {/* */}
}; break;
```

gcc 和 chromium 在实现词法分析时采用的就是手工维护自动机，有兴趣的同学可以去学习源码。

#### 3.3.2 语法分析

语法分析根据词法分析输出的 token 来判断源代码是否符合语法结构。但是对于文法规则没办法描述的内容，比如语义规则，就有些鞭长莫及了。

语法分析器通常会输出一棵抽象语法树（AST），语义分析器/代码生成器则会根据语法树进行后续操作。在语义规则较简单的情况下（特别是单一数据类型时），也可不使用AST的实现，参见[3.3.5-语法制导翻译](#335-语法制导翻译)。

比如对于一个简单文法：

```
<binary-expr>  ::= <primary-expr> <operator> <primary-expr>
<primary-expr> ::= <integer>
<operator>     ::= '+' | '-'
```

这里有一种可行的表达式语法树节点定义：

```c++
enum OP { ADD, SUB };

struct AST {};

struct ExprAST : AST {};

struct BinaryExprAST : ExprAST {
    // 运算符
    OP op;
    // 左右操作数
    ExprAST *lhs, *rhs;
    
    BinaryExprAST(OP op, ExprAST *lhs, ExprAST *rhs)
        : op(op), lhs(lhs), rhs(rhs) {}
};

struct IntExprAST : ExprAST {
    int value;
    IntExprAST(int value) : value(value) {}
};
```

那么对于`1+2`，那么我们期望的语法树可能长这样：
```
           BinaryExprAST
                |          
         /------+------\
         |      |      |
IntExprAST:1  Op:+  IntExprAST:2
```

语法分析主要分为自顶向下和自底向上两种分析思路。这里只介绍这两者中比较有代表性的，并且比较适合手工维护的方法：递归下降分析和算符优先分析。

> gcc 使用手工维护的 LR 分析器，这是一种基于自动机的分析方法。如同在词法分析章节中提到的，自动机节点自带语境，当进行输出和报错时非常精准。
>
> 但是语法往往复杂得多，使用LR分析构造自动机时，一个只有个位数规则的语法都可以产生几十种状态；而拥有数十条规则的C0，手动维护其LR分析器对我们的实验来说有点承受不起（笔者至今忘不了一道LR课后习题写一晚上的震撼）。

递归下降分析最大的优点就是形态简单，对于一条规则，只要对其右侧的语法成分顺序调用子程序处理即可。比如对于上面的简单文法例子：

```c++
int main() {
    nextToken();
    ExprAST* ast = parseBinaryExpr();
    // 用ast做语义分析、代码优化和代码生成
}

// 存在内存泄漏的风险，C++使用智能指针会好一点
ExprAST* parseBinaryExpr() {
    ExprAST* lhs = parsePrimaryExpr();
    if (lhs == nullptr) {
        return nullptr;
    }
    
    OP op;
    nextToken();
    if (/* currentToken is ADD */) {
        op = ADD;
    }
    else if (/* currentToken is SUB */) {
        op = SUB;
    }
    else {
        /* record(or report) the error */
        return nullptr;
    }
    
    nextToken();
    ExprAST* rhs = parsePrimaryExpr();
    if (rhs == nullptr) {
        return nullptr;
    }
    
    return new BinaryExprAST(op, lhs, rhs);
}

ExprAST* parsePrimaryExpr() {
    if (/* currentToken is integer */) {
        return new IntExprAST(/* value of the integer */);
    }
    else {
        /* record(or report) the error */
        return nullptr;
    }
}
```

可以看到递归下降分析生成语法树节点的顺序是“先子节点，后父节点”，这和在完整的语法树上进行后序遍历的顺序相同。

递归下降分析法的一个天敌是左递归文法：
```
<A> ::= <A><OP><M> | <M>
```
该文法中的左递归会导致递归下降分析器陷入死循环，因此如果要编写递归下降分析器，首先就应该通过改写文法来排除这种错误：
```
<A> ::= <M>{<OP><M>}
```

另外由于函数递归本身的开销就很大，当递归下降分析发生回溯时，一些较为复杂的规则会下一子带来很多次的递归，比如 C0 文法中的变量声明和函数定义（做了简化处理）：

```
<var-decl> ::= [<const>] <type> <id> ['=' <expr>] ';'
<func-def> ::=           <type> <id> '(' <param-decl-list> ')'
```

如果没有`const`修饰，那么除非完整地读了类型和标识符，否则我们是无法得知到底该去分析变量声明还是函数定义。如果你忠实地遵循上面两条规则写了递归下降分析程序，你很可能会遇到不得不在变量声明中连续回溯两个 token 再去函数定义中重新读它们的情况。

这种情况下，你或许可以对规则进行修改，使得你的递归下降子程序不会遇到回溯，代价则是你的递归下降子程序变得和文法规则长得不那么像，降低了一些可读性。

另一种回溯的情况是语句：
```
<stmt> ::= 
     <if-else-stmt>
    |<while-stmt>
    |<return-stmt>
    |...
```

如果你直接让程序去尝试逐候选项分析，那么如果当前要识别的语句在这个候选列表的最后一个，就会遭遇很多次回溯：

```c++
AST* parseStmt() {
    AST* ast;
    if ((ast = parseIfElseStmt()) != nullptr) {
        return ast;
    }
    if ((ast = parseWhileStmt()) != nullptr) {
        return ast;
    }
    /* */
}
```

假如说其实当前分析的语句就是if-else，只不过分析时发生了错误导致返回`nullptr`，这样的写法依然会去尝试后面的每一种可能。

比较显然的是，右侧候选项的第一个 token 都不一样，如果你在调用子程序前进行分发，就至少可以避免这种回溯：
```c++
AST* parseStmt() {
    switch(/* type of currentToken */) {
    case /* IF */:    return parseIfElseStmt();
    case /* WHILE */: return parseWhileStmt();
    /* */
    }
}
```

有些时候为了避免回溯会牺牲代码简洁性，具体要性能还是要简洁，依然取决于你。（~~我全都要~~）

对于不是表达式的内容，比如条件语句，可以类似地仿照推导语法树的模样来定义：
```c++
struct StmtAST : AST {};

struct IfStmtAST : StmtAST {
    ExprAST* condtion;
    StmtAST* thenStmt;
    StmtAST* elseStmt;
};

struct CompoundStmtAST : StmtAST {
    std::vector<StmtAST*> stmts; // StmtAST* stmts[];
}
```

C0 的表达式文法比较复杂，每次分析操作数都要递归到最深层的`<primary-expr>`，并且要为每一层表达式都编写长得很相似的递归分析子程序。

算符优先分析可以很好地解决递归下降分析的这个问题，因为文法是已知的，我们可以手动构造算符优先矩阵或算符优先函数，使得表达式的分析可以在一个函数内集成。每次入栈是读取 token 并将其作为运算数压入，出栈则是根据运算符类型生成对应的语法树节点：

```c++
// 没有考虑栈空的情况
ExprAST* parseExprByOPA() {
    std::stack<AST*> operands;
    std::stack<OP> operators;
    while(true) {
        if (/* currentToken is operator */) {
            // compare operator precedence
            if (/* stack.top() <= currentToken */) {
                // push currentToken OP onto operator stack
            }
            else {
                // pop 
                // build ast 
                // push ast onto operand stack
                // until <=
            }
        }
        else {
            ExprAST* ast = parsePrimaryExpr();
            if (ast != nullptr) {
                // push ast onto operand stack
            }
            else { /* */ }
        }
    }
}
```

可以证明的是，算符优先分析构造语法树的顺序也和后序遍历相同。

#### 3.3.3 回溯与缓冲机制

无论是词法分析还是语法分析，都依赖于一个输入值来推动分析：词法分析需要字符，语法分析需要 token。当有很多种可能的推动分支时，就会面临回溯问题。前文中也提到了一些避免回溯的手段，有些需要对规则进行改写，有些需要预先读取输入。

为了避免回溯而进行的预读，在之后还有有可能被多次输入：比如词法分析正在分析整数`123`，下一个输入是`+`，如果词法分析使用的是read（next，从输入中取走下一个值），那么这个字符必须要unread（把取走的值放回输入），否则下一次词法分析很可能就会漏掉它；如果词法分析实现的是peek（获得下一个值，但是不从输入中取走它），那么这一个字符下一次执行next还会再次被读。

不管是read+unread还是peek，这个值都将被反复读取。如果不进行缓存，对于词法分析程序只是回溯一个字符；但对于语法分析程序而言，反复调用词法分析程序的代价是比较大的。而如果你提供了一个缓冲区来存储会被多次读取的输入，至少可以避免重复分析。与实现相关的问题还有：缓冲区开多大合适？具体如何维护？

#### 3.3.4 语义分析与代码生成

如前文所言，语法分析只关心语法，不关心诸如“有没有重定义”、“是不是在`void`函数`return`了`int`”这些语义内容。这些都是语义分析考虑的。

依然采用语法分析中的`1+2`例子，如果我们为每一种语法树节点增加生成代码的功能：

```c++
enum class OP { ADD, SUB };

struct ExprAST {};

struct BinaryExprAST : ExprAST {
    // 运算符
    OP op;
    // 左右操作数
    ExprAST *lhs, *rhs;
    
    BinaryExprAST(OP op, ExprAST *lhs, ExprAST *rhs)
        : op(op), lhs(lhs), rhs(rhs) {}
        
    /* */ generate() {
        lhs->generate();
        rhs->generate();
        switch(op) {
        case OP::ADD: output("iadd"); break;
        case OP::SUB: output("isub"); break;
        default: /* error */
        }
        return /* */;
    }
};

struct IntExprAST : ExprAST {
    int value;
    
    /* */ generate() {
        output("ipush %d", value);
        return /* */;
    }
};
```

简而言之，`generate()`函数的作用就是执行该节点的语义分析以及代码生成，你所需要做的就是在在对应节点的`generate()`中实现该节点的语义动作（比如字面量节点执行加载常量、变量声明节点操作符号表）。只要调用语法树根部的`generate()`，就能够得到整棵语法树对应的代码了。

注意`generate()`的返回类型没有给出，这是因为我们并没有考虑语义规则。事实上这里可以返回中间代码块，也可以返回子节点的类型和值信息，也可以返回其它内容。

如果要对复杂语义进行考虑，比如现在`BinaryExprAST`的两个操作数可能一个是`int`一个是`double`，那么它在生成代码时就不应该按部就班地生成`iadd`或`isub`，而是`dadd`或`dsub`，因为`int`在这里要发生隐式的类型提升。这个时候表达式节点就应当附加一个类型属性，或是在`generate()`返回值中提供类型信息。

如果要考虑各种控制结构，跳转指令的代码回填的是一个比较大的问题。这时返回类型如果能设置为代码块的信息则会是相当不错的选择。

mini 实验中采用的是语法分析制导翻译，因为遍历语法树执行语义动作的顺序实际上和递归下降分析的顺序完全一致。需要注意的是，使每一个递归下降子程序考虑各种语义规则，很容易让函数体不那么简洁（特别是考虑错误处理的情况下）。

但从实用的角度考虑，很多语义分析的内容在语法分析阶段也可以较轻松地完成，比如符号表的增删改查。但是有些内容在语法分析阶段一起完成会显著增大代码体积，比如上文提到的不同类型之间的运算，以及控制结构的跳转。实现细节依然取决于个人。

#### 3.3.5 语法制导翻译

所谓语法制导翻译，指的是一边进行语法分析一边进行语义分析和代码生成。我们在 mini 实验采用的就是这种策略。上一节语义分析中提到了遍历语法树的顺序实际和递归下降分析的路径完全相同，因此直接在递归下降分析子程序中添加对应语法节点的语义分析代码即可，比如：

```c++
/* */ parseBinaryExpr() {
    parsePrimaryExpr();
    
    OP op;
    auto opToken = nextToken();
    
    nextToken();
    parsePrimaryExpr();
    
    if (/* opToken is ADD */) {
        output("iadd");
    }
    else if (/* opToken is SUB */) {
        output("isub");
    }
    else {/* */}
}

/* */ parsePrimaryExpr() {
    if (/* currentToken is integer */) {
        output("ipush %d", /* value of the integer*/);
    }
    else {/* */}
}
```

#### 3.3.6 错误处理

构建一个错误处理系统，目前课程没有要求，留待以后完善。

#### 3.3.7 代码优化

进行机器无关的优化。目前课程没有要求，留待以后完善。

### 3.4 评测方式

我们会采用半自动的方式来评测代码。

你的编译器必须实现本指导书中要求的接口，我们会使用预先写好的测试脚本，让你的编译器接受我们的代码输入，之后用虚拟机运行程序和期望输出进行比对以确定是否正确实现。

同时我们也会手动去检查代码。

### 3.5 提交方式和内容要求

对于提交物的要求：

- **编译环境和miniplc0相同**，请务必保证在[这个镜像](https://hub.docker.com/r/lazymio/compilers-env)中编译通过且可正常运行。
- 提交完整的项目结构，无论开发环境是什么
- 提供一个文档
  - 说明如何编译和使用你的编译器（推荐提供一个批处理脚本）。
  - 说明你完成了哪些部分的实验内容。
  - 说明你在实现中对文法/语义规则进行了哪些等价的改写。
  - 说明你对哪些未定义行为进行了定性。
- 如果你对文法/语义规则进行了不等价的改写，那么你应当**专门提供一个文档来说明改写后的内容与改写的理由**。
- 如果没有使用参考的虚拟机，那么项目**必须包含你使用的虚拟机的说明文档**，且保证虚拟机也能在[这个镜像](https://hub.docker.com/r/lazymio/compilers-env)可运行。
    - 如果虚拟机是你自己实现的，还应当提供源代码并保证在[这个镜像](https://hub.docker.com/r/lazymio/compilers-env)可编译并运行。

将所有提交物打包后提交，且：

- 压缩包格式请使用zip
- 压缩包命名统一 ***学号\_姓名\_C0.zip***，比如 *16211044\_李明\_C0.zip*。压缩包根目录应该是项目根目录
- 不要打包任何编译产物和输出产物

最终将压缩包提交到 [软件学院云平台](http://cloud.beihangsoft.cn/)。 

## 附录A EBNF

参见 [miniplc0 指导书](https://github.com/BUAA-SE-Compiling/miniplc0-handbook#%E9%99%84%E5%BD%95a-ebnf)

## 附录B C0文法

以下文法包含了扩展C0，是完整版：

```c
<digit> ::= 
    '0'|<nonzero-digit>
<nonzero-digit> ::= 
    '1'|'2'|'3'|'4'|'5'|'6'|'7'|'8'|'9'
<hexadecimal-digit> ::=
    <digit>|'a'|'b'|'c'|'d'|'e'|'f'|'A'|'B'|'C'|'D'|'E'|'F'

<integer-literal> ::= 
    <decimal-literal>|<hexadecimal-literal>
<decimal-literal> ::= 
    '0'|<nonzero-digit>{<digit>}
<hexadecimal-literal> ::= 
    ('0x'|'0X')<hexadecimal-digit>{<hexadecimal-digit>}


<nondigit> ::=    'a'|'b'|'c'|'d'|'e'|'f'|'g'|'h'|'i'|'j'|'k'|'l'|'m'|'n'|'o'|'p'|'q'|'r'|'s'|'t'|'u'|'v'|'w'|'x'|'y'|'z'|'A'|'B'|'C'|'D'|'E'|'F'|'G'|'H'|'I'|'J'|'K'|'L'|'M'|'N'|'O'|'P'|'Q'|'R'|'S'|'T'|'U'|'V'|'W'|'X'|'Y'|'Z'

<identifier> ::= 
    <nondigit>{<nondigit>|<digit>}
<reserved-word> ::= 
     'const'
    |'void'   |'int'    |'char'   |'double'
    |'struct'
    |'if'     |'else'   
    |'switch' |'case'   |'default'
    |'while'  |'for'    |'do'
    |'return' |'break'  |'continue' 
    |'print'  |'scan'


<char-liter> ::= 
    "'" (<c-char>|<escape-seq>) "'" 
<string-literal> ::= 
    '"' {<s-char>|<escape-seq>} '"'
<escape-seq> ::=  
      '\\' | "\'" | '\"' | '\n' | '\r' | '\t'
    | '\x'<hexadecimal-digit><hexadecimal-digit>

    
<sign> ::= 
    '+'|'-'
<digit-seq> ::=
    <digit>{<digit>}
<floating-literal> ::= 
     [<digit-seq>]'.'<digit-seq>[<exponent>]
    |<digit-seq>'.'[<exponent>]
    |<digit-seq><exponent>
<exponent> ::= 
    ('e'|'E')[<sign>]<digit-seq>
   
    
<unary-operator>          ::= '+' | '-'
<additive-operator>       ::= '+' | '-'
<multiplicative-operator> ::= '*' | '/'
<relational-operator>     ::= '<' | '<=' | '>' | '>=' | '!=' | '=='
<assignment-operator>     ::= '='   

    
<single-line-comment> ::=
    '//'{<any-char>}<LF>
<multi-line-comment> ::= 
    '/*'{<any-char>}'*/'  
    
    
<type-specifier>         ::= <simple-type-specifier>
<simple-type-specifier>  ::= 'void'|'int'|'char'|'double'
<const-qualifier>        ::= 'const'
    
    
<C0-program> ::= 
    {<variable-declaration>}{<function-definition>}


<variable-declaration> ::= 
    [<const-qualifier>]<type-specifier><init-declarator-list>';'
<init-declarator-list> ::= 
    <init-declarator>{','<init-declarator>}
<init-declarator> ::= 
    <identifier>[<initializer>]
<initializer> ::= 
    '='<expression>    

    
<function-definition> ::= 
    <type-specifier><identifier><parameter-clause><compound-statement>

<parameter-clause> ::= 
    '(' [<parameter-declaration-list>] ')'
<parameter-declaration-list> ::= 
    <parameter-declaration>{','<parameter-declaration>}
<parameter-declaration> ::= 
    [<const-qualifier>]<type-specifier><identifier>

    
<compound-statement> ::= 
    '{' {<variable-declaration>} <statement-seq> '}'
<statement-seq> ::= 
	{<statement>}
<statement> ::= 
     <compound-statement>
    |<condition-statement>
    |<loop-statement>
    |<jump-statement>
    |<print-statement>
    |<scan-statement>
    |<assignment-expression>';'
    |<function-call>';'
    |';'   
    
    
<condition> ::= 
     <expression>[<relational-operator><expression>] 
   
<condition-statement> ::= 
     'if' '(' <condition> ')' <statement> ['else' <statement>]
    |'switch' '(' <expression> ')' '{' {<labeled-statement>} '}'

<labeled-statement> ::= 
     'case' (<integer-literal>|<char-literal>) ':' <statement>
    |'default' ':' <statement>

    
<loop-statement> ::= 
    'while' '(' <condition> ')' <statement>
   |'do' <statement> 'while' '(' <condition> ')' ';'
   |'for' '('<for-init-statement> [<condition>]';' [<for-update-expression>]')' <statement>

<for-init-statement> ::= 
    [<assignment-expression>{','<assignment-expression>}]';'
<for-update-expression> ::=
    (<assignment-expression>|<function-call>){','(<assignment-expression>|<function-call>)}


<jump-statement> ::= 
     'break' ';'
    |'continue' ';'
    |<return-statement>
<return-statement> ::= 'return' [<expression>] ';'
    
    
<scan-statement> ::= 
    'scan' '(' <identifier> ')' ';'
<print-statement> ::= 
    'print' '(' [<printable-list>] ')' ';'
<printable-list>  ::= 
    <printable> {',' <printable>}
<printable> ::= 
    <expression> | <string-literal>

<assignment-expression> ::= 
    <identifier><assignment-operator><expression>
    
   
  
<expression> ::= 
    <additive-expression>
<additive-expression> ::= 
     <multiplicative-expression>{<additive-operator><multiplicative-expression>}
<multiplicative-expression> ::= 
     <cast-expression>{<multiplicative-operator><cast-expression>}
<cast-expression> ::=
    {'('<type-specifier>')'}<unary-expression>
<unary-expression> ::=
    [<unary-operator>]<primary-expression>
<primary-expression> ::=  
     '('<expression>')' 
    |<identifier>
    |<integer-literal>
    |<char-literal>
    |<floating-literal>
    |<function-call>

<function-call> ::= 
    <identifier> '(' [<expression-list>] ')'
<expression-list> ::= 
    <expression>{','<expression>}
```
