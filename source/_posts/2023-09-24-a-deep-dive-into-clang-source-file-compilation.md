layout: post
title: A deep dive into Clang's source file compilation
author: MaskRay
tags: [llvm,clang]
toc: true
---

Clang is a C/C++ compiler that generates LLVM IR and utilitizes LLVM to generate relocatable object files.
Using the classic three-stage compiler structure, the stages can be described as follows:
```
C/C++ =(front end)=> LLVM IR =(middle end)=> LLVM IR (optimized) =(back end)=> relocatable object file
```

If we follow the internal representations of instructions, a more detailed diagram looks like this:
```
C/C++ =(front end)=> LLVM IR =(middle end)=> LLVM IR (optimized) =(instruction selector)=> MachineInstr =(AsmPrinter)=> MCInst =(assembler)=> relocatable object file
```

LLVM and Clang are designed as a collection of libraries.
This post describes how different libraries work together to create the final relocatable object file. I will focus on how a function goes through the multiple compilation stages.

<!-- more -->

## Compiler frontend

The compiler frontend primarily comprises the following libraries:

* clangDriver
* clangFrontend
* clangParse and clangSema
* clangCodeGen

The clangDriver library is located in `clang/lib/Driver/` and `clang/include/clang/Driver/`, while other libraries have similar structures.
In general, when a header file in one library (let's call it library A) is needed by another library, it is exposed to `clang/include/clang/$A/`.
Downstream projects can also include the header file using `#include <clang/$A/$file.h>`.

Let's use a C++ source file as an example.

```
% cat a.cc
template <typename T>
T div(T a, T b) {
  return a / b;
}

__attribute__((noinline))
int foo(int a, float b, int c) {
  int s = a + (b == b);
  return div(s, c);
}

int main() {
  return foo(3, 2, 1);
}
% clang++ -g a.cc
```

The entry point of the Clang executable is implemented in `clang/tools/driver/`.
`clang_main` creates a `clang::driver::Driver` instance, calls `BuildCompilation` to construct a `clang::driver::Compilation` instance, and then calls `ExecuteCompilation`.

### clangDriver

[clangDriver](https://clang.llvm.org/docs/DriverInternals.html) parses the command line arguments, constructs compilation actions, assigns actions to tools, generates commands for these tools, and executes the commands.

You may read [Compiler driver and cross compilation](/blog/2021-03-28-compiler-driver-and-cross-compilation#clang-driver) for additional information.

```
BuildCompilation
  getToolchain
  HandleImmediateArgs
  BuildInputs
  BuildActions
    handleArguments
  BuildJobs
    BuildJobsForAction
      ToolChain::SelectTool
      Clang::ConstructJob
        Clang::RenderTargetOptions
        renderDebugOptions
ExecuteCompilation
  ExecuteJobs
    ExecuteJob
      CC1Command::Execute
        cc1_main
```

For `clang++ -g a.cc`, clangDriver identifies the following phases: preprocessor, compiler (C++ to LLVM IR), backend, assembler, and linker.
The first several phases can be performed by one single `clang::driver::tools::Clang` object (also known as Clang cc1), while the final phase requires an external program (the linker).

```
% clang++ -g a.cc '-###'
...
 "/tmp/Rel/bin/clang-18" "-cc1" "-triple" "x86_64-unknown-linux-gnu" "-emit-obj" ...
 "/usr/bin/ld" "-pie" ... -o a.out ... /tmp/a-f58f75.o ...
```

`cc1_main` in clangDriver calls `ExecuteCompilerInvocation` defined in clangFrontend.

### clangFrontend

`clangFrontend` defines `CompilerInstance`, which manages various classes, including `CompilerInvocation`, `DiagnosticsEngine`, `TargetInfo`, `FileManager`, `SourceManager`, `Preprocessor`, `ASTContext`, `ASTConsumer`, and `Sema`.

```
ExecuteCompilerInvocation
  CreateFrontendAction
  ExecuteAction
    FrontendAction::BeginSourceFile
      CompilerInstance::createFileManager
      CompilerInstance::createSourceManager
      CompilerInstance::createPreprocessor
      CompilerInstance::createASTContext
      CreateWrappedASTConsumer
        BackendConsumer::BackendConsumer
          CodeGenerator::CodeGenerator
      CompilerInstance::setASTConsumer
        CodeGeneratorImpl::Initialize
          CodeGenModule::CodeGenModule
    FrontendAction::Execute
      FrontendAction::ExecutionAction => CodeGenAction
        ASTFrontendAction::ExecuteAction
          CompilerInstance::createSema
          ParseAST
    FrontendAction::EndSourceFile
```

In `ExecuteCompilerInvocation`, a `FrontAction` is created based on the `CompilerInstance` argument and then executed.
When using the `-emit-obj` option, the selected `FrontAction` is an `EmitObjAction`, which is a derivative of `CodeGenAction`.

During `FrontendAction::BeginSourceFile`, several classes mentioned earlier are created, and a `BackendConsumer` is also established.
The `BackendConsumer` serves as a wrapper around `CodeGenerator`, which is another derivative of `ASTConsumer`.
Finally, in `FrontendAction::BeginSourceFile`, `CompilerInstance::setASTConsumer` is called to create a `CodeGenModule` object, responsible for managing an LLVM IR module.

In `FrontendAction::Execute`, `CodeGenAction::ExecuteAction` is invoked, primarily handling the compilation of LLVM IR files.
This function, in turn, calls the base function `ASTFrontendAction::ExecuteAction`, which, in essence, triggers the entry point of `clangParse`: `ParseAST`.

### clangParse and clangSema

`clangParse` consumes tokens from `clangLex` and invokes parser actions, many of which are named `Act*`, defined in `clangSema`.
`clangSema` performs semantic analysis and generates AST nodes.

```
ParseAST
  ParseFirstTopLevelDecl
    Sema::ActOnStartOfTranslationUnit
  ParseTopLevelDecl
    ParseDeclarationOrFunctionDefinition
      ParseDeclOrFunctionDefInternal
        ParseDeclGroup
          ParseFunctionDefinition
            ParseFunctionStatementBody
              ParseCompoundStatementBody
                ParseStatementOrDeclaration
                  ParseStatementOrDeclarationAfterAttributes
                    Sema::ActOnDeclStmt
                Sema::ActOnCompoundStmt
              Sema::ActOnFinishFunctionBody
          Sema::ConvertDeclToDeclGroup
    ActOnEndOfTranslationUnit
      ActOnEndOfTranslationUnitFragment
        PerformPendingInstantiations
          InstantiateFunctionDefinition
  BackendConsumer::HandleTopLevelDecl
  BackendConsumer::HandleTranslationUnit
```

When `ParseTopLevelDecl` consumes a `tok::eof` token, implicit instantiations are performed.

In the end, we get a full AST (actually a misnomer as the representation is not abstract, not only about syntax, and is not a tree).
`ParseAST` calls virtual functions `HandleTopLevelDecl` and `HandleTranslationUnit`.

### clangCodeGen

`BackendConsumer` defined in clangCodeGen overrides `HandleTopLevelDecl` and `HandleTranslationUnit` to generate unoptimized LLVM IR and hand over the IR to LLVM for machine code generation.

```
BackendConsumer::HandleTopLevelDecl
  CodeGenModule::EmitTopLevelDecl
    CodeGenModule::EmitGlobal
      CodeGenModule::EmitGlobalDefinition
        CodeGenModule::EmitGlobalFunctionDefinition
          CodeGenFunction::CodeGenFunction
          CodeGenFunction::GenerateCode
            CodeGenFunction::StartFunction
            CodeGenFunction::EmitFunctionBody
BackendConsumer::HandleTranslationUnit
  setupLLVMOptimizationRemarks
  EmitBackendOutput
    EmitAssemblyHelper::EmitAssembly
      EmitAssemblyHelper::RunOptimizationPipeline
        PassBuilder::buildPerModuleDefaultPipeline // There are other build*Pipeline alternatives
        MPM.run(*TheModule, MAM);
      EmitAssemblyHelper::RunCodegenPipeline
        EmitAssemblyHelper::AddEmitPasses
          LLVMTargetMachine::addPassesToEmitFile
        CodeGenPasses.run(*TheModule);
```

`BackendConsumer::HandleTopLevelDecl` generates LLVM IR for each top-level declaration.
This means that Clang generates a function at a time.

`BackendConsumer::HandleTranslationUnit` invokes `EmitBackendOutput` to create an LLVM IR file, an assembly file, or a relocatable object file.
`EmitBackendOutput` establishes an optimization pipeline and a machine code generation pipeline.

Now let's explore `CodeGenFunction::EmitFunctionBody`.
Generating IR for a variable declaration and a return statement involve the following functions, among others:
```
EmitFunctionBody
  EmitCompoundStmtWithoutScope
    EmitStmt
      EmitSimpleStmt
        EmitDeclStmt
          EmitDecl
            EmitVarDecl
      EmitStopPoint
      EmitReturnStmt
        EmitScalarExpr
          ScalarExprEmitter::EmitBinOps
```

After generating the LLVM IR, clangCodeGen proceeds to execute `EmitAssemblyHelper::RunOptimizationPipeline` to perform middle-end optimizations and subsequently `EmitAssemblyHelper::RunCodegenPipeline` to generate machine code.

For our integer division example, the function `foo` in the unoptimized LLVM IR looks like this (attributes are omitted):
```llvm
; Function Attrs: mustprogress noinline uwtable
define dso_local noundef i32 @_Z3fooifi(i32 noundef %a, float noundef %b, i32 noundef %c) #0 {
entry:
  %a.addr = alloca i32, align 4
  %b.addr = alloca float, align 4
  %c.addr = alloca i32, align 4
  %s = alloca i32, align 4
  store i32 %a, ptr %a.addr, align 4, !tbaa !5
  store float %b, ptr %b.addr, align 4, !tbaa !9
  store i32 %c, ptr %c.addr, align 4, !tbaa !5
  call void @llvm.lifetime.start.p0(i64 4, ptr %s) #4
  %0 = load i32, ptr %a.addr, align 4, !tbaa !5
  %1 = load float, ptr %b.addr, align 4, !tbaa !9
  %2 = load float, ptr %b.addr, align 4, !tbaa !9
  %cmp = fcmp oeq float %1, %2
  %conv = zext i1 %cmp to i32
  %add = add nsw i32 %0, %conv
  store i32 %add, ptr %s, align 4, !tbaa !5
  %3 = load i32, ptr %s, align 4, !tbaa !5
  %4 = load i32, ptr %c.addr, align 4, !tbaa !5
  %call = call noundef i32 @_Z3divIiET_S0_S0_(i32 noundef %3, i32 noundef %4)
  call void @llvm.lifetime.end.p0(i64 4, ptr %s) #4
  ret i32 %call
}
```

## Compiler middle end

`EmitAssemblyHelper::RunOptimizationPipeline` creates an LLVM pass manager to schedule the middle-end optimization pipeline.
This pass manager executes numerous optimization passes and analyses.

The option `-mllvm -print-pipeline-passes` provides insight into these passes:
```
% clang -c -O1 -mllvm -print-pipeline-passes a.c
annotation2metadata,forceattrs,declare-to-assign,inferattrs,coro-early,...
```

For our integer division example, the optimized LLVM IR looks like this:
```llvm
; Function Attrs: mustprogress nofree noinline norecurse nosync nounwind willreturn memory(none) uwtable
define dso_local noundef i32 @_Z3fooifi(i32 noundef %a, float noundef %b, i32 noundef %c) local_unnamed_addr #0 {
entry:
  %cmp = fcmp ord float %b, 0.000000e+00
  %conv = zext i1 %cmp to i32
  %add = add nsw i32 %conv, %a
  %div.i = sdiv i32 %add, %c
  ret i32 %div.i
}

; Function Attrs: mustprogress nofree norecurse nosync nounwind willreturn memory(none) uwtable
define dso_local noundef i32 @main() local_unnamed_addr #1 {
entry:
  %call = tail call noundef i32 @_Z3fooifi(i32 noundef 3, float noundef 2.000000e+00, i32 noundef 1)
  ret i32 %call
}
```

The most notaceable differences are the following

* `SROAPass` runs mem2reg and optimizes out many `AllocaInst`s.
* `InstCombinePass` (`InstCombinerImpl::visitFCmpInst`) replaces `fcmp oeq float %1, %1` with `fcmp ord float %1, 0.000000e+00`, canonicalize NaN testing to `FCmpInst::FCMP_ORD`.
* `InlinerPass` inlines the instantiated `div` function into its caller `foo`

## Compiler back end

The demarcation between the middle end and the back end may not be entirely distinct.
Within `LLVMTargetMachine::addPassesToEmitFile`, several IR passes are scheduled.
It's reasonable to consider these IR passes (everything before `addCoreISelPasses`) as part of the middle end, while the phase beginning with instruction selection can be regarded as the actual back end.

Here is an overview of `LLVMTargetMachine::addPassesToEmitFile`:

```
LLVMTargetMachine::addPassesToEmitFile
  addPassesToGenerateCode
    TargetPassConfig::addISelPasses
      TargetPassConfig::addIRPasses => X86PassConfig::addIRPasses
      TargetPassConfig::addCodeGenPrepare  # -O1 or above
      TargetPassConfig::addPassesToHandleExceptions
      TargetPassConfig::addISelPrepare
        TargetPassConfig::addPreISel => X86PassConfig::addPreISel
        addPass(createCallBrPass());
        addPass(createPrintFunctionPass(...));  # if -print-isel-input
        addPass(createVerifierPass());
      TargetPassConfig::addCoreISelPasses    # SelectionDAG or GlobalISel
    TargetPassConfig::addMachinePasses
  LLVMTargetMachine::addAsmPrinter  
  PM.add(createPrintMIRPass(Out)); // if -stop-before or -stop-after
  PM.add(createFreeMachineFunctionPass());
```

These IR and machine passes are scheduled by the legacy pass manager.
The option `-mllvm -debug-pass=Structure` provides insight into these passes:
```
clang -c -O1 a.c -mllvm -debug-pass=Structure
```

### Instruction selector

There are three instruction selectors: SelectionDAG, FastISel, and GlobalISel.
FastISel is integrated within the SelectionDAG framework.

For most targets, FastISel is the default for `clang -O0` while SelectionDAG is the default for optimized builds.
However, for most AArch64 `-O0` configurations, GlobalISel is the default.

To force using GlobalISel, we can specify `-mllvm -global-isel`.

### SelectionDAG

The official documentation <https://llvm.org/docs/WritingAnLLVMBackend.html#instruction-selector> is dry and incomplete.
_2024 LLVM Dev Mtg - A Beginners’ Guide to SelectionDAG_ has a great introduction.

```
SectionDAG: normal code path
LLVM IR =(visit)=> SDNode =(DAGCombiner,LegalizeTypes,DAGCombiner,Legalize,DAGCombiner,Select,Schedule)=> MachineInstr

SectionDAG: FastISel (fast but not optimal)
LLVM IR =(FastISel)=> MachineInstr
```

```
TargetPassConfig::addCoreISelPasses
  addInstSelector(); // add an instance of a target-specific derived class of SelectionDAGISel
  addPass(&FinalizeISelID);

SelectionDAGISel::runOnMachineFunction
  TargetMachine::resetTargetOptions
  SelectionDAGISel::SelectAllBasicBlocks
    SelectionDAGISel::SelectBasicBlock
      SelectionDAGBuilder::visit
      SelectionDAGISel::CodeGenAndEmitDAG
        CurDAG->Combine(BeforeLegalizeTypes, AA, OptLevel);
        Changed = CurDAG->LegalizeTypes(); // legalize-types
        if (Changed)
          CurDAG->Combine(AfterLegalizeTypes, AA, OptLevel);
        Changed = CurDAG->LegalizeVectors(); // legalizevectorops
        if (Changed) {
          CurDAG->LegalizeTypes();
          CurDAG->Combine(AfterLegalizeVectorOps, AA, OptLevel);
        }
        CurDAG->Legalize(); // legalizedag
          SelectionDAGLegalize::LegalizeOp
        DoInstructionSelection
          Select
            SelectCode
          PreprocessISelDAG()
        Scheduler->Run(CurDAG, FuncInfo->MBB); // pre-RA-sched
        Scheduler->EmitSchedule(FuncInfo->InsertPt); // instr-emitter
          EmitNode
            CreateMachineInstr
```

Each backend implements a derived class of `SelectionDAGISel`. For example, the X86 backend implements `X86DAGToDAGISel` and overrides `runOnMachineFunction` to set up variables like `X86Subtarget` and then invokes the base function `SelectionDAGISel::runOnMachineFunction`.
`SelectionDAGISel` creates a `SelectionDAGBuilder`.

A function is split into basic blocks and each SelectionDAG represents a single basic block.
`SelectionDAGISel::SelectBasicBlock` iterates over all IR instructions and calls `SelectionDAGBuilder::visit` on them, creating a new `SDNode` for each `Value` that becomes part of the initial DAG.

The initial DAG may contain types and operations that are not natively supported by the target.
`SelectionDAGISel::CodeGenAndEmitDAG` invokes `LegalizeTypes` and `Legalize` to convert unsupported types and operations to supported ones.

`ScheduleDAGSDNodes::EmitSchedule` emits the machine code (`MachineInstr`s) in the scheduled order.

`FinalizeISel` expands instructions marked with `MCID::UsesCustomInserter`.

`-mllvm -debug-only=isel-dump` provides insight into the instruction selection phases.

#### SelectionDAG concept

TODO

An EntryToken node represents the dependency on entering the block.
Each DAG has a root node, usually the terminator instruction.

Each `SDNode` contains many fields including:

* opcode
* flags
* 0 or more operands represented by `SDUse`
* 1 or more results represented by `SDValue`

Chain values represent non-data dependencies.

Now, let's take a closer look at our `foo` function.

#### SelectionDAG construction

For the IR instruction `%cmp = fcmp ord float %b, 0.000000e+00`, `SelectionDAGBuilder::visit` creates a new `SDNode` with the opcode `ISD::SETCC` (`t9: i1 = setcc t4, ConstantFP:f32<0.000000e+00>, seto:ch`).
```
SelectionDAGBuilder::visit
  SelectionDAGBuilder::visitFcmp
```

A new `SDNode` with the opcode `ISD::ZERO_EXTEND` is created for `%conv = zext i1 %cmp to i32`.

For the IR instruction `%add = add nsw i32 %conv, %a`, `SelectionDAGBuilder::visit` creates a new `SDNode` with the opcode `ISD::ADD`.
```
SelectionDAGBuilder::visit
  SelectionDAGBuilder::visitAdd
    SelectionDAGBuilder::visitBinary    # binary operators are handled similarly
```

Similarly, `SelectionDAGBuilder::visit` creates a new `SDNode` with the opcode `ISD::SDIV` for `%div.i = sdiv i32 %add, %c`, `SelectionDAGBuilder::visit`.
For the `ret i32 %div.i` instruction, the created `SDNode` has a target-specific opcode `X86ISD::RET_GLUE` (target-specific opcodes are legal for almost all targets).

After all instructions are visited, we get an initial DAG that looks like:
```
Initial selection DAG: %bb.0 '_Z3fooifi:entry'
SelectionDAG has 17 nodes:
  t0: ch,glue = EntryToken
            t4: f32,ch = CopyFromReg t0, Register:f32 %1
          t9: i1 = setcc t4, ConstantFP:f32<0.000000e+00>, seto:ch
        t10: i32 = zero_extend t9
        t2: i32,ch = CopyFromReg t0, Register:i32 %0
      t11: i32 = add nsw t10, t2
      t6: i32,ch = CopyFromReg t0, Register:i32 %2
    t12: i32 = sdiv t11, t6
  t15: ch,glue = CopyToReg t0, Register:i32 $eax, t12
  t16: ch = X86ISD::RET_GLUE t15, TargetConstant:i32<0>, Register:i32 $eax, t15:1
```

The `DAGCombiner` process changes `t9: i1 = setcc t4, ConstantFP:f32<0.000000e+00>, seto:ch` to `t19: i1 = setcc t4, t4, seto:ch`.
After the initial DAG combining, the output looks like:
```
Optimized lowered selection DAG: %bb.0 '_Z3fooifi:entry'
SelectionDAG has 16 nodes:
  t0: ch,glue = EntryToken
  t4: f32,ch = CopyFromReg t0, Register:f32 %1
          t19: i1 = setcc t4, t4, seto:ch
        t10: i32 = zero_extend t19
        t2: i32,ch = CopyFromReg t0, Register:i32 %0
      t11: i32 = add nsw t10, t2
      t6: i32,ch = CopyFromReg t0, Register:i32 %2
    t12: i32 = sdiv t11, t6
  t15: ch,glue = CopyToReg t0, Register:i32 $eax, t12
  t16: ch = X86ISD::RET_GLUE t15, TargetConstant:i32<0>, Register:i32 $eax, t15:1
```

#### Type legalization

The type legalizer (entry: `SelectionDAG::LegalizeTypes`) traverses nodes in a topological order.
The targets call `addRegisterClass` and `TargetLoweringBase::computeRegisterProperties` to set up legal types.
Nodes with illegal value types are transformed according to a few predefined actions:

* `TypePromoteInteger`
* `TypeExpandInteger`
* `TypeSoftenFloat`
* `TypePromoteFloat`
* `TypeSoftPromoteFloat`
* `TypeScalarizeFloat`
* ...

```cpp
EVT ResultVT = N->getValueType(i);
switch (getTypeAction(ResultVT)) {
  ...
}
```

Many actions actually call a hook `CustomLowerNode` befores generic code.
If the target specifies a Custom action handler attached on the operation with the ilegal type, the custom handler will run instead.
This allows the target-specific code to legalize an operation with its original value type.

In our example, `i1` is an illegal type and the transformed-to type is `i8`.
The type legalizer will change `t10: i32 = zero_extend t19` to `t23: i32 = any_extend t22; t25: i32 = and t23, Constant:i32<1>`.

#### Operation legalization

Targets support a subset of (Opcode, MVT) combinations (legal). The rest are illegal and need to be transformed to legal operations for the target.
Targets call `setOperationAction(Opcode, MVT, LegalizeAction)` to inform the legalizer where a (Opcode, MVT) combination is legal or requires a legalize action.

* Legal
* Promote
* Expand
* Custom

The nodes are visited in a topological order.

x86 division instructions computes both the quotient and the reminder.
`X86ISelLowering.cpp` sets the legalization operation of `ISD::SDIV` with type `i32` to `Expand`.

```cpp
setOperationAction(ISD::SDIV, VT, Expand);
```

`SelectionDAGLegalize::LegalizeOp` notices that the operation is `Expand` and calls `SelectionDAGLegalize::ExpandNode`.
The node is replaced with a new node with the opcode `ISD::SDIVREM`.

The legalization operation of `ISD::SETCC` for i8 is `Custom`.

```cpp
setOperationAction(ISD::SETCC, VT, Custom);
```

`X86TargetLowering::LowerOperation` provides custom lowering hooks and replaces the `ISD::SETCC` node with `t32: i8 = X86ISD::SETCC TargetConstant:i8<11>, t30` that uses another node `t30: i32 = X86ISD::FCMP t4, t4`.

Legalization might create new nodes. These new nodes need legalization as well. `SelectionDAG::Legalize` uses an iterative algorithm.

The result of `Legalize` looks like the following:
```
Optimized legalized selection DAG: %bb.0 '_Z3fooifi:entry'
SelectionDAG has 17 nodes:
  t0: ch,glue = EntryToken
  t4: f32,ch = CopyFromReg t0, Register:f32 %1
            t30: i32 = X86ISD::FCMP t4, t4
          t32: i8 = X86ISD::SETCC TargetConstant:i8<11>, t30
        t26: i32 = zero_extend t32
        t2: i32,ch = CopyFromReg t0, Register:i32 %0
      t11: i32 = add nsw t26, t2
      t6: i32,ch = CopyFromReg t0, Register:i32 %2
    t29: i32,i32 = sdivrem t11, t6
  t15: ch,glue = CopyToReg t0, Register:i32 $eax, t29
  t16: ch = X86ISD::RET_GLUE t15, TargetConstant:i32<0>, Register:i32 $eax, t15:1
```

#### Instruction selection

`DoInstructionSelection` visits DAG nodes in a topological order (root node first; operator selected before operands) and calls `SelectionDAGISel::Select`.
`Select` replaces most generic `SDNode`s with `MachineSDNode`s (derivied class of `SDNode` with negative `NodeType`), which will be converted to `MachineInstr`.
Some `SDNode`s (e.g. `CopyFromReg`) remain `SDNode`.

`Select` is derived by targets to perform custom logic and handle over the rest to `SelectCode`.
`SelectCode` is the entry point of TableGen-generated pattern matching code for instruction selection.
TableGen generates code that matches DAG patterns and emits machine nodes.

You can pass `-mllvm -debug-only=isel` to Clang to observe the process.
In our example,

* `X86::RET` is selected for the `X86ISD::RET_GLUE` node.
* Some nodes, such as `ISD::Register` and `ISD::CopyFromReg`, remain the same.
* `X86::IDIV32r` is selected for the `ISD::SDIVREM` node.
* `X86::ADD32rr` is selected for the `ISD::ADD` node.
* `X86::MOVZX32rr8` is selected for the `ISD::ZERO_EXTEND` node.

#### Instruction scheduling

The instruction scheduler takes DAG nodes and produces a linear sequence of machine instructions.

This scheduler during instruction selection shares code with the leagcy post register allocation scheduler, `llvm/lib/CodeGen/PostRASchedulerList.cpp`.

(There will be a machine scheduler pass immediately before register allocation and possibly another after register allocation.)
(Register allocation will introduce spilling code, destroying the schedule.)

SelectionDAG calls `CreateSchedule` to create a scheduler.
The default for many targets is a list scheduler (`ScheduleDAGRRList`).
It simulates execution of the instructionns annd tries to schedule instructions when all operands can be used without stalling the pipeline.

The list scheduler places nodes into units where glued nodes are in the same schedule unit.
Then, it builds a data dependency graph.

depth: max path down to EntryToken
height: max path down to any terminal node (e.g. RET)

The type legalizer (entry: `SelectionDAG::LegalizeTypes`) traverses nodes in a topological order.

In the `EmitSchedule` process, `MachineInstr` objects are created from these `MachineSDNode` and regular `SDNode` objects.

Note, another scheduler, machine scheduler, will run immediately before register allocation (`MachineScheduler::runOnMachineFunction`) at -O1 and above.
Some targets' schedule models (grep `PostRAScheduler`) enable a post-RA scheduler (grep `PostRASchedulerID` annd `PostMachineSchedulerID`) at -O2 and above.
The legacy post-RA scheduler (`llvm/lib/CodeGen/PostRASchedulerList.cpp`) shares code with the SelectionDAG instruction scheduler.

#### FastISel

FastISel, typically used for `clang -O0`, represents a fast path of SelectionDAG that generates less optimized machine code.

When FastISel is enabled, `SelectAllBasicBlocks` tries to skip `SelectBasicBlock` and select instructions with FastISel.
However, FastISel only handles a subset of IR instructions.
For unhandled instructions, `SelectAllBasicBlocks` falls back to `SelectBasicBlock` to handle the remaining instructions in the basic block.

### GlobalISel

[GlobalISel](https://llvm.org/docs/GlobalISel/index.html) is a new instruction selection framework that operates on the entire function, in contrast to the basic block view of SelectionDAG.
GlobalISel offers improved performance and modularity (multiple passes instead of one monolithic pass).

The design of the [generic `MachineInstr`](https://llvm.org/docs/GlobalISel/GMIR.html) replaces an intermediate representation, `SDNode`, which was used in the SelectionDAG framework.

```
LLVM IR =(IRTranslator)=> generic MachineInstr =(Legalizer)=> regular and generic MachineInstr =(RegBankSelect,GlobalInstructionSelect)=> regular MachineInstr
```

```
TargetPassConfig::addCoreISelPasses
  addIRTranslator(); // irtranslator
  addPreLegalizeMachineIR();
  addPreRegBankSelect();
  addRegBankSelect(); // regbankselect
  addPreGlobalInstructionSelect();
  addGlobalInstructionSelect(); // instruction-select
  Pass to reset the MachineFunction if the ISel failed.
  addInstSelector();
  addPass(&FinalizeISelID);
```

Similar to FastISel, GlobalISel does not handle all instructions. If GlobalISel fails to handle a function, SelectionDAG will be used as the fallback.

<!-- TargetOpcode::PRE_ISEL_GENERIC_OPCODE_START, TargetOpcode::PRE_ISEL_GENERIC_OPCODE_END -->

### Machine passes

After instruction selector, there are machine SSA optimizations, register allocation, machine late optimizations, and pre-emit passes.

```
TargetPassConfig::addMachinePasses
  TargetPassConfig::addMachineSSAOptimization  # -O1 or above
    addPass(&OptimizePHIsLegacyID);
    addPass(&StackColoringLegacyID);
    addPass(&MachineCSELegacyID);
    addPass(&LocalStackSlotAllocationID);
    addILPOpts();
      // Many targets enable EarlyIfConversion
    addPass(&PeepholeOptimizerLegacyID);
  // RISCV has createRISCVPreRAExpandPseudoPass.
  TargetPassConfig::addPreRegAlloc
  TargetPassConfig::addOptimizedRegAlloc
  TargetPassConfig::addPostRegAlloc
  addPass(createPrologEpilogInserterPass());
  TargetPassConfig::addMachineLateOptimization
  addPass(&ExpandPostRAPseudosID)
  // RISCVPostRAExpandPseudoInsts is in addPreSched2
  TargetPassConfig::addPreSched2
  // Some target replace PostRAScheduler with PostMachineScheduler.
  addPass(&PostRASchedulerID);
  TargetPassConfig::addBlockPlacement  # -O1 or above
  TargetPassConfig::addPreEmitPass
  createBasicBlockSectionsPass
  // kcfi indirect call checks.
  // RISCV has createRISCVExpandPseudoPass.
  // X86 has security hardening passes.
  TargetPassConfig::addPreEmitPass2
```

`PHIElimination` and `TwoAddressInstructionPass` precede the register allocation pass.
After `PHIElimination`, the machine function is no longer in a SSA form.

The optimization level affects the default register allocation strategy:

* `-O0`: A simpler allocator called `RegAllocFast` is used.
* `-O1` and above: A sophisticated allocator called `RegAllocGreedy` is used. It runs additional passes and utilizes `VirtRegRewriter` to finalize the allocation by replacing virtual register references with actual physical registers.

`X86::RET` describes a pseudo instruction. `X86ExpandPseudo::ExpandMI` expands the `X86::RET` `MachineInstr` to an `X86::RET64`.

### AsmPrinter

`LLVMTargetMachine::addAsmPrinter` incorporates a target-specific `AsmPrinter` (derived from `AsmPrinter`) pass into the machine code generation pipeline.
These target-specific `AsmPrinter` passes are responsible for converting `MachineInstr`s to `MCInst`s and emitting them to an `MCStreamer`.

In our x86 case, the target-specific class is named `X86AsmPrinter`.
`X86AsmPrinter::runOnMachineFunction` invokes `AsmPrinter::emitFunctionBody` to emit the function body.
The base member function handles function header/footer, comments, and instructions.
Target-specific classes override `emitInstruction` to lower `MachineInstr`s with target-specific opcodes to `MCInst`s.
Pseudo instructions that expand to multiple instructions are expanded here.

An `MCInst` object can also be created by the LLVM integrated assembler. `MCInst` is like a simplified version of `MachineInstr` with less information.
When `MCInst` is emitted to an `MCStreamer`, it results in either assembly code or bytes for a relocatable object file.

### MC

Clang has the capability to output either assembly code or an object file.
Generating an object file directly without involving an assembler is referred to as "direct object emission".

To provide a unified interface, `MCStreamer` is created to handle the emission of both assembly code and object files.
The two primary subclasses of `MCStreamer` are `MCAsmStreamer` and `MCObjectStreamer`, responsible for emitting assembly code and machine code respectively.

In the case of an assembly input file, LLVM creates an `MCAsmParser` object (LLVMMCParser) and a target-specific `MCTargetAsmParser` object.
The `MCAsmParser` is responsible for tokenizing the input, parsing assembler directives, and invoking the `MCTargetAsmParser` to parse an instruction.
Both the `MCAsmParser` and  `MCTargetAsmParser` objects can call the `MCStreamer` API to emit assembly code or machine code.

In our case, LLVMAsmPrinter calls the `MCStreamer` API to emit assembly code or machine code.

If the streamer is an `MCAsmStreamer`, the `MCInst` will be pretty-printed.
If the streamer is an `MCELFStreamer` (other object file formats are similar), `MCELFStreamer::emitInstToData` will use `${Target}MCCodeEmitter` from LLVM${Target}Desc to encode the `MCInst`, emit its byte sequence, and records needed relocations.
An `ELFObjectWriter` object is used to write the relocatable object file.

In our example, we get a relocatable object file. If we invoke `clang -S a.cc` to get the assembly, it will look like:
```asm
...
        .globl  _Z3fooifi                       # -- Begin function _Z3fooifi
        .p2align        4, 0x90
        .type   _Z3fooifi,@function
_Z3fooifi:                              # @_Z3fooifi
        .cfi_startproc
# %bb.0:
        xorl    %eax, %eax
        ucomiss %xmm0, %xmm0
        setnp   %al
        addl    %edi, %eax
        cltd
        idivl   %esi
        retq
.Lfunc_end0:
        .size   _Z3fooifi, .Lfunc_end0-_Z3fooifi
        .cfi_endproc

...
        .globl  main
        .p2align        4, 0x90
        .type   main,@function
main:                                   # @main
        .cfi_startproc
# %bb.0:
        movss   .LCPI1_0(%rip), %xmm0           # xmm0 = mem[0],zero,zero,zero
        movl    $3, %edi
        movl    $1, %esi
        jmp     _Z3fooifi                       # TAILCALL
.Lfunc_end1:
        .size   main, .Lfunc_end1-main
        .cfi_endproc
```

You may read my post [Assemblers](/blog/2023-05-08-assemblers) for more information about the LLVM integrated assembler.
