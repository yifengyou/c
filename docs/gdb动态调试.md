# gdb动态调试

> gdb 是 Linux 的动态调试主力——本库所有实测的"运行验证"背后都是它。本页用真实会话演示：断点、栈回溯、寄存器/标志位观察，以及**用"改ZF"当场破解 crackme01**（OllyDbg 动态三式的 Linux 版）。
> 实测环境：Ubuntu 24.04 / gdb 15.1 / x86-64

## 静态 vs 动态（与IDA/OD的分工）

| | 静态（objdump/IDA） | 动态（gdb/OD） |
| --- | --- | --- |
| 程序跑不跑 | 不跑 | 真跑 |
| 看到什么 | 全部代码 | 实际路径+**运行时的值**（strcmp的返回值、栈上的字符串） |
| 本库章节 | [编译链接](C语言/编译链接与库.md)、[IDAPro](IDAPro.md) | 本页、[OllyDbg](OllyDbg.md) |

## 命令速查（与OD按键对照）

| gdb | 作用 | OD对应 |
| --- | --- | --- |
| `b 函数名` / `b *地址` | 下断点 | `F2` / 命令栏bp |
| `run`（r） | 运行 | `F9` |
| `ni` / `si` | 单步（过/进函数） | `F8` / `F7` |
| `finish` | 执行到函数返回 | `Ctrl+F9` |
| `info registers`（i r） | 看寄存器 | 寄存器窗格 |
| `i r eax eflags` | 看指定寄存器 | 同上 |
| `x/s $rdi` | 按字符串看内存 | dump窗格 |
| `x/16bx $rsp` | 按16字节hex看栈 | 栈窗格 |
| `disassemble check` | 反汇编函数 | CPU主窗 |
| `bt` | **栈回溯**（谁调的我） | 调用栈窗 |
| `set $eax = 0` | 改寄存器 | 双击改值 |
| `set $eflags \|= 0x40` | **改ZF标志位** | 改标志位 |
| `set var x=5` | 改C变量（有调试信息时） | —— |
| `c` | 继续 | `F9` |

## 真实会话：动态破解 crackme01

靶子：[crackme01](../crackme/crackme01/crackme01.md)（strcmp 密码校验）。目标：**不改文件**，让错误密码通过——方法：在决胜的 `jne` 处把 **ZF 标志位**掰成1，让跳转不发生。

```text
$ gdb -q ./crackme01.bin
(gdb) break check                    ← ①断在check函数入口
Breakpoint 1 at 0x11d1
(gdb) run                            ← ②运行（输入错误密码 wrongpass）
Breakpoint 1, 0x00005555555551d1 in check ()

(gdb) x/s $rdi                       ← ③第1参数（rdi）就是输入的密码
0x7fffffffdf10: "wrongpass"

(gdb) bt                             ← ④栈回溯：谁调的check？
#0  0x00005555555551d1 in check ()
#1  0x0000555555555263 in main ()    ←   main → check，调用链与源码一致

(gdb) break *(check+0x28)            ← ⑤精确断在jne（0x11f1，见crackme01反汇编）
Breakpoint 2 at 0x5555555551f1
(gdb) c
Breakpoint 2, 0x00005555555551f1 in check ()

(gdb) i r eax eflags                 ← ⑥看strcmp的战果
eax            0x20    32            ← ★strcmp返回差值！'w'(119)-'W'(87)=32
eflags         0x202   [ IF ]        ← ★ZF=0（非零结果）→ jne 将跳向失败

(gdb) set $eflags = $eflags | 0x40   ← ⑦★★动态破解：置ZF（0x40=bit6）
(gdb) c
Password: Congratulations!           ← ★★错误密码通过！
[Inferior 1 (process ...) exited normally]
```

逐条拆解这次会话用到的知识：

| 现场现象 | 背后的知识 |
| --- | --- |
| `x/s $rdi` 直接看到密码 | [参数传递](汇编/参数传递.md)：第1参数在rdi |
| `eax=0x20` | **strcmp返回差值**而非0/1——32正是'w'-'W'（[常量区](C语言/常量区.md)里密码首字符'W'） |
| `eflags=0x202`，只亮IF | [EFLAGS](汇编/EFLAGS寄存器.md)：bit6(ZF)=0 → 非零结果 |
| `\|= 0x40` 后jne不跳 | jne=ZF=0才跳；ZF掰成1 → 落入`mov eax,1`成功分支 |
| 断点自动落在0x11d1（非函数首） | gdb跳过序言（endbr64/push/mov/sub四条，见[堆栈图](汇编/堆栈图.md)） |
| bt 显示main→check | 栈上返回地址串成的链（调用现场存证） |

**三式对照**：静态补丁改字节（[crackme01·75 07→90 90](../crackme/crackme01/crackme01.md)）；动态改标志（本页 `set $eflags |= 0x40`）；动态改寄存器（断在strcmp返回处 `set $eax = 0` 同效）。三式殊途同归——**让ZF=1**。

## 批处理模式（自动化调试）

```bash
gdb -q -batch \
    -ex 'break check' -ex 'run' \
    -ex 'x/s $rdi' \
    -ex 'break *(check+0x28)' -ex 'continue' \
    -ex 'set $eflags = $eflags | 0x40' -ex 'continue' \
    ./crackme01.bin < input.txt        # -ex串命令，适合脚本化验证
```

## 调试带 -g 编译的程序（更甜的姿势）

```bash
gcc -g -O0 crackme02.c -o crackme02_dbg    # 带调试符号
(gdb) b gen            # 直接按源码函数名断点
(gdb) run
(gdb) p name           # 按C变量名查看（不再需要算偏移）
(gdb) p s              # 循环变量s实时值——算法重建的"作弊器"
(gdb) watch s          # 监视点：s每次变化都停——算法骨架自动现形
(gdb) info locals      # 列出全部局部变量
```

**watch 是算法型靶的大杀器**：对[crackme02](../crackme/crackme02/crackme02.md)的`s`下监视，每轮循环的加权累加全部停给你看——keygen逻辑当场显形。

## 常见坑

| 坑 | 原因 | 解法 |
| --- | --- | --- |
| `break strcmp` 断到链接器内部 | ld.so自己也用strcmp | 断业务函数（check），或`b *(check+0x28)`精确断 |
| PIE地址每次不同 | ASLR | 用符号+偏移（`check+0x28`），别写死绝对地址 |
| 改了$eax标志没变 | **标志在test时已定格** | 改`$eflags`（本页做法），或改在test之前 |
| 输入被gdb抢走 | scanf与gdb共用stdin | `run < input.txt` 重定向 |

## 小结

1. gdb三板斧：`b`断点、`x`看内存、`i r`看寄存器——其余都是变体；
2. **动态破解=改运行时状态**：寄存器（`set $eax=0`）、标志位（`set $eflags |= 0x40`）、内存（`set {int}0xaddr=...`）；
3. strcmp返回**差值**（0x20='w'-'W'）——动态现场比静态更容易发现这类细节；
4. `-g`+`watch`+`p 变量名`是算法型靶的keygen加速器；
5. 与OD概念一一对应（F2/F9/F7/F8 ↔ b/run/si/ni）——工具在换，[堆栈图/寻址公式/EFLAGS](汇编.md)永不贬值。
