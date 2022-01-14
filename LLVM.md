[TOC]
# LLVM essentials
## Chapter 1. Playing with LLVM

### LLVM IR looks like

```
; ModuleID = 'add.c'
source_filename = "add.c"
target datalayout = "e-m:e-p270:32:32-p271:32:32-p272:64:64-i64:64-f80:128-n8:16:32:64-S128"
target triple = "x86_64-unknown-linux-gnu"

@globvar = dso_local global i32 12, align 4

; Function Attrs: noinline nounwind optnone uwtable
define dso_local i32 @add(i32 %0) #0 {
  %2 = alloca i32, align 4
  store i32 %0, i32* %2, align 4
  %3 = load i32, i32* @globvar, align 4
  %4 = load i32, i32* %2, align 4
  %5 = add nsw i32 %3, %4
  ret i32 %5
}

attributes #0 = { noinline nounwind optnone uwtable "frame-pointer"="all" "min-legal-vector-width"="0" "no-trapping-math"="true" "stack-protector-buffer-size"="8" "target-cpu"="x86-64" "target-features"="+cx8,+fxsr,+mmx,+sse,+sse2,+x87" "tune-cpu"="generic" }

!llvm.module.flags = !{!0, !1, !2}
!llvm.ident = !{!3}

!0 = !{i32 1, !"wchar_size", i32 4}
!1 = !{i32 7, !"uwtable", i32 1}
!2 = !{i32 7, !"frame-pointer", i32 2}
!3 = !{!"clang version 14.0.0 (https://github.com/llvm/llvm-project.git 18d10fbe87b36cd922faeeb04d18078aea071c95)"}

```

### LLVM command line

- llvm-as  LLVM组装器

将.ll文件组装为bitcode形式的.bc文件

```shell
llvm-as add.ll -o add.bc
```
查看.bc文件内容
```shell
hexdump -c add.bc
```
- llvm-dis LLVM拆解器
将.bc文件拆解为.ll文件

- llvm-link 链接两个.bc文件并输出为一个.bc文件

```shell
llvm-link main.bc add.bc -o output.bc
```
- lli 直接在当前host上运行.bc文件
```shell
lli output.bc
```
- llc 静态编译器 将.ll文件或.bc文件组装为指定架构语言
```shell
llc output.bc -o output.s
```
- opt 模块化LLVM分析器和优化器

## Chapter 2. Building LLVM IR

