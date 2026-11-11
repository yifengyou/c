# ELF解析器

> ELF（Executable and Linkable Format）是 Linux 世界的可执行文件格式——gcc 产出的每一个 `.o`、`.so`、`a.out` 都是 ELF。看懂它 = 看懂"程序从文件到进程"的全过程。

## 为什么必须学

* **逆向第一步**：拿到二进制先 `readelf`/`file` 看类型、位数、是否 PIE、动态依赖（`ldd`）；
* **补丁计算**：改字节要算"虚拟地址 ↔ 文件偏移"的换算（[crackme01](../crackme/crackme01/crackme01.md)用到了）；
* **链接的载体**：[编译链接与库](C语言/编译链接与库.md)里讲的符号、重定位，全部记录在 ELF 的表里；
* **内存图的源头**：[内存图](C语言/内存图.md)里 text/rodata/data/bss 的分区，就是 LOAD 段映射出来的。

## 工具速查

| 命令 | 看什么 |
| --- | --- |
| `file prog` | 位数/类型/是否静态 |
| `readelf -h prog` | ELF头（魔数/类型/入口/位数） |
| `readelf -l prog` | **程序头表**：LOAD段 → 运行时内存布局 |
| `readelf -S prog` | **节表**：.text/.rodata/.data/.bss 文件内布局 |
| `readelf -d prog` | 动态段：依赖的.so、RPATH |
| `readelf -r prog` | 重定位表（对应 `objdump -r`） |
| `readelf -s prog` | 符号表（对应 `nm`） |
| `ldd prog` | 运行时依赖的动态库清单 |

## 章节导航

* [ELF结构](ELF解析器/ELF结构.md) —— 文件头/程序头/节表逐字段实测解析

## 与PE的对照

Windows 的对应格式是 PE（见[PE解析器](PE解析器.md)）——逆向两栖必备，两套对照见 PE 章。
