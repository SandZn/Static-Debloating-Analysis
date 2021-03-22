# paper reading

I read three papers about static debloating technology: 

Two of them are for Linux C/C++ programs. I find they are using the same approach as I mentioned in my last document:

> Generate a complete call graph for the entire modules(including the target debloating program and all its depending libraris), traverse the call graph to find all the unused functions for each module, and then eliminate these functions.

The other paper is for JavaScript programs. It uses a different approach of debloating.

These three papers are:

>1. **[Debloating Software through Piece-Wise Compilation and Loading](https://arxiv.org/pdf/1802.00759.pdf)**
>2. **[Bloat Factors and Binary Specialization](https://dl.acm.org/ft_gateway.cfm?id=3359765&amp;type=pdf)**
>3. **[Slimming Languages by Reducing Sugar: A Case for Semantics-Altering Transformations](http://cs.brown.edu/~sk/Publications/Papers/Published/lppk-slim-lang-sem-alt-trans/paper.pdf)**

In the following document, I will use *paper[1]* , *paper[2]* and *paper[3]* to refer to these three papers. Since the core idea of the first two papers are similar, so I'll elaborate them together. The rest of the document is structured as follows: **section1** for first two papers, **section2** for the last one. In each section, I'll give my conclusion of their key results, strengths and limitations first, then elaborate in detail.

## Section 1

### 1.1 Conclusion

#### 1.1.1 Key results

**paper[1]** introduces a static debloating approach in implementing an LLVM based piece-wise compiler, which can get the complete call gragh among all the modules (the program and shared libraries) and embed the call gragh information into the executable binary's optional section **during compilation**. It also implements a backward compatible piece-wise loader. When executes binary, the piece-wise loader loads all the necessary things (including executable binary and shared libraries) into the program memory image, then it uses the call graph infomation to detect the unused functions and their addresses in memory, and finally eliminates them from memory.

**paper[2]** first introduces a set of comprehensive and generic metrics called bloat factors to systematically measure code bloating in binaries, including:

> Program Bloat Factor (PBF)
>
> Library Bloat Factor (LBF)
>
> System Library Bloat Factor (SLBF)

Then it proposes a static debloating approach, which constructs the complete call gragh by analyzing the targe executable binary and its dependent shared libraries directly, using the call graph as a guidance to detect the unused functions and eliminate them from each ELF files (executable binary and shared libraries on Linux are ELF format). 

#### 1.1.2 Strengths

**both:** debloating programs based on shared libraries is much more difficult than the ones based on static libraries. Shared library doesn't link into executable binary but is loaded at runtime. So it is a challenge to get a correct call gragh for programs based on shared library. Both of the two approaches can debloat the program based on shared libraries.

**paper[1]:**  An accurate call graph is important for debloating, the more accurate it is, the better debloating effect will be. Source code has the full story, *paper[1]* analyzes all the code and generates the call graph during compilation. So it can get better code size reduction. In addition, the call graph information is retained in the optional section of ELF file, so it can keep backward compatible.

**paper[2]:**  It generates the call gragh by analyzing the **executable binary** and its dependent shared libraries directly, so this approach can be applied to debloat close-source software.

#### 1.1.3 Limitations

**paper[1]:** i) This approach can only be applied on open-source software/library; ii) The eliminating action is performed by the loader, the loader keeps the ELF file intact and just eliminates dead code in program memory image, so it needs to do all the symbol resolution and relocation work at loading stage, which makes the benefits of lazy binding technique for shared library useless and brings some load-time overhead.

**paper[2]:** i) Its elimination action is perfomed on each ELF file (binary and shared libraries), so the modified shared library file cannot be used by other programs simultaneously. I think it breaks the shared library's advantages because the shared library cannot be shared anymore. ii) Analyzing binary is much more difficult than source code, so its call graph is less accurate than analyzing on source code, which may decrease the effect of debloating.

**both:** Both of the two approaches are static debloating, even though they can get an accurate call graph and make the debloating correctly, some paths of the call graph may never be invoked at runtime. For example, one path of the call graph is under the control of a variable in a specific program configuration. If the variable is always set to 'off', then the related path is never invoked and can be eliminated. Both of the two approaches cannot address this situation.

### 1.2 Introduction

Code bloating can be aggravated when a program depends on shared libraries. That is because shared library is usually designed for one-size-fits-all, it includes multiple disjoint functionalities but only few of them would be used in one specific program. Loading all of them into the running program memory can lead to negative impact on both performance and security. These two papers introduce two static approaches to detect dead codes through call graph analyzing and remove them before program running.