### creating an LLVM module
```c++
#include "llvm/IR/LLVMContext.h"
#include "llvm/IR/Module.h"
using namespace llvm;

static LLVMContext TheContext;

static LLVMContext& getGlobalContext() {
    return TheContext;
}

static LLVMContext &Context = getGlobalContext();
static Module *ModuleOb = new Module("my compiler", Context);
// The first argument is the name of the module. The second argument is LLVMContext.

int main(int argc, char *argv[]) {
  ModuleOb->print(errs(), nullptr);
  return 0;
}                                                            
```
### Emitting a function in a module
```c++
#include "llvm/IR/IRBuilder.h"
#include "llvm/IR/LLVMContext.h"
#include "llvm/IR/Module.h"
#include "llvm/IR/Verifier.h"
#include <vector>
using namespace llvm;

static LLVMContext TheContext;
static LLVMContext& getGlobalContext() {
    return TheContext;
}
static LLVMContext &Context = getGlobalContext();
static Module *ModuleOb = new Module("my compiler", Context);//创建module实例
// 创建自定义函数
// 创建一个返回值为int类型的函数fooFunc()
Function *createFunc(IRBuilder<> &Builder, std::string Name) {
  FunctionType *funcType = llvm::FunctionType::get(Builder.getInt32Ty(), false);//设置返回类型为int32
  Function *fooFunc = llvm::Function::Create(
      funcType, llvm::Function::ExternalLinkage, Name, ModuleOb);//创建一个function
  return fooFunc;
}

int main(int argc, char *argv[]) {
  static IRBuilder<> Builder(Context);//产生LLVM IR
  Function *fooFunc = createFunc(Builder, "foo");
  verifyFunction(*fooFunc);//对生成代码进行检查
  ModuleOb->print(errs(), nullptr);
  return 0;
}                                                                         
```
```shell 
$ clang++ module.cpp `llvm-config --cxxflags --ldflags --system-libs --libs core` -fno-rtti -o toy
$ ./toy
```
### Adding a block to a function
basic blocks包含一系列IR指令和出口。BasicBlock class用于产生和处理basic blocks，给basic blocks设置一个入口标识，插入basic blocks中IR指令。运用IRBuilder生成新的basic blocks
```c++
#include "llvm/IR/IRBuilder.h"                                                  
#include "llvm/IR/LLVMContext.h"
#include "llvm/IR/Module.h"
#include "llvm/IR/Verifier.h"
#include <vector>
using namespace llvm;

static LLVMContext TheContext;
static LLVMContext& getGlobalContext() {
    return TheContext;
}
static LLVMContext &Context = getGlobalContext();
static Module *ModuleOb = new Module("my compiler", Context);

Function *createFunc(IRBuilder<> &Builder, std::string Name) {
  FunctionType *funcType = llvm::FunctionType::get(Builder.getInt32Ty(), false);
  Function *fooFunc = llvm::Function::Create(
      funcType, llvm::Function::ExternalLinkage, Name, ModuleOb);
  return fooFunc;
}
//生成basic blocks
BasicBlock *createBB(Function *fooFunc, std::string Name) {
  return BasicBlock::Create(Context, Name, fooFunc);
}

int main(int argc, char *argv[]) {
  static IRBuilder<> Builder(Context);
  Function *fooFunc = createFunc(Builder, "foo");
  BasicBlock *entry = createBB(fooFunc, "entry");//给foo函数设置入口
  Builder.SetInsertPoint(entry);//在此处插入foo中IR指令
  verifyFunction(*fooFunc);
  ModuleOb->print(errs(), nullptr);
  return 0;
}
```
### Emitting a global variable
全局变量对于给定module所有function可见。GlobalVariable class创建全局变量。getOrInsertGlobal()是创建全局变量方法，It takes two arguments—the first is the
name of the variable and the second is the data type of the variable.
```c++
#include "llvm/IR/IRBuilder.h"                                                  
#include "llvm/IR/LLVMContext.h"
#include "llvm/IR/Module.h"
#include "llvm/IR/Verifier.h"
#include <vector>
using namespace llvm;

static LLVMContext TheContext;
static LLVMContext& getGlobalContext() {
    return TheContext;
}
static LLVMContext &Context = getGlobalContext();
static Module *ModuleOb = new Module("my compiler", Context);

Function *createFunc(IRBuilder<> &Builder, std::string Name) {
  FunctionType *funcType = llvm::FunctionType::get(Builder.getInt32Ty(), false);
  Function *fooFunc = llvm::Function::Create(
      funcType, llvm::Function::ExternalLinkage, Name, ModuleOb);
  return fooFunc;
}

BasicBlock *createBB(Function *fooFunc, std::string Name) {
  return BasicBlock::Create(Context, Name, fooFunc);
}

GlobalVariable *createGlob(IRBuilder<> &Builder, std::string Name) {
    ModuleOb->getOrInsertGlobal(Name, Builder.getInt32Ty());//创建全局变量，类型为int32
    GlobalVariable *gVar = ModuleOb->getNamedGlobal(Name);//获取全局变量
    gVar->setLinkage(GlobalValue::CommonLinkage);//设置链接方式
    gVar->setAlignment(MaybeAlign(4));//设置对齐方式
    return gVar;
}

int main(int argc, char *argv[]) {
  static IRBuilder<> Builder(Context);
  GlobalVariable *gVar = createGlob(Builder, "x");//设置全局变量x
  Function *fooFunc = createFunc(Builder, "foo");
  BasicBlock *entry = createBB(fooFunc, "entry");
  Builder.SetInsertPoint(entry);
  verifyFunction(*fooFunc);
  ModuleOb->print(errs(), nullptr);
  return 0;
}
```
Linkage 可设置为inline weak static等
![](/home/mi/Pictures/Screenshot from 2022-01-13 10-39-30.png)

