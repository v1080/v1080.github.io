## LLVM的基本设计原理及其历史
略

## 理解目前的LLVM
基础架构中最重要的组成部分：
- **前端**：将计算机程序语言转换为LLVM IR。包括词法分析器、语法分析器、语义分析器、LLVM IR代码生成器。
- **IR**：LLVM IR既有可读的表现形式、也有二进制表现形式。提供IR构建、组装和拆卸的接口。LLVM优化器还可以处理IR。
- **后端**：将LLVM IR转换为特定于目标的汇编代码或二进制文件。包含寄存器分配、循环转换、窥视孔优化器及特定于目标的优化器。

总体结构：
Clang前端或带DragonEgg的GCC -> LLVM IR链接器 -> LLVM IR优化器 -> LLVM后端 -> LLVM集成汇编器 -> GCC链接器或LLD ->二进制程序


## 与编译器驱动程序交互
为简单的程序生成可执行文件：
```
~ clang test.c -o test
```
如果想查看调用的其他工具，使用```-###```命令：
```
~ clang -### test.c -o test
Apple clang version 13.0.0 (clang-1300.0.29.3)
Target: x86_64-apple-darwin20.6.0
Thread model: posix
InstalledDir: /Applications/Xcode.app/Contents/Developer/Toolchains/XcodeDefault.xctoolchain/usr/bin
"/Applications/Xcode.app/Contents/Developer/Toolchains/XcodeDefault.xctoolchain/usr/bin/clang" "-cc1" "-triple" "x86_64-apple-macosx11.0.0" "-Wundef-prefix=TARGET_OS_" "-Wdeprecated-objc-isa-usage" "-Werror=deprecated-objc-isa-usage" "-Werror=implicit-function-declaration" "-emit-obj" "-mrelax-all" "--mrelax-relocations" "-disable-free" "-disable-llvm-verifier" "-discard-value-names" "-main-file-name" "test.c" "-mrelocation-model" "pic" "-pic-level" "2" "-mframe-pointer=all" "-fno-strict-return" "-fno-rounding-math" "-munwind-tables" "-target-sdk-version=11.3" "-fvisibility-inlines-hidden-static-local-var" "-target-cpu" "penryn" "-tune-cpu" "generic" "-debugger-tuning=lldb" "-target-linker-version" "711" "-resource-dir" "/Applications/Xcode.app/Contents/Developer/Toolchains/XcodeDefault.xctoolchain/usr/lib/clang/13.0.0" "-isysroot" "/Applications/Xcode.app/Contents/Developer/Platforms/MacOSX.platform/Developer/SDKs/MacOSX.sdk" "-I/usr/local/include" "-internal-isystem" "/Applications/Xcode.app/Contents/Developer/Platforms/MacOSX.platform/Developer/SDKs/MacOSX.sdk/usr/local/include" "-internal-isystem" "/Applications/Xcode.app/Contents/Developer/Toolchains/XcodeDefault.xctoolchain/usr/lib/clang/13.0.0/include" "-internal-externc-isystem" "/Applications/Xcode.app/Contents/Developer/Platforms/MacOSX.platform/Developer/SDKs/MacOSX.sdk/usr/include" "-internal-externc-isystem" "/Applications/Xcode.app/Contents/Developer/Toolchains/XcodeDefault.xctoolchain/usr/include" "-Wno-reorder-init-list" "-Wno-implicit-int-float-conversion" "-Wno-c99-designator" "-Wno-final-dtor-non-final-class" "-Wno-extra-semi-stmt" "-Wno-misleading-indentation" "-Wno-quoted-include-in-framework-header" "-Wno-implicit-fallthrough" "-Wno-enum-enum-conversion" "-Wno-enum-float-conversion" "-Wno-elaborated-enum-base" "-fdebug-compilation-dir" "/Users/vvv/Desktop" "-ferror-limit" "19" "-stack-protector" "1" "-fstack-check" "-mdarwin-stkchk-strong-link" "-fblocks" "-fencode-extended-block-signature" "-fregister-global-dtors-with-atexit" "-fgnuc-version=4.2.1" "-fmax-type-align=16" "-fcommon" "-fcolor-diagnostics" "-clang-vendor-feature=+nullptrToBoolConversion" "-clang-vendor-feature=+messageToSelfInClassMethodIdReturnType" "-clang-vendor-feature=+disableInferNewAvailabilityFromInit" "-clang-vendor-feature=+disableNeonImmediateRangeCheck" "-clang-vendor-feature=+disableNonDependentMemberExprInCurrentInstantiation" "-fno-odr-hash-protocols" "-clang-vendor-feature=+revert09abecef7bbf" "-mllvm" "-disable-aligned-alloc-awareness=1" "-mllvm" "-enable-dse-memoryssa=0" "-o" "/var/folders/px/lc2mhj_n7fgc38fsj39m71j80000gn/T/test-80f13d.o" "-x" "c" "test.c"
"/Applications/Xcode.app/Contents/Developer/Toolchains/XcodeDefault.xctoolchain/usr/bin/ld" "-demangle" "-lto_library" "/Applications/Xcode.app/Contents/Developer/Toolchains/XcodeDefault.xctoolchain/usr/lib/libLTO.dylib" "-no_deduplicate" "-dynamic" "-arch" "x86_64" "-platform_version" "macos" "11.0.0" "11.3" "-syslibroot" "/Applications/Xcode.app/Contents/Developer/Platforms/MacOSX.platform/Developer/SDKs/MacOSX.sdk" "-o" "test" "-L/usr/local/lib" "/var/folders/px/lc2mhj_n7fgc38fsj39m71j80000gn/T/test-80f13d.o" "-lSystem" "/Applications/Xcode.app/Contents/Developer/Toolchains/XcodeDefault.xctoolchain/usr/lib/clang/13.0.0/lib/darwin/libclang_rt.osx.a"
```