#### 1.2.1 Where to get the call graph

*paper[1]* gets it by analyzing all the source code of program and its dependent libraries.

*paper[2]* gets it by analyzing the executable binary and its dependent library's object (all of them are ELF format on Linux).

So *paper[1]* can only debloat the open source programs while *paper[2]*  can debloat on both open and close source program. *paper[2]* has much more comprehensive applicability.

#### 1.2.2 When to get the call graph

*paper[1]* implements a specific compiler called **piece-wise compiler** based on LLVM, leverages the inter-modular code analysis and optimization logic present in LLVM to derive function-level dependencies both within a compilation unit and across a module **during compilation**, and embeds the call graph information into optional sections of the final executable binary. The information includes:

1. Dependency relationships between functions (i.e. the dependency graph) that comprise of functions and a list of dependencies.

2. Function-specific data that includes location and size in bytes for all the functions in the dependency graph. *(Notice: the real address in memory of each functions can only be obtained during program loading time, the location information in binary is relative relocatable and will be updated after loading)*

*paper[2]* gets call graph from the binary and shared library objects through some binary analysis techniques. First it constructs the inner call graph and derives the import-export functions list for each module, which is called **Module’s Dependence Graph(MDG) **, then connects every two MDGs using their import-export relationship after combining all the MDGs together, next we can get the entire call graph called **System's Dependence Graph(SDG)**

#### 1.2.3 Challenges of getting the call graph

Since these two papers are both about debloating C/C++ programs, they have some common challenges when trying to get correct call graph.

1. **Indirect branching/code pointers**

   code pointers can be used in C/C++ programs as below:

```
addr = &foo; addr();
```

to detect them accurately, some special handling is needed:

*paper[1]*  deals with it at LLVM IR instruction level. First piece-wise compile construct user-def chains by scanning all IR instructions, then recursively traverse the use-def chains until it encounters a referring instruction that refers a function. At that point, a dependency is recorded between the function that contains the referring instruction and the referred function.

*paper[2]*  performs code pointer analysis on ELF file and handles it at assembly code level. It scans different read-only sections in a binary to recover code pointers in places such as jump tables for switch statement, vtables, and pointer arrays. Functions in a shared library referenced by these addresses will always be included as dependencies.

2. **Dynamically loaded libraries**

Support for debloating shared libraries that are loaded dynamically (using dlopen) is another a challenge, especially for the case that the library name is generated dynamically. Both these two papers perform a trainning approach to handle it. For each program, they record all shared libraries loaded using *dlopen* at runtime as well as their functions that are invoked by *dlsym*, after that, all the dynamically loaded libraries and using functions(by *dlsym*)  and its denpendent functions can be gathered and marked as dependencies in call graph.

#### 1.2.4 How to use call graph as directive for debloating

The call graph is generated successfully at this point, but it is not sufficient enough to guide the debloating process. Because some inter module functions' dependence are determined at loading stage, for example, program P depends on function F() which is both defined in shared libraries A.so and B.so, the result of which one will be eventually invoked by P at runtime is determined by the loader, these process is knowed as symbol resolution. The two paper use different approaches to get the final call graph:

*paper[1]*  implements a specific loader called **piece-wise loader** except for doing what a normal loader usually do, piece-wise loader also update the call graph information in executable binary optional session which was embeded by piece-wise compile at compile stage. Piece-wise loader first loads all the dependent shared libraries into program memory image, performs necessary works, such as symbol resolution and relocation, after that, all functions have their certain address in memory and piece-wise use these address to update the function location information in call graph. Then piece-wise loader walks through the call graph to get all the unused functions list and their real address. 

*paper[2]* prefers to get the call graph statically, so all the works need to be done before loading stage. It's more complex than *paper[1].* Recall the method to generate SDG I mentioned before: 

> every two MDGs are connected by their import-export functions list

To determine the final functions dependence relationship, *paper[2]* needs to replicate how the loader performs symbol binding at runtime here, it's an essential and complex work. After doing that, we can get the call graph and the location of each unused functions in every ELF file (executable binary and shared libraries).   

#### 1.2.5 How to perform debloating

At this point, the final unused functions list are generated, *paper[1]* has all the unused function's addresses resided in program's memory image while *paper[2]* has the addresses resided in each ELF file. Then, 

*paper[1]* overwrites all unused functions in program memory image with byte 0x6d, which is a reserved instruction that raises an *‘Illegal Instruction’* exception.