Alignment最大为29位对齐
### Emitting a return statement
```c++
#include "llvm/IR/IRBuilder.h"                                                  
#include "llvm/IR/LLVMContext.h"
#include "llvm/IR/Module.h"
#include "llvm/IR/Verifier.h"
#include <vector>
using namespace llvm;

static LLVMContext TheContext;
static LLVMContext& getGlobalContext() {
    return TheContext;
}
static LLVMContext &Context = getGlobalContext();
static Module *ModuleOb = new Module("my compiler", Context);

Function *createFunc(IRBuilder<> &Builder, std::string Name) {
  FunctionType *funcType = llvm::FunctionType::get(Builder.getInt32Ty(), false);
  Function *fooFunc = llvm::Function::Create(
      funcType, llvm::Function::ExternalLinkage, Name, ModuleOb);
  return fooFunc;
}

BasicBlock *createBB(Function *fooFunc, std::string Name) {
  return BasicBlock::Create(Context, Name, fooFunc);
}

GlobalVariable *createGlob(IRBuilder<> &Builder, std::string Name) {
    ModuleOb->getOrInsertGlobal(Name, Builder.getInt32Ty());
    GlobalVariable *gVar = ModuleOb->getNamedGlobal(Name);
    gVar->setLinkage(GlobalValue::CommonLinkage);
    gVar->setAlignment(MaybeAlign(4));
    return gVar;
}

int main(int argc, char *argv[]) {
  static IRBuilder<> Builder(Context);
  GlobalVariable *gVar = createGlob(Builder, "x");
  Function *fooFunc = createFunc(Builder, "foo");
  BasicBlock *entry = createBB(fooFunc, "entry");
  Builder.SetInsertPoint(entry);
  Builder.CreateRet(Builder.getInt32(0));//设置返回值为0
  verifyFunction(*fooFunc);
  ModuleOb->print(errs(), nullptr);
  return 0;
}
```
### Emitting function arguments
```c++
#include "llvm/IR/IRBuilder.h"
#include "llvm/IR/LLVMContext.h"
#include "llvm/IR/Module.h"
#include "llvm/IR/Verifier.h"
#include <vector>
using namespace llvm;

static LLVMContext TheContext;
static LLVMContext& getGlobalContext() {
    return TheContext;
}
static LLVMContext &Context = getGlobalContext();
static Module *ModuleOb = new Module("my compiler", Context);
static std::vector<std::string> FunArgs;//保存函数参数名

Function *createFunc(IRBuilder<> &Builder, std::string Name) {
  std::vector<Type *> Integers(FunArgs.size(), Type::getInt32Ty(Context));//设置参数类型为int32
  FunctionType *funcType = llvm::FunctionType::get(Builder.getInt32Ty(), Integers, false);//设置参数为int32类型
  Function *fooFunc = llvm::Function::Create(
      funcType, llvm::Function::ExternalLinkage, Name, ModuleOb);
  return fooFunc;
}

void setFuncArgs(Function *fooFunc, std::vector<std::string> FunArgs) {
    unsigned Idx = 0;
    Function::arg_iterator AI, AE;
    for (AI = fooFunc->arg_begin(), AE = fooFunc->arg_end(); AI != AE;
         ++AI, ++Idx) //for-loop进行参数命名
        AI->setName(FunArgs[Idx]);
}

BasicBlock *createBB(Function *fooFunc, std::string Name) {
  return BasicBlock::Create(Context, Name, fooFunc);
}

GlobalVariable *createGlob(IRBuilder<> &Builder, std::string Name) {
    ModuleOb->getOrInsertGlobal(Name, Builder.getInt32Ty());
    GlobalVariable *gVar = ModuleOb->getNamedGlobal(Name);
    gVar->setLinkage(GlobalValue::CommonLinkage);
    gVar->setAlignment(MaybeAlign(4));                                                     
    return gVar;
}

int main(int argc, char *argv[]) {
  FunArgs.push_back("a");
  FunArgs.push_back("b");
  static IRBuilder<> Builder(Context);
  GlobalVariable *gVar = createGlob(Builder, "x");
  Function *fooFunc = createFunc(Builder, "foo");
  setFuncArgs(fooFunc, FunArgs);
  BasicBlock *entry = createBB(fooFunc, "entry");
  Builder.SetInsertPoint(entry);
  Builder.CreateRet(Builder.getInt32(0));
  verifyFunction(*fooFunc);
  ModuleOb->print(errs(), nullptr);
  return 0;
}
```
### Emitting a simple arithmetic statement in a basic block
基本算数语句
二元运算API include/llvm/IR/IRBuild.h
```c++
#include "llvm/IR/IRBuilder.h"
#include "llvm/IR/LLVMContext.h"
#include "llvm/IR/Module.h"
#include "llvm/IR/Verifier.h"
#include <vector>
using namespace llvm;

static LLVMContext TheContext;
static LLVMContext& getGlobalContext() {
    return TheContext;
}
static LLVMContext &Context = getGlobalContext();
static Module *ModuleOb = new Module("my compiler", Context);
static std::vector<std::string> FunArgs;

Function *createFunc(IRBuilder<> &Builder, std::string Name) {
  std::vector<Type *> Integers(FunArgs.size(), Type::getInt32Ty(Context));
  FunctionType *funcType = llvm::FunctionType::get(Builder.getInt32Ty(), Integers, false);
  Function *fooFunc = llvm::Function::Create(
      funcType, llvm::Function::ExternalLinkage, Name, ModuleOb);
  return fooFunc;
}

void setFuncArgs(Function *fooFunc, std::vector<std::string> FunArgs) {
    unsigned Idx = 0;
    Function::arg_iterator AI, AE;
    for (AI = fooFunc->arg_begin(), AE = fooFunc->arg_end(); AI != AE;
         ++AI, ++Idx)
        AI->setName(FunArgs[Idx]);
}

BasicBlock *createBB(Function *fooFunc, std::string Name) {
  return BasicBlock::Create(Context, Name, fooFunc);
}

GlobalVariable *createGlob(IRBuilder<> &Builder, std::string Name) {
    ModuleOb->getOrInsertGlobal(Name, Builder.getInt32Ty());
    GlobalVariable *gVar = ModuleOb->getNamedGlobal(Name);
    gVar->setLinkage(GlobalValue::CommonLinkage);
    gVar->setAlignment(MaybeAlign(4));
    return gVar;
}

Value *createArith(IRBuilder<> &Builder, Value *L, Value *R) {
  return Builder.CreateMul(L, R, "multmp");//乘法
}

int main(int argc, char *argv[]) {
  FunArgs.push_back("a");
  FunArgs.push_back("b");
  static IRBuilder<> Builder(Context);
  GlobalVariable *gVar = createGlob(Builder, "x");
  Function *fooFunc = createFunc(Builder, "foo");
  setFuncArgs(fooFunc, FunArgs);
  BasicBlock *entry = createBB(fooFunc, "entry");
  Builder.SetInsertPoint(entry);
  Value *Arg1 = fooFunc->arg_begin(); //第一个参数，foo函数的第一个参数 默认值0
  Value *constant = Builder.getInt32(16);//第二个参数，16
  Value *val = createArith(Builder, Arg1, constant);//乘
  Builder.CreateRet(val);//返回乘法结果
  verifyFunction(*fooFunc);
  ModuleOb->print(errs(), nullptr);
  return 0;
}
```
### Emitting if-else condition IR
```c++
int max(int a, int b) {
  if (a > b) {
    return a + 1;
  } else {
    return b * 16;
  }
}
```
```c++
#include "llvm/IR/LLVMContext.h"
#include "llvm/IR/Module.h"
#include "llvm/IR/IRBuilder.h"
#include "llvm/Support/raw_ostream.h"

using namespace llvm;

static std::unique_ptr<LLVMContext> TheContext;
static std::unique_ptr<Module> TheModule;
static std::unique_ptr<IRBuilder<>> Builder;

static void InitializeModule() {
  TheContext = std::make_unique<LLVMContext>();
  TheModule = std::make_unique<Module>("first modlue", *TheContext);
  Builder = std::make_unique<IRBuilder<>>(*TheContext);
}

Function *createFunc(Type *RetTy, ArrayRef<Type *> Params, std::string Name, bool isVarArg = false) {
  FunctionType *funcType = FunctionType::get(RetTy, Params, isVarArg);
  Function *fooFunc = Function::Create(funcType, Function::ExternalLinkage, Name, TheModule.get());
  return fooFunc;
}

void setFuncArgs(Function *Func, std::vector<std::string> FuncArgs) {
  unsigned Idx = 0;
  Function::arg_iterator AI, AE;
  for(AI = Func->arg_begin(), AE = Func->arg_end(); AI != AE; ++AI, ++Idx) {
      AI->setName(FuncArgs[Idx]);
    }
}

BasicBlock *createBB(Function *fooFunc, std::string Name) {
  return BasicBlock::Create(*TheContext, Name, fooFunc);
}
//创建大小比较函数
Function *createMaxProto(std::string funcName) {
  Function *fooFunc = createFunc(Builder->getInt32Ty(), {Builder->getInt32Ty(), Builder->getInt32Ty()}, funcName);
  //返回值int32类型，两个参数为int32类型，name为funcName
  std::vector<std::string> FuncArgs;
  FuncArgs.push_back("a");
  FuncArgs.push_back("b");
  setFuncArgs(fooFunc, FuncArgs);//传入参数名
  return fooFunc;
}


void createMax_phi() {
  Function *fooFunc = createMaxProto("max");

  // args
  Function::arg_iterator AI = fooFunc->arg_begin();
  Value *Arg1 = AI++;
  Value *Arg2 = AI;

  BasicBlock *entry = createBB(fooFunc, "entry");
  BasicBlock *ThenBB = createBB(fooFunc, "then");
  BasicBlock *ElseBB = createBB(fooFunc, "else");
  BasicBlock *MergeBB = createBB(fooFunc, "ifcont");

  // entry
  Builder->SetInsertPoint(entry);
  // if condition
  Value *Compare = Builder->CreateICmpULT(Arg1, Arg2, "cmptmp");
  Value *Cond = Builder->CreateICmpNE(Compare, Builder->getInt1(false), "ifcond");
  Builder->CreateCondBr(Cond, ThenBB, ElseBB);//条件跳转分支

  // Then 
  Builder->SetInsertPoint(ThenBB);
  Value *ThenVal = Builder->CreateAdd(Arg1, Builder->getInt32(1), "thenVal");
  Builder->CreateBr(MergeBB);//跳转到mergeBB

  // else
  Builder->SetInsertPoint(ElseBB);
  Value *ElseVal = Builder->CreateMul(Arg2, Builder->getInt32(16), "elseVal");
  Builder->CreateBr(MergeBB);//跳转到mergeBB

  // end
  Builder->SetInsertPoint(MergeBB);
  PHINode *Phi = Builder->CreatePHI(Builder->getInt32Ty(), 2, "iftmp");//创建一个phinode，包含两个预留值
  //填充两个预留值，如果来自thenBB分支，则phi值为thenVal，反之为ElseVal
  Phi->addIncoming(ThenVal, ThenBB);
  Phi->addIncoming(ElseVal, ElseBB);

  Builder->CreateRet(Phi);//phi值作为返回值
}


int main(int argc, char *argv[]) {
  InitializeModule();

  createMax_phi();

  TheModule->print(outs(), nullptr);
  return 0;
}
```
输出

