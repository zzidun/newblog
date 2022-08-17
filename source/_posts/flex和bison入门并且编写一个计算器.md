---
title: flex和bison入门并且编写一个计算器
categories:
  - 经验和总结
tags:
  - bison
  - flex
  - 编译原理
abbrlink: 3f5950fe
date: 2022-03-18 16:57:39
---

<!-- more -->

## flex

flex是一个生成词法分析器的工具.

我们可以编写flex程序,输入到flex.

flex会输出一个c代码,这个c代码能够对我们在flex程序中描述的词法做词法分析.

编译这个c代码,可以得到一个可执行文件,这就是词法分析的程序.

如果我们把文本输入到这个可执行文件,就会得到输入文本的词法分析的结果.

### flex程序

一个flex程序通常包含三部分,形如

```
<声明部分>
%%
<装换规则>
%%
<辅助函数>
```

这三个部分用`%%`分割.

#### 声明部分

声明部分包含了名称声明和选项设置.

其中`%{`和`%}`之间的内容会被原样复制到生成的C代码的开头,可以编写任意合法的C代码.就像这样

```
%{
	/*C代码*/
%}
```

选项设置的语法如下:

```
%option yylineno noyywrap
```

以`%option`开头.

`yylineno`表示flex生成一个变量`yylineno`保存当前行号.

`noyywrap`表示flex不使用`yywrap()`函数.

名称声明的语法如下:

```
<名字> (<正则表达式>)
```

这个就是给正则表达式起一个别名.

