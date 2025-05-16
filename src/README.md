# SysyCompiler 源代码模块

本文档概述了负责 SysY 编译器不同阶段的关键源文件。

## 编译器阶段及对应文件：

1.  **词法分析 (Lexical Analysis):**

    - `sysy.l`: 此文件包含 Flex 定义，用于将 SysY 源代码词法分析为词法单元（token）流。

2.  **语法分析 (Syntax Analysis) & 抽象语法树构建 (AST Construction):**

    - `sysy.y`: 此文件包含 Bison 语法规则，用于解析词法单元流并定义 SysY 语言的语法结构。这些规则中的语义动作负责构建抽象语法树 (AST)。
    - `AST.h`: 定义了抽象语法树各个节点的 C++ 类。
    - `AST.cpp`: 实现 AST 节点的方法，包括其构造逻辑（通常由 `sysy.y` 调用）和初始处理。

3.  **语义分析 (Semantic Analysis) & 中间代码生成 (Intermediate Code Generation - Koopa IR):**

    - `AST.h` & `AST.cpp`: 语义分析（如类型检查、作用域解析）主要在 AST 遍历过程中处理。`AST.cpp` 中每个 AST 节点的 `Dump()` 方法（或类似名称的函数）负责生成 Koopa IR。
    - `Symbol.h` & `Symbol.cpp`: 定义并实现符号表及相关数据结构（如类型），这些对于语义分析至关重要。

4.  **目标代码生成 (Target Code Generation - RISC-V):**

    - `visit.h` & `visit.cpp`: 这些文件包含遍历 Koopa IR（中间表示）并将其转换为 RISC-V 汇编代码的逻辑。`Visit` 函数处理不同的 Koopa IR 指令和结构。

5.  **中间代码优化 (Intermediate Code Optimization):**
    - 项目根目录下的主 `README.md` 文件指出，为了保持项目可管理性，复杂的优化（如寄存器分配）被简化或未实现（"所有变量局部变量都保存在栈中: 出于简单起见的实现，没有寄存器分配"）。
    - Koopa IR 框架本身（通过 `Makefile` 中的 `-lkoopa` 链接）可能提供或执行某种程度的优化，但此代码库中直接实现的特定优化遍并未明确作为独立模块详细说明。

## 辅助模块：

- `main.cpp`: 编译器的主驱动程序，负责协调不同的编译阶段。
- `utils.h`: 包含在不同模块中使用的工具函数和辅助类（例如，如项目主 `README.md` 中提到的，用于管理 Koopa IR 字符串和 RISC-V 字符串）。

有关更详细的设计和实现细节，请参阅项目根目录下的主 `README.md` 文件。