```
; ModuleID = 'first modlue'
source_filename = "first modlue"

define i32 @max(i32 %a, i32 %b) {
entry:
  %cmptmp = icmp ult i32 %a, %b
  %ifcond = icmp ne i1 %cmptmp, false
  br i1 %ifcond, label %then, label %else

then:                                             ; preds = %entry
  %thenVal = add i32 %a, 1
  br label %ifcont

else:                                             ; preds = %entry
  %elseVal = mul i32 %b, 16
  br label %ifcont

ifcont:                                           ; preds = %else, %then
  %iftmp = phi i32 [ %thenVal, %then ], [ %elseVal, %else ]
  ret i32 %iftmp
}
```



### Emitting LLVM IR for loop
函数原型
```c++
int sum(int a, int b) {
  int addTmp = a - b;
  for (int i = a; i <= b; i++) {
    addTmp = addTmp + i;
  }
  return addTmp;
}
```

```c++
#include "llvm/IR/LLVMContext.h"
#include "llvm/IR/Module.h"
#include "llvm/IR/IRBuilder.h"
#include "llvm/Support/raw_ostream.h"

using namespace llvm;

static std::unique_ptr<LLVMContext> TheContext;
static std::unique_ptr<Module> TheModule;
static std::unique_ptr<IRBuilder<>> Builder;

static void InitializeModule() {
  TheContext = std::make_unique<LLVMContext>();
  TheModule = std::make_unique<Module>("first modlue", *TheContext);
  Builder = std::make_unique<IRBuilder<>>(*TheContext);
}

Function *createFunc(Type *RetTy, ArrayRef<Type *> Params, std::string Name, bool isVarArg = false) {
  FunctionType *funcType = FunctionType::get(RetTy, Params, isVarArg);
  Function *fooFunc = Function::Create(funcType, Function::ExternalLinkage, Name, TheModule.get());
  return fooFunc;
}

void setFuncArgs(Function *Func, std::vector<std::string> FuncArgs) {
  unsigned Idx = 0;
  Function::arg_iterator AI, AE;
  for(AI = Func->arg_begin(), AE = Func->arg_end(); AI != AE; ++AI, ++Idx) {
      AI->setName(FuncArgs[Idx]);
    }
}

BasicBlock *createBB(Function *fooFunc, std::string Name) {
  return BasicBlock::Create(*TheContext, Name, fooFunc);
}
//创建函数原型
Function *createMaxProto(std::string funcName) {
  Function *fooFunc = createFunc(Builder->getInt32Ty(), {Builder->getInt32Ty(), Builder->getInt32Ty()}, funcName);
  std::vector<std::string> FuncArgs;
  FuncArgs.push_back("a");
  FuncArgs.push_back("b");
  setFuncArgs(fooFunc, FuncArgs);
  return fooFunc;
}

void createSum() {
  Function *fooFunc = createMaxProto("sum");

  // args
  Function::arg_iterator AI = fooFunc->arg_begin();
  Value *StartVal = AI++;
  Value *EndVal = AI;

  BasicBlock *entryBB = createBB(fooFunc, "entry");
  BasicBlock *loopBB = createBB(fooFunc, "loop");
  BasicBlock *endEntryBB = createBB(fooFunc, "endEntry");
  BasicBlock *endLoopBB = createBB(fooFunc, "endLoop");

  // entry
  // entry中的指令先设置初始值给initVal, 然后根据a和b的大小决定是否进入loop:
  Builder->SetInsertPoint(entryBB);
  Value *initVal = Builder->CreateSub(StartVal, EndVal, "init");
  Value *EndCond = Builder->CreateICmpULE(StartVal, EndVal, "entryEndCond");
  EndCond = Builder->CreateICmpNE(EndCond, Builder->getInt1(false), "entryCond");
  Builder->CreateCondBr(EndCond, loopBB, endEntryBB);

  // loop 
  // 先为i和sum创建PHINode, 然后i自增1，sum自增i,最后判断i<=b来决定是否跳出loop
  Builder->SetInsertPoint(loopBB);

  PHINode *iPhi = Builder->CreatePHI(Builder->getInt32Ty(), 2, "i");
  iPhi->addIncoming(StartVal, entryBB);
  PHINode *sumPhi = Builder->CreatePHI(Builder->getInt32Ty(), 2, "sum");
  
  sumPhi->addIncoming(initVal, loopBB);
  Value *nextI = Builder->CreateAdd(iPhi, Builder->getInt32(1), "nextI");
  Value *nextSum = Builder->CreateAdd(sumPhi, iPhi, "nextSum");

  EndCond = Builder->CreateICmpULE(nextI, EndVal, "loopEndCond");
  EndCond = Builder->CreateICmpNE(EndCond, Builder->getInt1(false), "loopCond");
  Builder->CreateCondBr(EndCond, loopBB, endLoopBB);

  iPhi->addIncoming(nextI, loopBB);
  sumPhi->addIncoming(nextSum, loopBB);

  // endLoopBB
  Builder->SetInsertPoint(endLoopBB);
  Builder->CreateRet(sumPhi);

  // endInit
  Builder->SetInsertPoint(endEntryBB);
  Builder->CreateRet(initVal);
}


int main(int argc, char *argv[]) {
  InitializeModule();

  createSum();

  TheModule->print(outs(), nullptr);
  return 0;
}
```
## Chapter 3. Advanced LLVM IR
### Memory access operations
### Getting the address of an element

通过getelementptr指令[<sup>1</sup>](#refer-1)获取一个数据结构的地址，只计算地址，不访问内存
```
%a1 = getelementptr i32, <2 x i32>* %a, i32 1
```
第一个参数为计算指针的类型




## Reference
<div id="refer-1"></div>
- [getelementptr](https://www.kancloud.cn/digest/xf-llvm/162268)