*paper[2]* overwrites each unused function in their own ELF file with byte 0x6d. After that, when execute the executable binary, the loader would load all the necessary codes into program memory image, keep the rest parts of memory with 0x6d.

we can see that *paper[1]* keeps all the ELF files intact and only eliminate unused functions in program memory image, so all the shared libraries can be loaded and mapped by other programs at the same time, and different eliminating parts on same shared libraies memory image by two different running programs will bring some memory overhead due to COW(Copy on write). 

While *paper[2]* overwrites the executable binary as well as shared libraries files directly, so the modified shared library file cannot be loaded by other programs simultaneously. I think that it breaks the shared library's advantages and makes it like a static library.

#### 1.2.6 Evaluate

1. **correctness:** *paper[1]* uses the piece-wise complied musl-libc and test coreutils to evaluate correctness and performance. All of the programs (109 in total) in coreutils passed the coreutils test suite that is packaged with coreutils source code without errors. *paper[2]* runs several real-world programs (like firefox, vlc...) against large workload and doesn't observe any failures. After reading some papers about debloating software, I find the common correctness experiments for debloating is to 1) debloat some famous software using the approach introduced by the paper; 2) run the target software's test cases and real-world workload, if the testing is passed without errors, the correctness of the debloating approach is proved.

2. **code size of reduction:  ** The two approaches can debloat COTS binaries successfully. They both pick `vlc` as one of their target software, and evaluate the reduction for libc. *paper[1]* gets 82% of reductions while *paper[2]* only gets 13.88%. I think the reason is that *paper[1]*'s code analysis is based on LLVM IR instruction while *paper[2]* is based on binary, and LLVM IR is much closer to sourse code and has more detailed information can be used for call graph construction.

## Section 2

### 2.1 Conclution

#### 2.1.1 Key results

*paper[3]* introduces a static debloating approach in the aspect of semantics. It analysis the semantic bloat in a type of JavaScript desugared language λS5, demonstrates how bloat the code will be after desugaring a rich featured scripting language into λS5. It assumes that if we put some restrictions on the use of the language, we can greatly slimming its desugared language. Along with these assumptions, it defines some semantic-altering transformations, shows how to use them to perform semantic debloating and experimentally demonstrates their effectiveness. 

#### 2.1.2 Strengths

Unlike the traditional debloating approachs, *paper[3]* makes aggressive assumptions of altering language semantic. This provides a good guide for future language design, and leads us to think whether scripting languages really need so much rich semantics.

#### 2.1.3 Limitations

This approach gets codes reduction at the cost of altering the semantics, even though its experiment shows that debloating by altering semantics is correct at most of time, but the incomplete correctness makes it have limited applicability in real world.

### 2.2 Introduction

JavaScript has a large quantity of implicit and overloaded behavior. When it is handled by browsers, it should first be splitted to a core language (namely 'user code'), then be shipped with an environment that implements built-in JavaScript functions, finally become λS5 (a semantics of the strick-mode of ECMAScript5 language, running in browsers, namely 'environment code'). We call this process a "desugaring" process, which means, the end goal of desugaring is to produce an λS5 program from JavaScript source. The output of desugarring (i.e. λS5) can be handled directedly by tools, their behaviors are explicit and much simpler. Due to the complexity and the rich semantics of JS,  λS5 has to implement all of the possibilities of the semantic, so even tiny JS fragments can produce large λS5 output which brings semantic bloating. *paper[3]* proposes an approach to resolve this semantic bloating.

#### 2.2.1 Traditional semantic debloating

Traditional optimizations applied to most languages are semantics-preserved. *paper[3]* lists eight types of semantics-preserving transformations, these are: i) Assignment Conversion, ii) Constant Propagation, iii) Constant Folding, iv) Dead Code Elimination, v) Non-constant Propagation, vi) Assertion Elimination, vii) Function Inlining, viii) Environment Cleaning. After measuring shrinkage and examining these transformations' effectiveness using ECMAScript test suite (ECMAScript test262) on both test suite and real-world libraries, the figures show that semantics-preserving transformations have a large impact on environment code (38.2% of test suite and 20% of libraries), while just a little impact on user code (9.6% of test suite and 6.8% of libraries). However, it is user code that reﬂects the original JavaScript program rather than environment code, we should concern more about user code. In terms of user code, *paper[3]* proposes a more aggressive method.

#### 2.2.2 Semantics-altering debloating

*paper[3]* describes five semantics-altering transformations. It purposefully simplifies the semantics of JavaScript by weakening particular language features. After that, the complex JavaScript semantic becomes simpler, as a result, the corresponding λS5 code debloated. It is useful for practical purpose of managing λS5 more easily, and showcasing the great cost of some rich JavaScript features. These transformations are: 