可以看到使用了clang自身及链接程序ld

## 使用独立工具
新建两个文件：
```
/* main.c */
#include <stdio.h>

int sum(int x, int y);
int main() {
    int r = sum(3, 4);
    printf("r = %d", r);
    return 0;
}
```

```
/* sum.c */
int sum(int x, int y) {
    return x+y;
}
```
然后执行：
```
~ clang -emit-llvm -c main.c -o main.bc
~ clang -emit-llvm -c sum.c -o sum.bc
```

```-emit-llvm```标志告诉clang根据是否存在```-c```或```-S```标志来生成LLVM bitcode或者LLVM汇编码文件
|命令|产物|
|:-----|:-----|
|```-c```|LLVM bitcode|
|```-S```|LLVM 汇编码|


使用bitcode文件完成后续编译，可以采用两种方式：

 1. 从每个bitcode文件生成特定于目标的目标文件，再链接成可执行文件:
```
~ llc -filetype=obj main.bc -o main.o
~ llc -filetype=obj sum.bc -o main.o
~ clang main.o sum.o -o sum
```

 2. 先将bitcode链接成最终的bitcode文件，然后再生成可执行文件:
```
~ llvm-link main.bc sum.bc -o sum.linked.bc
~ llc -filetype=obj sum.linked.bc sum.linked.o
~ clang sum.linked.o -o sum
```

## 深入LLVM内部设计

### 了解LLVM的基本库
- **libLLVMCore**：包含与LLVM IR相关的所有逻辑：IR构造、IR校验器。负责管理编译器中各种编译流程。
- **libLLVMAnalysis**：包含几个IR分析过程：别名分析、依赖分析等
- **lbLLVMCodeGen**：实现与目标无关的代码生成、机器级别的分析和转换
- **libLLVMTarget**：提供对目标机器信息的抽象访问接口。
- **libLLVMX86CodeGen**：特定于x86的目标代码生成信息、转换和分析过程，他们组成x86后端。类似还有ARM的LLVMARMCodeGen，MIPS的LLVMMipsCodeGen。
- **libLLVMSupport**：通用工具集合。
- **libclang**：Clang大部分功能
- **libclangDriver**：编译器驱动程序工具使用的一组类，用于理解类似GCC的命令行参数
- **libclangAnalysis**：Clang提供的一组前端分析器。
