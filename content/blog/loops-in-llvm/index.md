---
title: "Loops in LLVM"
date: 2022-11-20
draft: false
tags: [llvm, compiler]
categories: []
---

Loops are quite important in basicly every high level programming languages, it provides the concept to execute a piece of codes repeatedly to desired number of times. Incidentally, loops are often the most computationally intensive parts of the program. For example, at the heart of deep learning applications are matrix multiplications and most basic 2d matrix multiplication can be expressed as a 3 nested for-loops.

```cpp
// A[m][k]
// B[k][n]
// C[m][n]
void matmul_2d(float *A, float *B, float *C, 
                   int m, int n, int k) {
    for(int i = 0; i < m; i++) {
        for(int j = 0; j < n; j++) {
            C[i][j] = 0;
            for(int h = 0; h < k; h++) {
                C[i][j] += A[i][h] * B[h][j];
            }
        }
    }
}
```

However, programmer usually do not write loop codes as optimal as it can be, so loops require all kinds of optimizations by the compiler or even hand tuned by experts in order to achieve better performance. These optimizations often results in less redundant execution (LICM), better pipline utilization of the processor (loop unrolling), better utilization of SIMD instructions (vectorization), better utilization of processor cores (multithreading), better data locality (tiling/loop reordering), and much more.

This blog will record my encounters with how loops are represented, analzyed and optimized in LLVM.

## Loop Terminology

#### Natural Loops

In compilers, a natural loop is a set of nodes from the control-flow graph such that:
- exist a `header` dominates all nodes in the loop
- no other edges can enter the loop nodes except `header` 
- nodes are strongly connected, contains a back edge to the `header`

{{< mermaid >}}
graph TD;
A[Entry]-->H[h: Header];
H-->B{b: Body};
B-->L[l: Latch];
L-->H;
B-->E[e: Exit];
subgraph Natural Loop
H
B
L
end
{{< /mermaid >}}

In this example, CFG exist a back edge `l -> h` whose destination dominates its source (`h dom l`) and a single entry point `h`, a header dominates all nodes in the loop.  
The natural loop of the back edge is `h + {nodes that can reach l without going through h}`.  
Thus the natural loop is the set `[h, b, l]`.

Natural loops are the most common loops in programs, and compiler can assume certain inital conditions to hold at beigning of each iteration through the loop [Compilers Principles Techniques and Tools:665].

#### LLVM Loop Terminology

In LLVM, natural loop is just called loop and more generalized definition is called a cycle. Some terminology for LLVM Loop:

- **entering block:** a non-loop node that has an edge into the loop (the header)  
- **preheader:** if only one entering block and its only edge is to the header  
- **header:** single entry point of the loop  
- **latch:** a loop node that has an edge to the header  
- **backedge:** edge from a latch to the header  
- **exiting edge:** edge from inside the loop to a node outside of the loop  
- **exiting block:** source of exiting edge  
- **exit block:** target of exiting edge  

More on:
- https://llvm.org/docs/LoopTerminology.html#loop-terminology

## Loop Representation in LLVM

#### Simple Loop CFG

```cpp
void test(int n) {
  for (int i = 0; i < n; i += 1) {
    // Loop body
  }
}
```

This is a baisc loop in c. Cas use these command to compile to llvm IR and visualize CFG of the loop.

```bash
clang -emit-llvm -S -O0 test.cpp -o test.ll
opt -passes=mem2reg,dot-cfg test.ll -S -o test1.ll
dot -Tpng ._Z4testi.dot > loop_basic.png
```

```llvm
define dso_local void @_Z4testi(i32 noundef %0) #0 {
; preheader
  br label %2

; header
2:                                                ; preds = %6, %1
  %.0 = phi i32 [ 0, %1 ], [ %7, %6 ]
  %3 = icmp slt i32 %.0, %0
  br i1 %3, label %5, label %4

; exit
4:                                                ; preds = %2
  br label %8

; body
5:                                                ; preds = %2
  br label %6

; latch
6:                                                ; preds = %5
  %7 = add nsw i32 %.0, 1
  br label %2, !llvm.loop !3

; return
8:                                                ; preds = %4
  ret void
}

```