+ **Fixing Function Arity**. In JS, a function can be called with parameters less or more than when the function is declared. The desugarred output is very complex to support variable arity. To fix function arity, this transformation must be applied only when functions are called with exactly as many parameters as their headers declare. The argument keyword can not be used. Further more, this transformation can not be applied to getter and setter attributes.

+ **Function Restoration**. In JS, a function is an object as well as a constructor that can be used to created objects. This transformation removes the "constructor" action and restores a function object to a pure function.
+ **Simplifying Arithmetic Operations**. In JS, arithmetic operators have complicated semantics. This transformation simpliﬁes the semantics by making strong assumptions about how operators are used: Additive and Relational operators must be applied to arguments of the same type (either Number or String), and Bitwise Shift and Multiplicative must be applied to Numbers only.
+ **Identiﬁer Restoration**. In JS, a variable can be accessed dynamically through the context object `this`. For example, `var n0 = 2`,  you can visit `n0` by `this['n' + 0]`. After this transformation, you cannot delete or enumerate properties in `this` and cannot access properties of the context object by computing names.
+ **Unsafe Assertion Elimination**. JS has a plenty of implicit type conversions and type checks which lead to large λS5 outputs. This transformation removes these type conversions and type checks and assumes the running code is correct and does not need these checks.

#### 2.2.3 Evaluation

The evaluation of semantic-altering debloating method is same with that in semantics-preserving transformations. It can be seen that semantics-altering transformations dramatically reduce user code size: 57.5% of test suite and 52.4% of libraries. Since it is a semantic-altering method, it cannot guarantee a large-scale of code correctness as expected. Nevertheless, the code still passes over 70% of tests after using just one or a combination of these transformations.

With this cogent evidence, we can assert that some language features in JavaScript desugar into much more code than necessary, and involve a great deal of semantic bloating. However, in the actual use of scripting languages, this optimization can not really be used, because it can not guarantee the overall correctness.



# experiment protocal

In this experiment, we are going to perform slimming JS applications through a tool (named *JSSlimmer*, by means of static analysis), and validate the effectiveness of this tool. 

We focus on two research questions:

RQ1: How to validate the correctness of each application after using *JSSlimmer*?

RQ2: How much an application can be reduced after slimmed by *JSSlimmer*?

In order to evaluate  *JSSlimmer's* ability at debloating programs, we select a group of prevalent JS applications from GitHub as the target programs. Each application must i) be open-source and well implemented,  ii) have a test suite with a test coverage greater than 85%, since we are going to use a well-performed test suite to validate the tool, and iii) depend on other libraries, that is because, as the prerequisite, we know *JSSlimmer* takes effect by removing unused JS functions from other depending libraries.

To answer RQ1, we validate this tool through both static and dynamic method for each application.

The static evaluation is as follows:

1. Obtain all the functions defined by the program itself. We can traverse all JS files in the direction that implements these functions. For each JS file, we analyze its corresponding AST tree using the method mentioned in my last document, identify each function by its file name and location information, and collect them to a temporary file. Then we can use the identified information to recognize each function and get a function list.
2. Generate the call graph for the bundle file of the application, namely bundle_callgraph.
3. Taking each function obtained in step1 as a starting point, we traverse bundle_callgraph to get a set of call chains. (We expect that before and after debloating, all functions and their calling relationships are consistent in this set.)
4. Perform *JSSlimmer* on the bundle file and generate an optimized file.
5. Generate the call graph for the optimized file, namely optimized_callgraph.
6. Compare the set of call chains we generated in step3 with the optimized_callgraph, if they are exactly the same, this static  evaluation passes.

The dynamic evaluation is as follows: 

1. Run test cases provided by the program. Make sure that all test units pass the test before we use *JSSlimmer*. 
2. Act test suite on the bundle file of the application namely bundle.js. There probably be some issues in this step, since different program has different test suite, not every test suite can perform directly on its bundle file. In some cases, we need to modify the entry of the test suite pointing to its bundle file. 
3. Perform *JSSlimmer* on bundle.js and generate an optimized file namely bundle_optimized.js. 
4. Modify the test entry to bundle_optimized.js and run test again. If the program can still pass all tests, we can assert this appoach does not affect the behavior of the application, this dynamic evaluation passes.

To answer RQ2, we compare the original bundle file with the optimized file, and calculate the reduction from 3 dimensions: i) File size, ii) Code line, iii) Loading time. With these results, we can answer RQ2.

