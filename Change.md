# 在 SysyCompiler 中添加 float 类型支持

## 1. 概述

本文档旨在指导开发者如何在现有的 `SysyCompiler` 项目中逐步添加对 `float` 数据类型的完整支持。这将包括词法分析、语法分析、抽象语法树（AST）修改、语义分析与中间代码（Koopa IR）生成，以及最终的目标代码（RISC-V）生成。

**最终目标符合的 SysY 语言特性：**

- 支持 `float` 关键字和类型。
- `float` 类型为 32 位单精度浮点数。
- 支持浮点数字面量 (例如 `3.14`, `1.0e-5`)。
- 支持元素为 `float` 类型的多维数组。
- 支持 `int` 和 `float` 之间的隐式类型转换 (通常 `int` 到 `float`)。
- 支持涉及 `float` 类型的算术运算、比较运算等。
- 函数可以接受 `float`类型的参数并返回 `float` 类型。

## 2. Step-by-Step 实现指南

### 阶段一：词法分析 (Lexical Analysis)

- **任务：**

  1. 让词法分析器能够识别 `float` 关键字。
  2. (推荐) 让词法分析器能够识别浮点数字面量 (例如 `3.14`, `1e-5`, `0.5f` - 注意 SysY 是否支持 `f` 后缀)。

- **需修改文件：**

  - `SysyCompiler/src/sysy.l`

- **具体修改：**

  1. 在 `sysy.l` 中，为 `float` 关键字添加一条规则，使其返回一个新的 token，例如 `FLOAT`。

     ```lex
     /* ... other keyword rules ... */
     "float"         { return FLOAT; }
     ```

  2. 为浮点数字面量添加规则，使其返回一个新的 token，例如 `FLOAT_CONST`，并将其值存储在 `yylval` (可能需要扩展 `yyunion`)。

     ```lex
     /* ... Integer literal patterns ... */
     FloatSimple     [0-9]+\.[0-9]*
     FloatExp        [0-9]+([eE][-+]?[0-9]+)
     FloatSimpleExp  [0-9]+\.[0-9]*([eE][-+]?[0-9]+)?
     FloatSuffix     (f|F) /* Optional: if SysY supports 'f' suffix */

     /* More robust pattern combining these, e.g.: */
     FloatLiteral    ({FloatSimple}|{FloatExp}|{FloatSimpleExp}){FloatSuffix}?

     %%
     /* ... other rules ... */
     {FloatLiteral}  { /* yylval.float_val = atof(yytext); */ return FLOAT_CONST; }
     ```

- **验收标准：**

  - 编写包含 `float` 关键字的 SysY 代码片段 (例如 `float a;`)，编译器在词法分析阶段不报错，并能正确识别 `FLOAT` token。
  - 编写包含浮点数字面量的代码片段 (例如 `float b = 3.14;`), 编译器在词法分析阶段不报错，并能正确识别 `FLOAT_CONST` token 及其值。

### 阶段二：语法分析 (Syntax Analysis)

- **任务：**
  1. 在 Bison 文件中声明新的 `FLOAT` 和 `FLOAT_CONST` token。
  2. 修改 `BType` (基本类型) 的语法规则以包含 `FLOAT`。
  3. (如果支持浮点数字面量) 修改处理数字字面量的语法规则 (例如 `PrimaryExp` 或 `Number`) 以包含 `FLOAT_CONST`。
- **需修改文件：**
  - `SysyCompiler/src/sysy.y`
