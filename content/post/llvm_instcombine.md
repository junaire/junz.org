---
title: "LLVM 中端优化之 InstCombine"
author: "Jun"
date: 2023-07-18T16:45:13+08:00
---

InstCombine 是 LLVM 中的一个窥孔式的优化，为 LLVM 在 canonicalize IR 过程中的一个重要组成部分。这个 Pass 的主要目的就是尽可能地根据数学规则合并优化 IR，同时将 IR 归一化。

## 代码实现
整个算法是一个基于 Worklist 的一个算法，首先会将一个 `llvm::Function` 中的所有指令放到一个工作集中，然后不停从 worklist 中取一个指令使用访问者模式处理，直到 worklist 为空，或达到最大迭代次数（说明优化器卡死了）。

InstCombine 关于填充 worklist 的代码 (https://github.com/llvm/llvm-project/blob/a5cee3e386bde28ce21ff2ead3fc420f018604ca/llvm/lib/Transforms/InstCombine/InstructionCombining.cpp#L4035)：
```c++
/// Populate the IC worklist from a function, by walking it in depth-first
/// order and adding all reachable code to the worklist.
///
/// This has a couple of tricks to make the code faster and more powerful.  In
/// particular, we constant fold and DCE instructions as we go, to avoid adding
/// them to the worklist (this significantly speeds up instcombine on code where
/// many instructions are dead or constant).  Additionally, if we find a branch
/// whose condition is a known constant, we only visit the reachable successors.
static bool prepareICWorklistFromFunction(Function &F, const DataLayout &DL,
                                          const TargetLibraryInfo *TLI,
                                          InstructionWorklist &ICWorklist) {
  bool MadeIRChange = false;
  SmallPtrSet<BasicBlock *, 32> Visited;
  SmallVector<BasicBlock*, 256> Worklist;
  Worklist.push_back(&F.front());

  SmallVector<Instruction *, 128> InstrsForInstructionWorklist;
  DenseMap<Constant *, Constant *> FoldedConstants;
  AliasScopeTracker SeenAliasScopes;

  do {
    BasicBlock *BB = Worklist.pop_back_val();

    // We have now visited this block!  If we've already been here, ignore it.
    if (!Visited.insert(BB).second)
      continue;

    for (Instruction &Inst : llvm::make_early_inc_range(*BB)) {
      // 常量折叠指令，如果可能。
    }
    // 如果 terminator 是分支指令，看看有没有明显不可达的分支，若有则跳过。
  } while (!Worklist.empty());

  // 删掉所有不可达的基本块。

  // Once we've found all of the instructions to add to instcombine's worklist,
  // add them in reverse order.  This way instcombine will visit from the top
  // of the function down.  This jives well with the way that it adds all uses
  // of instructions to the worklist after doing a transformation, thus avoiding
  // some N^2 behavior in pathological cases.
  ICWorklist.reserve(InstrsForInstructionWorklist.size());
  for (Instruction *Inst : reverse(InstrsForInstructionWorklist)) {
    // 删掉死代码。
    ICWorklist.push(Inst);
  }

  return MadeIRChange;
}
```
主要核心逻辑 (https://github.com/llvm/llvm-project/blob/a5cee3e386bde28ce21ff2ead3fc420f018604ca/llvm/lib/Transforms/InstCombine/InstructionCombining.cpp#L3793)：
```c++
bool InstCombinerImpl::run() {
  while (!Worklist.isEmpty()) {
    if (Instruction *Result = visit(*I)) {
      ++NumCombined;
      // Should we replace the old instruction with a new one?
      if (Result != I) {
        // Everything uses the new instruction now.
        I->replaceAllUsesWith(Result);

        // Move the name to the new instruction first.
        Result->takeName(I);

        // Insert the new instruction into the basic block...
        BasicBlock *InstParent = I->getParent();
        BasicBlock::iterator InsertPos = I->getIterator();
        Result->insertInto(InstParent, InsertPos);

        // Push the new instruction and any users onto the worklist.
        Worklist.pushUsersToWorkList(*Result);
        Worklist.push(Result);

        eraseInstFromFunction(*I);
      } else {
        // If the instruction was modified, it's possible that it is now dead.
        // if so, remove it.
        if (isInstructionTriviallyDead(I, &TLI)) {
          eraseInstFromFunction(*I);
        } else {
          Worklist.pushUsersToWorkList(*I);
          Worklist.push(I);
        }
      }
      MadeIRChange = true;
    }
  }
  return MadeIRChange;
}
```

而处理各类 Instruction 的代码被放在 `llvm/lib/Transforms/InstCombine` 里，有以下等文件：

* InstCombineAddSub.cpp 处理 `add`, `sub`, `fadd`, `fsub` 指令。
* InstCombineAndOrXor.cpp 处理 `and`, `or`, `xor` 指令。
* InstCombineAtomicRMW.cpp 处理 `atomicrmw` 指令。
* InstCombineCalls.cpp 处理 `call` 指令。
* InstCombineCasts.cpp 处理如 `zext`, `sext` 等类型转换指令。
* InstCombineCompares.cpp 处理 `icmp` 和 `fcmp` 指令。
* InstCombineLoadStoreAlloca.cpp 处理 `load`，`store`，`alloc` 指令。
* InstCombineMulDivRem.cpp 处理 `mul`，`div`，`rem` 等指令及变种。
* InstCombinePHI.cpp 处理 `phi` 节点。
* InstCombineSelect.cpp 处理 `select` 指令。
* InstCombineShifts.cpp 处理 `shl`, `lshr`, `ashr` 等移位指令。 
* InstCombineVectorOps.cpp 处理和向量操作有关的操作。

## Pattern match

InstCombine 是一个启发式的优化 Pass，其中定义了海量优化规则，当匹配到这些模式时，就可以对相应的代码进行简化（fold）。而这里的模式匹配使用到了 LLVM 中的另一基础设施 -- PatternMatch。通过它我们可以使用 C++ 添加新的优化规则，并且非常容易理解和阅读。事实上，以上一系列实现文件包含的就是各类匹配规则。

举一个简单的例子(https://github.com/llvm/llvm-project/blob/e0ac46e69d7adbe327148550ffafe746cbc0ec78/llvm/lib/Transforms/InstCombine/InstCombineAddSub.cpp#L1412-L1414)：

```c++
  // A + -B  -->  A - B
  if (match(RHS, m_Neg(m_Value(B))))
    return BinaryOperator::CreateSub(LHS, B);
```

`match` 函数接受一个自定义的 pattern，如果满足则返回真。而以 `m_` 开头的函数则是 LLVM 定义的一系列原语，对应着各种基本操作。这里的规则表示 LLVM 会将 `A + -B` fold 为 `A - B`。可以看到如果我们匹配上了这一条规则我们将创建一个新的 `sub` 指令并将其返回出去。结合上面 `InstCombine` 的主要逻辑可知这个就是 `visit` 返回的 `Instruction` 了，LLVM 会将新生成的指令继续放到 worklist 中迭代。

LLVM 定义的所有原语在这 (https://github.com/llvm/llvm-project/blob/main/llvm/include/llvm/IR/PatternMatch.h) 大部分的原语都比较好理解，这里简单介绍下一些比较特殊的：

* `m_Value` 这个原语非常重要并且被广泛使用，它会匹配一个 `llvm::Value` 并且捕获这个 `Value`。

举个简单例子，我们想匹配一个 `add` 指令并 dump 两个操作数。
```llvm
%add = add nsw i32 %a, %b
```
我们可以编写下面规则：
```c++
Value *X, *Y;
if (match(m_Add(m_Value(X), m_Value(Y)))) {
    X->dump(); // %a
    Y->dump(); // %b
}
```

另外注意 `m_Value` 也有一个参数为空的重载，表示匹配任意一个 `llvm::Value` 但是不捕获。

* `m_Specific` 此原语用来表示某一特定的值。
* `m_Deferred` 此原语也用来表示某一特定的值，但被用在同一 `match` 下。

这个地方是一个很容易混淆的地方，我举个简单的例子：

如果你要匹配 `(A + B) * (A + B)`，应该这么写：
```c++
Value *A, *B;
match(m_Mul(m_Add(m_Valule(A), m_Value(B)), m_Add(m_Specific(A), m_Specific(B))))
```

但是如果你要匹配 `A + A`，就必须使用 `m_Deferred`，因为它们在同一 `match` 下，即：
```
Value* A;
match(m_Add(m_Value(A), m_Deferred(A)));
```

* `m_OneUse` 此原语表示这条指令只在当前被使用了，这很重要因为其它地方也可能用到这个表达式的结果，这样删掉它就是不安全的操作了。比如[这里](https://github.com/llvm/llvm-project/blob/9c2f792dceb626f57c577b02468562e1e84822f1/llvm/lib/Transforms/InstCombine/InstCombineAddSub.cpp#L1431-L1433)。

```c++
  // LHS + (A + LHS) --> A + (LHS << 1)
  if (match(RHS, m_OneUse(m_c_Add(m_Value(A), m_Specific(LHS)))))
    return BinaryOperator::CreateAdd(A, Builder.CreateShl(LHS, 1, "reass.add"));
```
* `m_Intrinsic` 除了算术基本操作， LLVM 中也提供了匹配 intrinsic 的原语，比如 `m_Intrinsic<Intrinsic::ctlz>()` 可以匹配 `llvm.ctlz` 这个 intrinsic 函数调用。另外对于一些非常常用的 intrinsic，LLVM 也提供了一些 helper 原语，比如 `m_SMax` 和 `m_SMin`，分别匹配 `llvm.smax` 和 `llvm.smin`。
* `m_c_*` 一般的匹配原语是有序的，即 `A + -B` 和 `-B + A` 是不同的。如果你的规则符合交换律，请使用前缀为 `m_c_*` 的原语。

## Instruction Simplify

InstCombine 在 LLVM 中端优化中是一个有意思的 Pass，基本上只要是没办法在其它特定 Pass 做的事情都可以往这里塞。它规则的编写一般都比较简单，难的是你的规则放在**哪里**。虽然严格意义上都塞在每个对应的 `visit` 方法里就行，但为了可维护性一般不会这么做。各种详细规则都会以 pattern 归类，并放在特定的 helper 函数中。比如 [foldAddWithConstant](https://github.com/llvm/llvm-project/blob/9c2f792dceb626f57c577b02468562e1e84822f1/llvm/lib/Transforms/InstCombine/InstCombineAddSub.cpp#L1380-L1381) 里就专门放形式为 `add X, 常量` 的 pattern。

另外值得注意的是，每个 `visit` 方法在最开始一般都会调用 `simplify*Inst` 函数。它被定义在 [InstructionSimplify.cpp](https://github.com/llvm/llvm-project/blob/main/llvm/lib/Analysis/InstructionSimplify.cpp) 中。`InstructionSimplify` 并不是一个独立的 Pass，而是被其它 Pass 使用，比如这里的 `InstCombine`。**它的作用同样也是合并简化指令，但是它的不同之处在于它不会增加新的指令。**

举个例子，以下模式应该被放在 `InstructionSimplify` 中而不是 `InstCombine`：
```c++
x = a + 1;
y = x + 1;

==>
y = a + 2; // 没有新增指令。
```

除了这一区别，`InstructionSimplify` 和 `InstCombine` 的实现几乎没有任何不同，这里不赘述。

> 所以特别注意，如果你想新加一个优化规则，先确定它会不会新增加指令，如果不会的话请把规则放在 InstructionSimplify.cpp 中，否则放在对应的 InstCombine*.cpp 中。

## 贡献指南

向 `InstCombine` 中贡献代码可参照以下步骤：
1. 寻找可能的新规则。可以多翻翻 Github issues，它有对应的 [label](https://github.com/llvm/llvm-project/issues?q=is%3Aopen+is%3Aissue+label%3Allvm%3Ainstcombine)。但是这个 label 仅供参考，因为很多人乱加，一些标着特定 target 的 missed optimization 其实也属于中端优化的问题。你可以在 godbolt 上多开几个 target 看看它们是不是都有这个问题，如果是一般说明在中端就没有优化完全。

2. 接着要确定优化的规则。godbolt 里可以直接查看 LLVM IR，你可以观察它是否存在可能的简化模式，并使用 [alive2](https://alive2.llvm.org/ce/) 进行验证。Alive2 是一个用于检测 IR transformation 是否正确的工具，用法一般来说很简单，编写两个函数：`src` 为优化前的代码，`tgt` 为优化后的代码，如果优化正确 alive2 就会提示 `Transformation seems to be correct!`。这个 alive2 的链接一般要在 code review 时加上，这样你的 reviewer 可以很容易地看到你的 `fold` 是什么，以及是否正确。

3. 编写测试。除了编写规则的代码，我们也需要在 `llvm/test/Transforms/InstCombine` 中编写测试 IR。据我的经验 LLVM 的 reviewer 一般喜欢先 `precommit` 测试，然后再编写新的 pattern，这样可以很清楚地在 diff 中看到 IR 的改变。由于 LLVM 用于 review 的 `arcanist` 工具非常坑，每次只能上传最近的一个 `commit`，所以这里简要说下我个人喜欢的 `workflow`：

```bash
$ # 在 llvm/test/Transforms/InstCombine 中新增测试。
$ # 更新 check line。
$ python3 llvm/utils/update_test_checks.py --tool-binary=/path-to-your-build-dir/bin/opt 修改的测试文件
$ git commit # 这个就是 precommit test。
$ arc diff # 提交到 Phabricator
$ # 编写新规则
$ ninja # 重新编译
$ # 重新生成 check line，可以在 diff 中清楚看到我们的规则给 IR 带来的改变。
$ python3 llvm/utils/update_test_checks.py --tool-binary=/path-to-your-build-dir/bin/opt 修改的测试文件
$ git commit # 我们的主 patch。
```

举一个我之前写的一个优化为例：

1. [precommit test](https://github.com/llvm/llvm-project/commit/2b9330e41f5405328e28d4b34bb69461fbe227b5)
2. [main patch](https://github.com/llvm/llvm-project/commit/e3175f7f1b55a38e9ab845ca26f0c4343ee2f7ff)

最后，happy hacking :^)