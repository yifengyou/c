# crackme01：密码校验型

> 入门靶：`strcmp` 比较明文密码。目标：不重编译，让任意密码通过。
> 实测环境：Ubuntu 24.04 / gcc 13.3.0 / x86-64 / `-O0` 编译。

## 源码与行为

```c
#include <stdio.h>
#include <string.h>

int check(const char *pwd)
{
    if (strcmp(pwd, "WaterDrop2026") == 0)
        return 1;
    return 0;
}

int main(void)
{
    char buf[32];
    printf("Password: ");
    if (scanf("%31s", buf) != 1)
        return 1;
    if (check(buf))
        printf("Congratulations!\n");
    else
        printf("Wrong password!\n");
    return 0;
}
```

```text
$ echo "hackyou" | ./crackme01.bin       ← 错误密码
Password: Wrong password!
$ echo "WaterDrop2026" | ./crackme01.bin  ← 正确密码
Password: Congratulations!
```

## 第一步：strings 找锚点

```text
$ strings crackme01.bin | grep -i water
WaterDrop2026            ← 密码明文躺在 .rodata！第一分钟就缴械
```

**现实**：大量真实软件的"机密"就是 rodata 明文——`strings` 永远是逆向第一命令（见[常量区](../../docs/C语言/常量区.md)）。

## 第二步：反汇编定位关键 jcc

```text
$ objdump -d -M intel crackme01.bin
00000000000011c0 <check>:
    11d9: 48 8b 45 f8        mov    rax,QWORD PTR [rbp-0x8]      ; 取pwd
    11dd: 48 8d 15 20 0e..   lea    rdx,[rip+0xe20]  # 2004      ; ←密码"WaterDrop2026"地址
    11e4: 48 89 d6           mov    rsi,rdx                       ; 第2参数=密码
    11e7: 48 89 c7           mov    rdi,rax                       ; 第1参数=输入
    11ea: e8 d1 fe ff ff     call   10c0 <strcmp@plt>             ; ★比较
    11ef: 85 c0              test   eax,eax                       ; strcmp相等返回0→ZF=1
    11f1: 75 07              jne    11fa <check+0x31>             ; ★★决胜跳转！不等→失败分支
    11f3: b8 01 00 00 00     mov    eax,0x1                       ; 成功：return 1
    11f8: eb 05              jmp    11ff
    11fa: b8 00 00 00 00     mov    eax,0x0                       ; 失败：return 0
    11ff: c9                 leave
```

对照[函数指针/调用约定](../../docs/汇编/参数传递.md)：strcmp 双参数走 rdi/rsi；`test eax,eax` 读 ZF（[EFLAGS](../../docs/汇编/EFLAGS寄存器.md)）；`75 07` = **jne 短跳**（[跳转指令集](../../docs/汇编/跳转指令集.md)：75=jne、74=je、EB=jmp、90=nop）。

main 里还有第二处 jcc（`call check` 后 `test eax,eax; je .L7`）——**同一道门两把锁**，任改一处即可。

## 第三步：补丁（三选一）

| 方案 | 改法 | 效果 |
| --- | --- | --- |
| **A. NOP 掉 jne** | `75 07` → `90 90` | 永远走"成功"分支：**任意密码通过** |
| B. 反转条件 | `75`（jne）→ `74`（je） | 布尔反转：错密码通过、对密码报错（恶作剧版） |
| C. 爆破正确值 | 直接用 `WaterDrop2026` | 不算破解——本靶密码是明文 |

**执行方案A（真实补丁）**：

```bash
$ cp crackme01.bin crackme01_patched.bin
$ printf '\x90\x90' | dd of=crackme01_patched.bin bs=1 seek=$((0x11f1)) conv=notrunc
                        # 把文件偏移0x11f1处的两个字节(75 07)改成两个nop(90 90)
$ echo "iamhacker" | ./crackme01_patched.bin
Password: Congratulations!        ← ★任意密码通过！
```

补丁后反汇编验收：

```asm
    11ef: 85 c0     test   eax,eax
    11f1: 90        nop              ← 原jne被抹掉
    11f2: 90        nop
    11f3: b8 01..   mov    eax,0x1   ; 无论比较结果，直落成功分支
```

## 为什么 vaddr == 文件偏移

本例直接拿 `0x11f1` 当文件偏移成功——因为 PIE 可执行文件的第一个 LOAD 段把**文件偏移0映射到vaddr 0**，`.text` 落在文件开头区域，两者恰好相等。保险做法：`readelf -l` 查段表，按 `文件偏移 = vaddr - 段vaddr + 段offset` 换算（见[ELF结构](../../docs/ELF解析器/ELF结构.md)）。

## 用gdb动态视角（可选加餐）

```text
(gdb) break check
(gdb) run
(gdb) x/s $rdi          ← 输入的密码
(gdb) x/s $rsi          ← rodata里的 "WaterDrop2026"
(gdb) ni ...            单步到 test 后：
(gdb) p $eflags         ← 观察ZF：密码对=1错=0
```

## 本靶学到什么

1. `strings` 是第一命令——明文密码无处遁形；
2. **决胜点 = 库函数调用后的第一条 jcc**（strcmp 后的 `test+jne` 是个固定套路）；
3. 补丁=改字节：`75→90 90`（NOP门）、`75→74`（反门）、短跳两字节等长替换不破坏后续指令布局；
4. patch/jcc/EFLAGS/PLT 全是前面章节的知识点——**crackme就是它们的期末考试**。

## 下一关

[crackme02：算法型](../crackme02/crackme02.md)——密码藏进算法里，NOP大法失效，必须重建算法写注册机。
