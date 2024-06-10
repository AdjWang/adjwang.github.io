---
title: "藏在栈里的金丝雀"
summary: "缓冲区溢出检测"
date: 2021-04-25T22:29:50+08:00
---

# 藏在栈里的金丝雀

## canary

首先介绍 "canary" 的概念，根据某简称为 CSAPP 的“史籍”记载，金丝雀可以察觉煤矿中的有毒气体，而后来某天编译器加入了一种缓冲区检测的机制，就命名为 "canary"。简单来说，就是在栈帧里加个随机值，函数返回前检查下这个值有没被改动，就知道栈还正不正常了。

## 事情的起源

最近做补丁用到 `__declspec(naked)` 声明的函数，我还想在里面用局部变量。因为 `__declspec(naked)` 声明的函数去掉了头尾的入栈出栈代码，所以我还专门手写了处理堆栈的代码。为了方便说明，这里举个简单的例子：

```cpp
__declspec(naked) int __cdecl canariesTestNaked() {
    struct {
        int a;
        int b;
    } local_var;
    __asm {
        push ebp
        mov ebp, esp
        sub esp, 8
    }
    local_var.a = 5;
    local_var.b = 6;
    local_var.b += local_var.a;
    __asm {
        mov eax, local_var.b
        mov esp, ebp
        pop ebp
        retn
    }
}
```

像这种函数，局部变量的大小应该非常明确，取用局部变量就是 `[ebp-4]` 和 `[ebp-8]`，按照非数组元素倒序排列的规则，它们分别取得 `local_var.b` 和 `local_var.a`，可是奇妙的事情发生了，它编译出来的汇编代码是这样的：

```x86asm
 push        ebp  
 mov         ebp,esp  
 sub         esp,8  
 mov         dword ptr [ebp-0Ch],5  
 mov         dword ptr [ebp-8],6  
 mov         eax,dword ptr [ebp-8]  
 add         eax,dword ptr [ebp-0Ch]  
 mov         dword ptr [ebp-8],eax  
 mov         eax,dword ptr [ebp-8]  
 mov         esp,ebp  
 pop         ebp  
 ret  
```

可以看到， `local_var.a` 对应 `[ebp-C]`，`local_var.b` 对应 `[ebp-8]`，那么，`[ebp-4]` 现在是啥？

没错，就是 "canary" 。其实确切地说，控制编译器做这种事情的是 `/RTC` (Run-time error checks) 选项，而通常大家所说的 "canary" 是 /GS (Buffer Security Check) 选项添加的。对于这件事，**我觉得应该把 "canary" 看成一种安全防范的思想，不要纠结教条的东西，所以我就在这里认为这东西是 "canary" 了。**

当然，还有一些事得说清楚，**添加 "canary" 并不是 RTC 检查的唯一动作，更多细节可以参考** [**MSDN文档**](https://docs.microsoft.com/en-us/cpp/build/reference/rtc-run-time-error-checks?view=msvc-160) 。另外，RTC 检查是 Debug 模式的默认选项，而在 Release 模式下关闭，而 GS 在两种模式下都会默认开启，这也就解释了某些代码注入程序只能在 Release 模式下跑通的现象。然而，凡事都有例外，我也发现会有一些代码，尽管我没有声明数组，在 Release 模式下仍然会受到类似 RTC 这种保护的影响，这时就要考虑去关闭 GS 选项试试。当然，trade-off 就是引入了更大的安全风险。

## 验证

首先，在 `__declspec(naked)` 声明下，"canary" 完全没用，手动修改 `[ebp-4]` 的值，然后运行：

```cpp
__declspec(naked) int __cdecl canariesTestNaked() {
    struct {
        int a;
        int b;
    } local_var;
    __asm {
        push ebp
        mov ebp, esp
        sub esp, 0Ch
        mov dword ptr [ebp-4], 12345678h	// modify canary manually
    }
    local_var.a = 5;
    local_var.b = 6;
    local_var.b += local_var.a;
    __asm {
        mov eax, local_var.b
        mov esp, ebp
        pop ebp
        retn
    }
}
```

没有任何错误信息，程序顺利跑通。然后，去除 `__declspec(naked)` 选项，因为编译器会自动处理栈，我就把手动入栈出栈的代码去掉了，此时汇编代码可以说是面目全非，所以我就截取其中的一段。

```cpp
int __cdecl canariesTest() {
    struct {
        int a;
        int b;
    } local_var;
    __asm mov dword ptr [ebp-4], 12345678h
    local_var.a = 5;
    local_var.b = 6;
    local_var.b += local_var.a;
    return local_var.b;
}

...
 mov         dword ptr [ebp-4],12345678h  
 mov         dword ptr [ebp-0Ch],5  
 mov         dword ptr [ebp-8],6  
 mov         eax,dword ptr [ebp-8]  
 add         eax,dword ptr [ebp-0Ch]  
 mov         dword ptr [ebp-8],eax  
 mov         eax,dword ptr [ebp-8]  
...
```

然后，程序在返回时报错: Run-Time Check Failure #2 - Stack around the variable 'local_var' was corrupted.

## GS 选项的保护

说完了 RTC 中的栈保护，再来看下 GS 的栈保护。

```cpp
void canariesTest() {
    char str[3];
    std::cin >> str;
    std::cout << str << std::endl;
}
```

使用上面这段代码，输入字符数量超过缓冲区长度时，就会发生缓冲区溢出现象。因为输入信息是动态获取的，所以 IDE 的静态分析不会给出提示。

* 输入字符串长度大于等于 4 时，程序报错: Unhandled exception at 0x005F1DBB in test.exe: Stack cookie instrumentation code detected a stack-based buffer overrun.
* 手动关闭 GS 检查，程序报错: Exception thrown at 0x66636164 in test.exe: 0xC0000005: Access violation executing location 0x66636164.

由此可见，Stack cookie instrumentation code 其实就是 GS 选项添加的保护机制，这种防护手段确实起了作用。

此外，输入 3 个字符并没有报错，虽然已经溢出了 1 个字符。

# 在 naked 函数中使用 "局部变量" 的最佳实践

我给局部变量加了引号，是因为这些变量实际上要放到全局区——没错，就是给原来所有的局部变量添加 `static` 关键字，既不破坏命名空间，又能满足局部变量的功能需求。当然，这么做的副作用就是线程不安全，另外，会多占用一点内存空间。代码简化成下面这样：

```cpp
__declspec(naked) int __cdecl canariesTestNaked() {
    static struct {
        int a;
        int b;
    } local_var;
    local_var.a = 5;
    local_var.b = 6;
    local_var.b += local_var.a;
    __asm {
        mov eax, local_var.b
        retn
    }
}
```

# 参考资料

* https://www.informit.com/articles/article.aspx?p=2036582&seqNum=6 （好文请务必多看！）
* https://docs.microsoft.com/en-us/cpp/build/reference/rtc-run-time-error-checks?view=msvc-160
* https://docs.microsoft.com/en-us/cpp/build/reference/gs-buffer-security-check?view=msvc-160