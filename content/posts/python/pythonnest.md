---
title: "Python 括号层数限制(SyntaxError: too many nested parentheses)"
summary: " "
date: 2021-01-12T18:07:34+08:00
---

# Python 括号层数限制(SyntaxError: too many nested parentheses)

## 代码

```python
n = 201
s = '(' * n  + ')' * n
print(eval(s))
```

## 错误信息

```bash
Traceback (most recent call last):
  File "/mnt/hgfs/Python-3.9.1/test.py", line 7, in <module>
    print(eval(s))
  File "<string>", line 1
    ((((((((((((((((((((((((((((((((((((((((((((((((((((((((((((((((((((((((((((((((((((((((((((((((((((((((((((((((((((((((((((((((((((((((((((((((((((((((((((((((((((((((((((((((((((((((((((((((((((((((()))))))))))))))))))))))))))))))))))))))))))))))))))))))))))))))))))))))))))))))))))))))))))))))))))))))))))))))))))))))))))))))))))))))))))))))))))))))))))))))))))))))))))))))))))))))))))))))))))))))))
                                                                                                                                                                                                           ^
SyntaxError: too many nested parentheses
```

当 `n <= 200` 时，程序是正常的。

## debug

在 Python 源码目录下，使用 `find` 指令寻找源码中对应的错误信息：

```bash
$ find ./ -name "*.[ch]" | xargs grep "too many nested parentheses"
./Parser/tokenizer.c:            return syntaxerror(tok, "too many nested parentheses");
```

在 `${Python-3.9.1}/Parser/tokenizer.h` 中：

```c
#define MAXLEVEL 200    /* Max parentheses level */
```

在 `${Python-3.9.1}/Parser/tokenizer.c` 中可以看出，这个宏限制的不仅是小括号，中括号和大括号也是一起计算的。

```c
    /* Keep track of parentheses nesting level */
    switch (c) {
    case '(':
    case '[':
    case '{':
        if (tok->level >= MAXLEVEL) {
            return syntaxerror(tok, "too many nested parentheses");
        }
        tok->parenstack[tok->level] = c;
        tok->parenlinenostack[tok->level] = tok->lineno;
        tok->level++;
        break;
    case ')':
    case ']':
    case '}':
        if (!tok->level) {
            return syntaxerror(tok, "unmatched '%c'", c);
        }
        tok->level--;
        int opening = tok->parenstack[tok->level];
	    ...
```

把 `MAXLEVEL` 改为 `300` ，然后重新编译 Python ，程序成功执行：

```bash
()
```

当 `n` 增大到 `301` 时，再次报错 `SyntaxError: too many nested parentheses` 

## 旧版 Python

`MAXLEVEL` 这个宏不是一直就有的，我记得它是在 `3.8` 的某个版本引入的，详细信息参考 [issue33306](https://bugs.python.org/issue33306)

我下载了 Python 3.7.9 ，这个版本在 `n >= 94` 时报错：

```python
s_push: parser stack overflow
Traceback (most recent call last):
  File "test.py", line 6, in <module>
    print(eval(s))
MemoryError
```

寻找错误信息对应的源码：

```bash
$ find ./ -name "*.[ch]" | xargs grep "parser stack overflow"
./Parser/parser.c:        fprintf(stderr, "s_push: parser stack overflow\n");
```

然后在 `./Parser/parser.h` 中找到 DFA 栈的大小：

```c
#define MAXSTACK 1500
```

将这个值改为 3000 然后重新编译，最大括号层数从 93 增长到了 187

## 问题

1. `stackentry       s_base[MAXSTACK];` 里面都存了什么东西，为什么 `MAXSTACK` 的值都上千了，最大括号层数还不是很多？
2. 为什么使用新的实现，按照 issue 33306 的说明，是为了实现更友好的错误提示，那么为什么这么改？新的数据结构把括号分离出去以后怎么做计算？

这些问题只有在仔细研究源码以后才能了解，暂时搁置了，以后有时间再回来填上这个坑。