- **具体修改：**
  1. 在 `%token` 声明区域添加 `FLOAT` 和 `FLOAT_CONST`。
     ```yacc
     %token VOID INT RETURN ... CONST ... FLOAT /* New */
     %token <str_val> IDENT
     %token <int_val> INT_CONST
     %token <float_val> FLOAT_CONST /* New, assuming float_val in %union */
     ```
  2. 如果为 `FLOAT_CONST` 返回浮点值，需要在 `%union` 中添加一个 `float_val` (或类似) 成员。
     ```yacc
     %union {
       std::string *str_val;
       int int_val;
       float float_val; /* New */
       char char_val;
       BaseAST *ast_val;
     }
     ```
  3. 修改 `BType` 规则：
     ```yacc
     BType
       : INT {
           auto btype = new BTypeAST();
           btype->tag = BTypeAST::INT_TYPE; /* Or your existing enum */
           $$ = btype;
         }
       | FLOAT { /* New rule */
           auto btype = new BTypeAST();
           btype->tag = BTypeAST::FLOAT_TYPE; /* New enum value in BTypeAST */
           $$ = btype;
         }
       ;
     ```
  4. 修改 `PrimaryExp` (或等价规则) 以处理 `FLOAT_CONST`：
     ```yacc
     PrimaryExp
       /* ... other rules for (Exp), LVal, INT_CONST ... */
       | FLOAT_CONST {
           /* Create a NumberAST or similar AST node for float literal */
           /* $$ = new NumberAST(yylval.float_val); */
         }
       ;
     ```
- **验收标准：**
  - 包含 `float` 类型声明 (如 `float x;`, `const float PI = 3.14;`) 的 SysY 代码能够被正确解析而不产生语法错误。
  - 涉及浮点数字面量的表达式能够被正确解析。

### 阶段三：抽象语法树 (AST)

- **任务：**
  1. 更新 `BTypeAST` (或等价的类型节点) 以能够表示 `float` 类型。
  2. 如果支持浮点数字面量，确保 `NumberAST` (或等价节点) 能够存储和表示浮点值。
  3. 确保表达式相关的 AST 节点 (如 `AddExpAST`, `UnaryExpAST` 等) 能够在其类型推断逻辑中处理和存储 `float` 类型。
- **需修改文件：**
  - `SysyCompiler/src/AST.h`
  - `SysyCompiler/src/AST.cpp`
- **具体修改 (`AST.h`)：**
  1. 在 `BTypeAST` 中添加一个表示 `FLOAT` 的枚举值：
     ```c++
     class BTypeAST : public BaseAST {
     public:
         enum Type { INT_TYPE, VOID_TYPE, FLOAT_TYPE /* New */ };
         Type tag;
         // ...
     };
     ```
  2. 修改 `NumberAST` (如果用于字面量) 以支持 `float`：
     ```c++
     class NumberAST : public BaseAST {
     public:
         enum ValType { IS_INT, IS_FLOAT };
         ValType type_tag;
         union {
             int int_val;
             float float_val;
         };
         // Constructors for int and float
         // NumberAST(int val); NumberAST(float val);
         // ...
     };
     ```
  3. 表达式节点可能需要一个成员来存储其推断出的类型 (int 或 float)。
- **具体修改 (`AST.cpp`)：**
  1. 更新所有创建或使用 `BTypeAST` 和 `NumberAST` 的地方，以正确处理新的 `float` 相关逻辑。
  2. 表达式节点的构造函数和方法 (特别是用于类型检查或 `Dump` 的方法) 需要能处理操作数为 `float` 或结果为 `float` 的情况。
- **验收标准：**
  - 调试器检查：AST 能够正确表示 `float` 类型（在 `BTypeAST` 中）和浮点数字面量（在 `NumberAST` 中）。
  - 在后续的语义分析和 IR 生成阶段，能够从 AST 节点中准确获取 `float` 类型信息。

### 阶段四：语义分析 & Koopa IR 生成

