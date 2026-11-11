# crackme02：算法型

> 进阶靶：没有明文密码——`serial = f(name)`。NOP 补丁能让它"通过"，但**写出注册机（keygen）**才是正道。
> 实测环境：Ubuntu 24.04 / gcc 13.3.0 / x86-64 / `-O0` 编译。

## 源码与行为

```c
#include <stdio.h>

unsigned gen(const char *name)
{
    unsigned s = 0;
    for (int i = 0; name[i]; i++)
        s += (unsigned char)name[i] * (i + 1);   /* 加权求和 */
    return (s ^ 0xDEADBEEF) + 0x1337;
}

int main(void)
{
    char name[32];
    unsigned serial;
    printf("Name: ");
    if (scanf("%31s", name) != 1)  return 1;
    printf("Serial: ");
    if (scanf("%u", &serial) != 1) return 1;
    if (gen(name) == serial)
        printf("Registered!\n");
    else
        printf("Bad serial!\n");
    return 0;
}
```

```text
$ printf "Alice\n1234\n" | ./crackme02.bin
Name: Serial: Bad serial!      ← 1234当然不对
```

`strings` 这回什么都捞不到——秘密在**算法**里，必须从汇编重建。

## 静态分析：gen() 反汇编逐行注解

```text
$ objdump -d -M intel crackme02.bin   （提取 gen，同 gcc -S 产物）
gen:
    push    rbp
    mov     rbp, rsp
    mov     QWORD PTR -24[rbp], rdi   ; name指针
    mov     DWORD PTR -8[rbp], 0      ; s = 0
    mov     DWORD PTR -4[rbp], 0      ; i = 0
    jmp     .L2                       ; 先检查 name[i] != 0
.L3:                                  ; ── 循环体 ──
    mov     eax, DWORD PTR -4[rbp]    ; 取 i
    movsx   rdx, eax
    mov     rax, QWORD PTR -24[rbp]   ; name基址
    add     rax, rdx                  ; name + i
    movzx   eax, BYTE PTR [rax]       ; 读 name[i]（1字节、零扩展→无符号）
    movzx   eax, al
    mov     edx, DWORD PTR -4[rbp]    ; 取 i
    add     edx, 1                    ; ★i + 1（权重！）
    imul    eax, edx                  ; ★name[i] * (i+1)
    add     DWORD PTR -8[rbp], eax    ; s += …
    add     DWORD PTR -4[rbp], 1      ; i++
.L2:
    mov     eax, DWORD PTR -4[rbp]    ; name[i]（循环条件）
    movsx   rdx, eax
    mov     rax, QWORD PTR -24[rbp]
    add     rax, rdx
    movzx   eax, BYTE PTR [rax]
    test    al, al                    ; 字符非零？
    jne     .L3                       ; 继续循环
    mov     eax, DWORD PTR -8[rbp]    ; 取 s
    xor     eax, -559038737           ; ★★异或魔数！-559038737 = 0xDEADBEEF
    add     eax, 4919                 ; ★+4919 = +0x1337
    pop     rbp
    ret                                ; 返回值=注册码
```

## 算法重建：从指令反推公式

逐条指令翻译回C（用到的知识全部来自前章）：

| 汇编 | 解读 | 依据 |
| --- | --- | --- |
| `movzx eax, BYTE PTR [rax]` | 无符号字节 → `(unsigned char)name[i]` | [数据宽度·movzx](../../docs/汇编/数据宽度.md) |
| `add edx,1; imul eax,edx` | `× (i+1)` | [运算符·imul](../../docs/C语言/运算符.md) |
| 循环尾 `test al,al; jne` | `while (name[i])` 走查 | [字符串·实战](../../docs/C语言/字符数组与字符串.md) |
| `xor eax, -559038737` | `s ^= 0xDEADBEEF` | [进制](../../docs/汇编/进制.md)：0xDEADBEEF = 3735927055，补码显示为 -559038737 |
| `add eax, 4919` | `+ 0x1337` | 同上：0x1337 = 4919 |

**重建的算法**：

```c
serial = 0;
for (i = 0; name[i]; i++)
    serial += (unsigned char)name[i] * (i + 1);
serial = (serial ^ 0xDEADBEEF) + 0x1337;
```

> 常见坑：立即数以**有符号十进制**显示（-559038737），一眼认不出 0xDEADBEEF——进制转换（或心算补码）是逆向的日常。算术回绕（unsigned 溢出）按模 2³² 处理，C注册机用 `unsigned` 类型自动对齐。

## 写注册机（keygen）

```c
/* keygen.c —— 与 gen 逻辑一致 */
#include <stdio.h>
unsigned gen(const char *name)
{
    unsigned s = 0;
    for (int i = 0; name[i]; i++)
        s += (unsigned char)name[i] * (i + 1);
    return (s ^ 0xDEADBEEF) + 0x1337;
}
int main(int argc, char **argv)
{
    if (argc < 2) return 1;
    printf("%u\n", gen(argv[1]));
    return 0;
}
```

**真实验证**：

```text
$ gcc -O2 keygen.c -o keygen
$ ./keygen Alice
3735932525                        ← Alice 的注册码

$ printf "Alice\n3735932525\n" | ./crackme02.bin
Name: Serial: Registered!         ← ★注册成功！

$ printf "Bob\n3735932525\n" | ./crackme02.bin
Name: Serial: Bad serial!         ← 换名字同码失败：serial确实绑定name
```

**keygen与patch的尊严之差**：NOP掉比较 jcc 也能"Registered"，但那台机器只认补丁版；注册机对**原版二进制**给出任意名字的合法码——这是"理解算法"与"暴力绕过"的分水岭。

## 另一条路：补丁位在哪

若只想爆破：定位 main 里 `call gen` 之后的比较——`cmp DWORD PTR [rbp-XX], eax; jne .bad`（x86-64实测形态，gen返回值在eax、输入serial在栈槽）。同样 `75 xx → 90 90`。但见上：**能写keygen别用补丁**。

## 防御视角：如何对抗keygen

作为开发者，看完攻击要会防守：

| 弱点 | 本靶表现 | 加固方向 |
| --- | --- | --- |
| 算法在本地 | gen整体可见 | 校验放服务端；本地只发挑战-应答 |
| 魔数明文 | 0xDEADBEEF/0x1337 直接读出 | 多轮白盒化/常量混淆（增加重建成本，不根治） |
| 无符号回绕可复算 | unsigned运算全程确定 | 引入服务器时间戳/硬件指纹作输入 |
| 比较点单一 | 一条jcc定生死 | 多点校验+结果参与后续解密（改一处即崩） |

## 本靶学到什么

1. 算法型靶的灵魂是**循环体里的 imul/加法序列 + 出口处的 xor/加常量**——魔数是算法的指纹；
2. 立即数的有符号十进制脸（-559038737）要一眼认出十六进制真身（0xDEADBEEF）；
3. 注册机=汇编→C的反向工程，`unsigned`类型自动对齐模2³²回绕；
4. patch是暴力、keygen是理解——防御者该知道攻击者怎么看你的代码。