{{< figure
    src="loop_basic.png"
    caption="Basic Loop"
    class="center"
    >}}

#### Loop Simplify Form

LLVM use `-loop-simplify` pass to transform loop to a canonical form with:

- one preheader
- one backedge (one latch)
- header dominates all exit blocks (no edge to exit blocks from outside the loop)

#### Rotated Loops

LLVM use `-loop-rotate` pass to transform loop to a do-while canonical form. But if the loop is not guaranteed to excute at least once, a `guard` will be create to test inital condition.   
LoopRotate ensures that the loop is in Loop Simplify Form after rotation. The reason for rotation is this form will become easier for transform passes (e.g. LICM) to hoisting instructions into the preheader.  
Also generated code can have one less uncoditional jump: [Why are loops always compiled into "do...while" style (tail jump)?](https://stackoverflow.com/questions/47783926/why-are-loops-always-compiled-into-do-while-style-tail-jump)

```bash
opt -passes=mem2reg,loop-rotate,dot-cfg test.ll -S -o test1.ll
```

{{< figure
    src="loop_rotate.png"
    caption="Rotated Loop"
    class="center"
    >}}

#### Loop Closed SSA

LLVM use `-lcssa` pass to transforms loops by placing phi nodes at the end of the loops for all values that are live across the loop boundary. This will make other loop optimizations simpler. 

```c
// Before
for (...)
    if (c)
        X1 = ...
    else
        X2 = ...
    X3 = phi(X1, X2)
... = X3 + 4

// After
for (...)
    if (c)
        X1 = ...
    else
        X2 = ...
    X3 = phi(X1, X2)
X4 = phi(X3)
... = X4 + 4
```
> From [Loop-Closed SSA Form Pass](https://llvm.org/docs/Passes.html#lcssa-loop-closed-ssa-form-pass). 

## Loop Analysis in LLVM

#### LoopInfo

LoopInfo is analysis pass in LLVM to get information about loops. Provides APIs for simple analysis of the loop:
- getExitingBlocks/getExitingBlock: get all blocks inside loop have edge to outside
- getExitBlocks/getExitBlock: get all blocks out loop have edge from inside loop
- getExitEdges: get all edge (inside, outside)
- getLoopLatches/getLoopLatch: get all loop latch blocks inside loop
- getLoopPreheader: get the preheader of the loop
- getLoopPredecessor: get the single predecessor of the loop, if predecessor only have one outgoing edge, it is also the preheader
- getInnerLoopsInPreorder: get all inner loops inside the loop nest in preorder
- getLoopsInPreorder: get all inner loops inside the loop nest including the loop in preorder

> `getBlock` will return the block if only exist one block, otherwise return null

#### LoopInfoImpl

How LoopInfo impemented in `LoopInfoImpl.h`:

```c
// LoopInfoBase::analyze
for each [node] in post order dominator tree:
    backedges = []
    for each [predecessor] in node:
            if (node dominates predecessor) 
              and (entry can reach predecessor):
                add edge [predecessor, node] to backedges
    if (backedges not empty):
        create new loop [L]
        // discoverAndMapSubloop
        for each [block] in backward CFG list:
            if (block not discovered):
                map block to current loop L
                add predecessors of block to CFG list
            else:
                get block outermost loop [L_o]
                if (L_o is not L):
                    set L as parent of L_o
                    add predecessors of L_o header to CFG list
for each [block] in DFS post order CFG:
    // PopulateLoopsDFS
    get inner most loop [L] that block lives
    if (L exist) and (L header is block):
        if (L is outer most loop):
            mark L as top level loop 
        else:
            add L to sub loops of parent loop
    add block to L and outer loops of L when exist
```