- **任务：**
  1. 更新符号表中的类型系统 (`SysYType` 或等价物) 以包含 `float`。
  2. 实现类型检查：
     - 确保对 `float` 类型的操作是合法的。
     - 在二元运算（如 `+`, `-`, `*`, `/`, 비교）中，如果操作数之一是 `float`，另一个是 `int`，则需要进行隐式类型转换。
  3. 中间代码生成：
     - 为 `float` 类型的变量声明、函数参数/返回值生成相应的 Koopa IR 类型。
     - 为浮点数字面量生成 Koopa IR 表示。
     - 当 `int` 和 `float` 混合运算时，生成 `sitofp` (signed int to float) Koopa IR 指令进行类型提升。
     - 为浮点算术运算 (如 `fadd`, `fsub`, `fmul`, `fdiv`) 和比较运算 (`fcmp` 系列) 生成相应的 Koopa IR 指令。
- **需修改文件：**
  - `SysyCompiler/src/Symbol.h`
  - `SysyCompiler/src/Symbol.cpp`
  - `SysyCompiler/src/AST.cpp` (主要是各个 AST 节点的 `Dump()` 或类似方法)
- **具体修改 (`Symbol.h`)：**
  1. 在 `SysYType` 枚举中添加 `SYSY_FLOAT`, `SYSY_FLOAT_CONST` (如果区分), `SYSY_FUNC_FLOAT`。
     ```c++
     class SysYType {
     public:
         enum TYPE {
             SYSY_INT, SYSY_INT_CONST, /* ... */
             SYSY_FLOAT, SYSY_FLOAT_CONST, /* New */
             SYSY_FUNC_FLOAT /* New, for functions returning float */
         };
         TYPE ty;
         // ...
     };
     ```
- **具体修改 (`AST.cpp`)：**
  1. 在 `DeclAST`, `FuncDefAST` 等节点的 `Dump()` 方法中，为 `float` 类型生成正确的 Koopa IR 类型 (`float`)。
  2. 在处理算术和比较表达式的 AST 节点的 `Dump()` 方法中：
     - 获取左右操作数的类型。
     - 如果一个是 `int` 而另一个是 `float`，将 `int` 操作数通过 `sitofp` 指令转换为 `float`。
       ```koopa
       // %int_val: i32
       // %float_val_from_int = sitofp %int_val : float
       ```
     - 使用 Koopa IR 的浮点指令，如：
       ```koopa
       // %val1: float, %val2: float
       // %sum = fadd %val1, %val2
       // %diff = fsub %val1, %val2
       // %prod = fmul %val1, %val2
       // %quot = fdiv %val1, %val2
       // %eq = fcmp oeq %val1, %val2 ; oeq for ordered equal
       ```
  3. 为浮点数字面量生成 Koopa IR 常量表示。
     ```koopa
     // global @my_const_float: float = 3.14159
     // ...
     //   %const_val = global @my_const_float
     //   %loaded_const = load %const_val
     // 或者直接使用字面量，如果 Koopa IR 支持
     //   %another_const: float = 1.23
     ```
- **验收标准：**
  - 对于包含 `float` 声明、字面量、运算和函数调用的 SysY 代码，生成的 Koopa IR 在类型和操作上都是正确的。
  - `int` 到 `float` 的隐式转换 (`sitofp`) 在 IR 中清晰可见。
  - 使用如 `koopa-opt` (如果可用) 或手动检查，确认 IR 的语义正确性。
  - 对类型不兼容的操作（例如对 `float` 进行位运算），编译器应能报错。

### 阶段五：目标代码生成 (RISC-V)

- **任务：**
  1. 将与 `float` 相关的 Koopa IR 指令翻译成正确的 RISC-V 单精度浮点指令 (RV32F/RV64F 扩展)。
  2. 使用 RISC-V 的浮点寄存器 (`f0`-`f31`) 进行浮点运算和数据传递。
  3. 处理浮点数字面量的加载。
  4. 遵循 RISC-V ABI 处理函数调用中 `float` 参数的传递和返回值的获取 (通常通过 `fa0`-`fa7` 寄存器)。
- **需修改文件：**
  - `SysyCompiler/src/visit.h`
  - `SysyCompiler/src/visit.cpp`