关于正则表达式我找到一个[不错的文档](https://support.google.com/analytics/answer/1034324?hl=zh-Hans).

#### 装换规则

每个规则包含模式和动作.

当词法分析器识别出某个模式时,将执行相应的动作.

形如:

```
<模式>
<模式> |
<模式> |
<模式> { <动作> }
```

这里`<模式>`是一个正则表达式,如果这一行后面没跟东西,那么就没有动作.

如果后面是一个`|`,那么表示这个模式和下一行的模式采取相同的动作.

动作本身用`{}`括起来.

在匹配的过程中,会尽量匹配长的字符串

如果可以匹配多个模式,则使用较早出现的模式.

#### 辅助函数

可以包含任意合法的C代码,该部分的内容会被复制到生成的C代码中.

## bison

语法分析器,根据给定的语法规则将flex生成的token转换成抽象语法树(AST).

bison的输入是一个`.y`文件,输出C代码和一个头文件.

如果把bison和flex输出的C代码编到一起,就能得到一个用于将语言转换成语法树的可执行文件.

### bison语法

bison的输入`.y`文件包含以下内容.

#### 定义段

在`%{`和`%}`之间,以C语法写的一些定义和声明,例如文件包含,宏定义,全局变量定义,函数声明.

对语法的终结符和非终结符做一些相关声明,形如

```
%<选项> <值1> <值2> ...
```

越后定义的终结符优先级越高.

| 选项 | 作用 |
| --- | --- |
| `%token` | 定义词法中使用的终结符 |
| `%left` | 左结合 |
| `%right` | 右结合 |
| `%nonassoc` | 不可结合 |
| `%union` | 声明终结符的类型 |
| `%type` | 非终结符的类型 |
| `%start` | 声明起始规则,如果不声明,那就是第一个规则作为起始规则 |

#### 语法规则段

形如这样

```
exp : exp '+' exp { $$ = newast('+', $1, $3)；}
    | exp CMP exp { $$ = newast($2, $1, $3)；}
		| '(' exp ')' { $$ = newast($1)；}
```

这里表示exp的几种形式,如果满足其中一种,那么就执行这种模式同一行的行为.

#### 辅助函数段

可以包含任意合法的C代码,该部分的内容会被复制到生成的C代码中.

## 实现计算器

我觉得,如果使用bison和flex,应该是先写`calc.y`文件,再写`calc.l`文件.

因为`calc.y`文件经过bison处理之后,会产生一个`calc.tab.h`文件

`calc.l`文件需要包含这个`calc.tab.h`文件.

也就是先写语法分析再写词法分析.


### 语法分析

`calc.y`文件需要做以下事情

#### 定义段

在定义段需要


1. 用`%{ <C语言> %}`包含辅助函数所需的头文件.这里我们只需要`stdio.h`和`stdlib.h`

2. 用`%{ <C语言> %}`声明函数`yylex()`和`yyerror()`.(`yylex()`不需要我们自己实现,flex会生成他的C代码,`yyerror()`我们需要在辅助函数段实现).其实不声明也可以,gcc能够自己找到他们的定义,但是会警告.

```
%{
#include <stdio.h>
#include <stdlib.h>

int yylex();
int yyerror(const char *s);
%}
```

3. 用`%union`定义我们要用到的类型,并给他们命名.这个例子里面我们只需要用到`double`,我们给他起名为`double_value`.

```
%union {
  double double_value;
}
```

4. 用`%token`定义终结符,有一些终结符需要定义类型,定义类型使用上面`%union`中命名的类型.这里`DOUBLE_LITERAL`需要定义类型为上面的`double_value`.

```
%token <double_value> DOUBLE_LITERAL
%token ADD SUB MUL DIV CR LP RP
```

这里的`ADD SUB MUL DIV CR LP RP`分别是加号,减号,乘号,除号.换行,左括号,右括号.

他们在词法分析中也有出现.

5. 用`%type`定义非终结符的类型,定义类型使用上面`%union`中命名的类型.

```
%type <double_value> expression term primary_expression
```

这里的`expression`是只含加减操作和`term`的表达式.`term`是只含乘除操作和`primary_expression`的表达式.`primary_expression`是一个数字或带符号和括号的表达式.

显然这些表达式的结果都是`double`.

#### 语法规则段

语法规则段则是定义各个符号由其他什么符号组成,匹配到之后采取什么行动.

默认第一个规则是起始规则,也就是输入的内容的整体,会被看作是第一个符号.

输入通常有多行,所以第一个规则定义一个`line_lists`,他可能是一个`line`,也可能是多行(也就是`line_lists`本身加上一个`line`)

```
line_list
    : line
    | line_list line
    ;
```

这个计算器中,一行放一个表达式,所以`line`由一个`expression`和一个换行组成.一旦匹配到`line`,那就输出`line`的值.

```
line
    : expression CR
    {
        printf(">>%lf\n", $1);
    }
```

其他的表达式定义如下,这里的`$$`代表这个符号本身的值,`$<数值>`代表组成这个符号的各个部分的值.

由`$<数值>`可以得到`$$`,你可以类比通过参数算出返回值.

```
expression
    : term
    | expression ADD term
    {
        $$ = $1 + $3;
    }
    | expression SUB term
    {
        $$ = $1 - $3;
    }
    ;
term
    : primary_expression
    | term MUL primary_expression
    {
        $$ = $1 * $3;
    }
    | term DIV primary_expression
    {
        $$ = $1 / $3;
    }
    ;
primary_expression
    : DOUBLE_LITERAL
    | SUB primary_expression
    {
        $$ = -$2;
    }
    | LP expression RP
    {
        $$ = $2;
    }
   ;
```

#### 辅助函数段

这里实现`yyerror()`,用来打印一些信息.

```c
int yyerror(char const *str)
{
    extern char *yytext;
    fprintf(stderr, "parser error near %s\n", yytext);
    return 0;
}
```

这个`yytext`在词法分析产生的代码中定义,应该就是当前扫到的文本.

生成的代码中没有`main()`,我们这里自己定义一个`main()`,以便后面可以编译成一个可执行文件.


```c
int main()
{
    extern int yyparse(void);
    extern FILE *yyin;
    yyin = stdin;
    if (yyparse())
    {
        fprintf(stderr, "Error ! Error! Error!\n");
        exit(1);
    }
    return 0;
}
```

`yyparse`定义在bison生成的C代码中,他就是这一套操作下来生成的最关键的函数,负责语法分析的主要功能.

执行之后程序就会开始获取输入,直到遇到输入错误.

遇到错误会返回非`0`值.

#### 代码本体

我们将它命名为`mycalc.y`

```
%{
#include <stdio.h>
#include <stdlib.h>

int yylex();
int yyerror(const char *s);
%}

%union {
  double double_value;
}

%token <double_value> DOUBLE_LITERAL
%token ADD SUB MUL DIV CR LP RP
%type <double_value> expression term primary_expression

%%

line_list
    : line
    | line_list line
    ;
line
    : expression CR
    {
        printf(">>%lf\n", $1);
    }
expression
    : term
    | expression ADD term
    {
        $$ = $1 + $3;
    }
    | expression SUB term
    {
        $$ = $1 - $3;
    }
    ;
term
    : primary_expression
    | term MUL primary_expression
    {
        $$ = $1 * $3;
    }
    | term DIV primary_expression
    {
        $$ = $1 / $3;
    }
    ;
primary_expression
    : DOUBLE_LITERAL
    | SUB primary_expression
    {
        $$ = -$2;
    }
    | LP expression RP
    {
        $$ = $2;
    }
   ;

%%

int yyerror(char const *str)
{
    extern char *yytext;
    fprintf(stderr, "parser error near %s\n", yytext);
    return 0;
}

int main()
{
    extern int yyparse(void);
    extern FILE *yyin;
    yyin = stdin;
    if (yyparse())
    {
        fprintf(stderr, "Error ! Error! Error!\n");
        exit(1);
    }
    return 0;
}
```

### 词法分析

#### 声明部分

1. 用`%{ <C> %}`包含后面用到的头文件,这里需要`stdio.h`和上面bison生成的`mycalc.tab.h`.

这里的`mycalc.tab.h`定义了各种符号的枚举值,词法分析的作用就是根据读到的内容,返回一个在`mycalc.tab.h`中定义的枚举值.

执行`bison -d mycalc.y`可以产生这个头文件和C文件.

2. 定义一个函数`yywrap()`.

`yywrap()`函数的作用是将多个输入文件打包成一个输入，当`yylex()`函数读入到一个文件结束时，它会向`yywrap`函数询问，`yywrap`函数返回`1`的意思是告诉`yylex`函数后面没有其他输入文件了，此时`yylex`函数结束, `yywrap`函数也可以打开下一个输入文件，再向`yylex`函数返回`0`，告诉它后面还有别的输入文件，此时`yylex`函数会继续解析下一个输入文件。总之，由于我们不考虑连续解析多个文件，因此此处返回`1`。

```
%{
#include <stdio.h>
#include "mycalc.tab.h"

int yywrap(void) {return 1;}
%}
```

#### 转换规则

这里主要是各种正则表达式,以及匹配到之后的操作.

这里匹配到运算符就直接返回一个枚举.

如果匹配到一个数值,就赋值给`yylval`.然后返回枚举`DOUBLE_LITERAL`.

`yylval`其实就是上面定义的联合体,里面有`double_value`.

遇到空白符号无操作.

其他所有字符(正则表达式`.`),就报错并且`exit(1)`.

```
"+"    { return ADD;	}
"-"    { return SUB;	}
"*"    { return MUL;	}
"/"    { return DIV;	}
"("    { return LP; 	}
")"    { return RP;		}
"\n"   { return CR;		}
[0-9]*(\.[0-9]+)? {
    double temp;
    sscanf(yytext, "%lf", &temp);
    yylval.double_value = temp;
    return DOUBLE_LITERAL;
}
[ \t] ;
. {
    fprintf(stderr, "lexical error.\n");
    exit(1);
}
```

#### 辅助函数

不需要

#### 代码本体

把这个文件命名为`mycalc.l`

```
%{
#include <stdio.h>
#include "mycalc.tab.h"

int yywrap(void) {return 1;}
%}

%%

"+"    { return ADD;	}
"-"    { return SUB;	}
"*"    { return MUL;	}
"/"    { return DIV;	}
"("    { return LP; 	}
")"    { return RP;		}
"\n"   { return CR;		}
[0-9]*(\.[0-9]+)? {
    double temp;
    sscanf(yytext, "%lf", &temp);
    yylval.double_value = temp;
    return DOUBLE_LITERAL;
}
[ \t] ;
. {
    fprintf(stderr, "lexical error.\n");
    exit(1);
}

%%
```

### 编译

执行这个命令生成语法分析代码

```
bison -dv mycalc.y
```

`-d`表示生成头文件,`-v`表示同时生成一个`.output`文件,这里面有一些详情.

执行这个命令生成词法分析代码

```
flex mycalc.l
```

这时候会得到`lex.yy.c`(词法分析), `mycalc.tab.c`(语法分析), `mycalc.tab.h`(头文件).

用以下命令,把他们编译可执行文件`mycalc`.

```
cc -o mycalc mycalc.tab.c lex.yy.c
```

然后可以手动测试一下这个可执行文件.