# PE解析器

> PE（Portable Executable）是 Windows 的可执行格式（.exe/.dll/.sys）——DTDebug/OllyDbg/IDA 处理的主战场，本库 crackme 教程的原始语境。详细结构见 [PE结构](PE解析器/PE结构.md)。

## 为什么Linux党也要懂PE

* 逆向技能两栖：ELF 与 PE 是同一套思想的两套词汇，对照着学一遍 = 双倍巩固；
* 大量经典 crackme、病毒分析、CTF 题目是 PE 格式；
* `reference/PE结构.pdf` 是配套参考资料（本库自带的经典图表）。

## 工具速查（Windows/跨平台）

| 工具 | 用途 |
| --- | --- |
| LordPE（`reference/LordPE.exe`） | PE头编辑器：改入口/节属性/转储 |
| OD/DTDebug | 加载PE动态调试 |
| IDA | 静态反汇编（自动解析PE头） |
| `objdump -f`（Linux下） | 识别PE文件基本信息 |

## 与ELF的关系一句话

> 同样是"文件头 + 内存映射表 + 节数据 + 导入导出表"，PE 多了一个 DOS 头（历史包袱），把 ELF 的 Program Header 叫做 Section Header（术语互换陷阱）。

## 章节导航

* [PE结构](PE解析器/PE结构.md) —— DOS头/NT头/节表/导入表逐层拆解