- **具体修改 (`visit.cpp`)：**
  1. 在 `Visit` 函数（或其处理各种 Koopa IR 指令的重载版本）中添加对浮点指令的处理：
     - `koopa_ir::FAdd` -> `fadd.s fa, fb, fc`
     - `koopa_ir::FSub` -> `fsub.s fa, fb, fc`
     - `koopa_ir::FMul` -> `fmul.s fa, fb, fc`
     - `koopa_ir::FDiv` -> `fdiv.s fa, fb, fc`
     - `koopa_ir::FCmp` (various conditions) -> `feq.s`, `flt.s`, `fle.s` (可能需要结合 `bne`, `beq` 等跳转指令)。
     - `koopa_ir::SIToFP` (int to float) -> `fcvt.s.w fd, rs` (convert word in integer register `rs` to float in `fd`)
     - `koopa_ir::FPToSI` (float to int, if needed) -> `fcvt.w.s rd, fs` (convert float in `fs` to word in `rd`,注意取整模式)
  2. 处理 `load` 和 `store` Koopa 指令时，如果操作的是 `float` 类型：
     - `load` (float) -> `flw fd, offset(rs)` (load float word)
     - `store` (float) -> `fsw fs, offset(rd)` (store float word)
  3. 管理浮点寄存器的分配和使用。
  4. 加载浮点数字面量：通常将其存入 `.data` 段，然后使用 `lui` + `flw` (或 `auipc` + `flw`) 加载。
     ```assembly
     .data
     float_const_1: .float 3.14159
     .text
     # ...
     la t0, float_const_1
     flw ft0, 0(t0)
     ```
  5. 函数调用：
     - 参数传递：根据 RISC-V ABI，前几个浮点参数通过 `fa0` - `fa7` 传递。
     - 返回值：浮点返回值通常通过 `fa0` (或 `fa0`/`fa1` 对，如果需要) 返回。
- **验收标准：**
  - 生成的 RISC-V 汇编代码包含正确的浮点指令 (`.s` 后缀)。
  - 浮点运算使用浮点寄存器。
  - 代码在支持 F 扩展的 RISC-V 模拟器（如 Spike, QEMU）或硬件上能够正确执行。
  - 执行结果与预期（手动计算或与其他编译器比较）一致。

### 阶段六：全面测试

- **任务：**
  1. 编写大量包含 `float` 类型的测试用例 (`.sy` 文件)。
  2. 测试用例应覆盖：
     - `float` 变量和常量的声明与初始化。
     - 各种浮点算术运算。
     - 浮点比较运算。
     - `int` 与 `float` 的混合运算（测试隐式转换）。
     - 涉及 `float` 的函数（参数、返回值）。
     - `float` 数组的声明、访问和赋值。
     - 边界情况和特殊浮点值（如果适用，如 0.0, -0.0, NaN, Inf - 尽管 SysY 可能不要求处理这些）。
- **需修改文件：**
  - 在测试目录下创建新的 `.sy` 测试文件和对应的期望输出文件。
- **验收标准：**
  - 所有新的 `float` 相关测试用例通过编译且运行结果正确。
  - 原有的 `int` 相关测试用例不受影响，仍然通过。

## 3. 假设与前提

- 目标 RISC-V 架构支持单精度浮点扩展 (通常是 'F' 扩展，如 RV32IF, RV32G, RV64IF, RV64G)。
- Koopa IR 本身具有表示浮点类型、浮点字面量和浮点运算的能力。编译器的工作是正确生成这些 Koopa IR 结构。
- 项目的构建系统 (Makefile) 能够正确链接任何必要的运行时库（通常对于基本浮点运算，如果目标是裸机或标准系统，不需要特殊运行时库，但链接过程需要能处理生成的汇编）。

## 4. 总结

添加 `float` 支持是一个涉及编译器多个层面的重要更新。通过遵循上述步骤，细致地修改和测试每个阶段，可以成功地为 `SysyCompiler` 项目扩展此功能。务必在每个阶段后进行充分的单元测试和集成测试。
