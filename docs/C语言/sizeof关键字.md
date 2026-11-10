# sizeof关键字

## sizeof 是运算符，不是函数

`sizeof` 在**编译期**求值（VLA除外），**不会真正执行表达式**：

```c
int x = 10;
sizeof(x++);          /* x 仍是 10！x++ 从未执行（编译期被"看穿"） */
sizeof(int)           /* 对类型名必须加括号 */
sizeof x              /* 对表达式括号可省（证明它不是函数） */
```

结果类型是 `size_t`（`%zu` 打印）。

## 基本类型大小表

| 表达式 | 32位 | 64位(LP64) |
| --- | --- | --- |
| `sizeof(char)` | 1 | 1 |
| `sizeof(short)` | 2 | 2 |
| `sizeof(int)` | 4 | 4 |
| `sizeof(long)` | 4 | **8** |
| `sizeof(long long)` | 8 | 8 |
| `sizeof(float)` | 4 | 4 |
| `sizeof(double)` | 8 | 8 |
| `sizeof(void *)` | **4** | **8** |
| `sizeof(char[100])` | 100 | 100 |

## 数组 vs 指针：sizeof 的分水岭

```c
int arr[10];
int *p = arr;

sizeof(arr)      /* 40：整个数组大小（仅在数组定义所在作用域有效） */
sizeof(p)        /* 4/8：指针本身大小 */
sizeof(arr)/sizeof(arr[0])   /* 10：经典数组长度计算公式 */
```

**数组作为函数参数会退化为指针**——sizeof 在函数内失效：

```c
void func(int a[10])
{
    sizeof(a);     /* = 指针大小 4/8！参数 int a[10] 实际是 int *a */
}
```

这就是为什么C函数都要额外传长度参数：

```c
void func(int *a, size_t n);   /* 数组+长度 是C的惯用组合 */
```

## sizeof 与结构体：对齐的放大效应

```c
struct A {
    char  c;    /* 1 字节 */
    int   i;    /* 4 字节 */
    char  d;    /* 1 字节 */
};
/* 直觉 1+4+1=6，实际 sizeof = 12！因为填充(对齐) */
```

详见[内存对齐与位域](内存对齐与位域.md)。

## 经典应用

### 1. 计算数组元素个数

```c
#define ARRAY_SIZE(a) (sizeof(a) / sizeof((a)[0]))

int arr[] = {1, 2, 3, 4, 5};
size_t n = ARRAY_SIZE(arr);      /* 5 */
```

### 2. malloc 安全惯用法

```c
int *p = malloc(sizeof(*p) * n);   /* 用 *p 推导类型，改类型不用改这行 */
```

### 3. 判断平台位数

```c
printf("%d-bit\n", (int)(sizeof(void *) * 8));   /* 4或8 */
```

### 4. 编译期断言（C11前的技巧）

```c
/* 数组大小为负 → 编译错误，实现编译期检查 */
typedef char static_assert_1[(sizeof(int) == 4) ? 1 : -1];
/* C11 起用标准方案 */
_Static_assert(sizeof(int) == 4, "int must be 4 bytes");
```

## sizeof 的返回类型陷阱

```c
int n = 10;
/* 1. 循环变量用size_t方向搞反 → 死循环 */
for (size_t i = n - 1; i >= 0; i--)   /* size_t永>=0，死循环！ */

/* 2. sizeof做减法被提升为无符号 */
if (sizeof(int) - 5 > 0)              /* 恒真！(size_t)4294967295 */
```

**规则**：`sizeof` 参与算术运算的结果是**无符号**的 `size_t`，与有符号数比较时会触发** usual arithmetic conversion**（见[类型转换](类型转换.md)），小心比较方向。

## C99 变长数组（VLA）例外

```c
int n = 5;
int arr[n];           /* VLA：运行期才确定大小 */
sizeof(arr);          /* 必须运行期计算！这是唯一"运行期sizeof" */
```

## 小结

1. sizeof 编译期求值（VLA除外），表达式不执行；
2. 数组传参退化为指针，sizeof 只能测"本作用域的数组"；
3. `sizeof(arr)/sizeof(arr[0])` 计算元素个数；
4. sizeof 结果是 size_t（无符号），注意与负数比较的坑。
