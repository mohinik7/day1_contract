

# **An LLVM-Based Obfuscation Framework: Architectural Design and Implementation Plan**

## **I. Executive Summary & Strategic Vision**

### **Overview**

This document presents a comprehensive architectural design and implementation plan for the development of a modular, powerful, and configurable software obfuscation tool. The core of this solution is built upon the Low-Level Virtual Machine (LLVM) compiler infrastructure, a modern and highly extensible framework that provides the ideal foundation for sophisticated code analysis and transformation.1 The primary objective is to create an application that accepts C and C++ source code, applies a layered set of obfuscation techniques at the compiler's intermediate representation (IR) level, and generates highly resilient, difficult-to-reverse-engineer binaries for both Windows and Linux platforms. This initiative directly addresses the critical need for intellectual property protection, anti-piracy measures, and the mitigation of reverse engineering threats in the cybersecurity landscape.3

### **Value Proposition for the Hackathon**

The proposed solution transcends the creation of a simple utility; it represents the development of a complete obfuscation framework. This approach demonstrates a profound understanding of modern compiler architecture, the intricacies of language-agnostic code transformation, and advanced software protection paradigms. The project's competitive edge in the Smart India Hackathon will be established through several key differentiators: a foundation built on a modern, up-to-date version of LLVM, a suite of potent and configurable obfuscation passes, and the strategic inclusion of state-of-the-art techniques that move beyond trivial obfuscation methods. The final deliverable will not only meet all requirements of the problem statement but will also serve as a platform for further research and development in software security.

### **Key Features Synopsis**

The obfuscation tool will possess the following core capabilities:

* **Configurable, Multi-Pass Obfuscation Pipeline**: Users can select and configure a sequence of obfuscation passes, allowing for a tailored balance between security, performance, and binary size.  
* **C/C++ Source Code Support**: Leverages the Clang frontend to seamlessly process standard C and C++ codebases.4  
* **Cross-Platform Binary Generation**: Natively supports the generation of executables for both Windows (PE) and Linux (ELF) from a single development environment.  
* **Automated, Metric-Driven Report Generation**: Produces a detailed report with each run, logging all input parameters and providing quantitative metrics on the obfuscation applied, such as the volume of bogus code inserted and the number of strings encrypted.  
* **Innovative Obfuscation Techniques**: Implements advanced methods including Control Flow Flattening (CFF) and provides a proof-of-concept for Virtualization-Based Obfuscation (VBO), demonstrating a capacity for cutting-edge security engineering.

---

## **II. Architectural Blueprint for the Obfuscation Engine**

The architectural design is the bedrock of this project. The decisions outlined in this section are paramount for ensuring the tool's stability, extensibility, and effectiveness. A well-considered architecture will streamline development during the time-constrained hackathon environment and result in a more robust and professional final product.

### **2.1. Foundation: Selecting and Building the LLVM Toolchain**

#### **The LLVM Ecosystem**

The LLVM project is not a single compiler but a collection of modular and reusable compiler and toolchain technologies.2 Its architecture is famously defined by a three-phase design: the frontend, the middle-end (optimizer), and the backend.5 The frontend (e.g., Clang for C/C++) parses source code and translates it into a language-independent Intermediate Representation (IR).6 The middle-end operates exclusively on this IR, performing a series of analyses and transformations, known as "passes".7 Finally, the backend translates the processed IR into machine code for a specific target architecture.5

The LLVM IR is the cornerstone of this architecture and the reason LLVM is the ideal choice for this project. It is a well-defined, portable, high-level assembly language that is designed to be a universal medium for optimization and transformation.1 By implementing our obfuscation logic as middle-end passes that manipulate the LLVM IR, our tool becomes inherently language-agnostic (supporting any language with an LLVM frontend) and target-agnostic (supporting any architecture with an LLVM backend).10

#### **Strategic Decision \- Modern LLVM as a Plugin Host**

A critical analysis of the existing landscape of LLVM-based obfuscators reveals a significant architectural choice. The seminal Obfuscator-LLVM (O-LLVM) project, while historically important, is based on LLVM version 4.0, which was released in 2017\.12 Attempting to build upon this foundation would be a strategic error. The LLVM API undergoes substantial changes between major versions, rendering passes written for LLVM 4.0 incompatible with modern toolchains.14 Furthermore, the specific obfuscation patterns generated by this older version are well-documented and have been targeted by numerous deobfuscation tools and research papers, diminishing their effectiveness.16

The modern and more robust approach is to build the obfuscation passes as a dynamically loadable plugin for a contemporary, vanilla LLVM build (e.g., version 14 or newer). This strategy offers several distinct advantages:

1. **Future-Proofing**: The project remains compatible with the latest C++ standards and LLVM features.  
2. **Security**: It avoids the known vulnerabilities and deobfuscation patterns associated with older, specific forks.  
3. **Modularity**: It separates our custom logic from the core LLVM codebase, simplifying development and maintenance. Projects like eshard/obfuscator-llvm have successfully adopted this plugin-based model for modern LLVM versions.18

Therefore, the recommended course of action is to build our tool as a plugin that can be loaded by the standard LLVM opt tool.

#### **Build Process**

The following steps detail the process for building the required LLVM toolchain from source, which will serve as the host for our obfuscation plugin.

1. **Clone the LLVM Project Repository**:  
   Bash  
   git clone https://github.com/llvm/llvm-project.git  
   cd llvm-project

2. **Configure with CMake**: Create a build directory and run CMake to generate the build files. We will use the Ninja build system for its speed.4 It is essential to enable the Clang (frontend) and LLD (linker) subprojects, as they are required for a complete compilation pipeline. A  
   Release build type is recommended for performance.  
   Bash  
   mkdir build  
   cd build  
   cmake \-G Ninja../llvm \\  
         \-DLLVM\_ENABLE\_PROJECTS="clang;lld" \\  
         \-DCMAKE\_BUILD\_TYPE=Release \\  
         \-DCMAKE\_INSTALL\_PREFIX="/opt/llvm-obfuscator" \\  
         \-DLLVM\_ENABLE\_ASSERTIONS=ON

3. **Compile and Install**: Use Ninja to execute the build. This process is computationally intensive and may take a significant amount of time.  
   Bash  
   ninja  
   sudo ninja install

This process will result in a complete, modern LLVM toolchain installed in /opt/llvm-obfuscator, ready to be used by our application.

### **2.2. The Obfuscation Pipeline: From Source to Obfuscated Binary**

The core workflow of our application will follow the standard LLVM compilation pipeline, with our custom obfuscation stage injected into the middle-end. The entire process, from a developer's C++ file to a protected executable, is visualized in the flowchart below.

Code snippet

graph TD  
    A \--\> B{Frontend: Clang};  
    B \--\> C;  
    C \--\> D{Middle-End: Standard Optimizations (opt \-O2)};  
    D \--\> E;  
    E \--\> F{Middle-End: Custom Obfuscation Passes (Our Tool)};  
    F \--\> G;  
    G \--\> H{Backend: Code Generator (llc)};  
    H \--\> I;  
    I \--\> J{Linker: LLD};  
    J \--\> K;

*Figure 1: End-to-end obfuscation pipeline.*

1. **Input**: The process begins with a standard C or C++ source file.  
2. **Frontend (Clang)**: The clang compiler is invoked with flags like \-S \-emit-llvm to parse the source code and generate human-readable LLVM IR.4  
3. **Standard Optimization**: The generated IR is first passed through a standard optimization pipeline (e.g., \-O2). This is a critical step to ensure that our obfuscations are applied to an already efficient representation of the code, preventing simple optimizations from undoing our work later.  
4. **Custom Obfuscation**: The optimized IR is then fed into our custom tool, which uses the LLVM opt utility to load and run our sequence of obfuscation passes (implemented as a dynamic plugin).  
5. **Backend (Code Generation)**: The llc tool takes the final, obfuscated LLVM IR and compiles it into an object file for the specified target platform (e.g., x86-64 Windows).19  
6. **Linking (LLD)**: The lld linker combines the object file with any necessary system libraries to produce the final executable binary.2

### **2.3. The Pass Manager Configuration: A Critical Step**

The LLVM Pass Manager is responsible for scheduling the execution of passes and managing the analysis results they depend on.7 The order in which passes are executed is of paramount importance in obfuscation. Many standard compiler optimizations, particularly Dead Code Elimination (DCE), are designed to simplify code and remove redundancies.21 If run

*after* our obfuscation passes, these optimizations could potentially identify and remove the very bogus code and complex structures we have inserted, thereby nullifying our efforts.23

This leads to a core architectural principle for our tool: the obfuscation pipeline must be the *final* set of transformations applied to the LLVM IR before it is handed off to the backend for code generation. The standard workflow must be:

1. Compile source to IR.  
2. Apply standard optimizations (-O1, \-O2, etc.).  
3. Apply our custom obfuscation passes.  
4. Generate machine code.

This ordering ensures that the obfuscated structures are preserved in the final binary. Our command-line application will be responsible for orchestrating this sequence of clang, opt, and llc calls.

The modularity of LLVM passes enables a "mix-and-match" approach to security. The true resilience of the obfuscated binary will not stem from a single, monolithic transformation but from the synergistic composition of multiple, layered passes. For instance, a Control Flow Flattening pass makes the program's structure difficult to discern, but a subsequent String Encryption pass can then hide critical data *within* that already confusing structure. One layer of defense reinforces the next. Consequently, the tool's user interface must allow the user to define a custom sequence of passes, enabling them to experiment with different combinations to achieve the desired level of protection. The final report and presentation should emphasize not just the individual obfuscation techniques, but the powerful, compounding effect of their strategic combination, which erects a significantly higher barrier for reverse engineers.

---

## **III. Core Obfuscation Pass Implementation: A Practical Guide**

This section provides the technical blueprint for implementing the core obfuscation passes required by the problem statement. The focus will be on using the modern LLVM PassManager API, which is standard in recent LLVM versions. Each pass will be developed as a self-contained module in C++, designed to be compiled into a shared library plugin.

### **3.1. Writing a Custom LLVM Pass: The Boilerplate**

All modern LLVM passes follow a standard structure. They are C++ classes that inherit from the llvm::PassInfoMixin\<T\> template, which provides the necessary boilerplate for integration with the new PassManager.25 The core logic of the pass is implemented in a

run method, which receives a unit of IR (e.g., a Function or Module) and an AnalysisManager as arguments.

To make the pass available as a dynamically loadable plugin, an entry point function must be defined using the LLVM\_PLUGIN\_API\_VERSION macro. This function registers a callback with the PassBuilder, allowing the pass to be added to the pipeline via a command-line string.27

A typical skeleton for a FunctionPass plugin looks like this:

**MyPass.h:**

C++

\#**include** "llvm/IR/PassManager.h"

namespace llvm {  
class MyPass : public PassInfoMixin\<MyPass\> {  
public:  
  PreservedAnalyses run(Function \&F, FunctionAnalysisManager \&AM);  
};  
} // namespace llvm

**MyPass.cpp:**

C++

\#**include** "MyPass.h"  
\#**include** "llvm/Passes/PassBuilder.h"  
\#**include** "llvm/Passes/PassPlugin.h"

using namespace llvm;

PreservedAnalyses MyPass::run(Function \&F, FunctionAnalysisManager \&AM) {  
  // Core pass logic goes here.  
  //...

  // Return PreservedAnalyses::all() if the IR was not changed.  
  // Return PreservedAnalyses::none() if the IR was changed.  
  return PreservedAnalyses::all();  
}

// Plugin registration  
extern "C" LLVM\_ATTRIBUTE\_WEAK ::llvm::PassPluginLibraryInfo  
llvmGetPassPluginInfo() {  
  return {LLVM\_PLUGIN\_API\_VERSION, "MyPass", "v0.1",  
         (PassBuilder \&PB) {  
            PB.registerPipelineParsingCallback(  
               (StringRef Name, FunctionPassManager \&FPM,  
                   ArrayRef\<PassBuilder::PipelineElement\>) {  
                  if (Name \== "my-pass") {  
                    FPM.addPass(MyPass());  
                    return true;  
                  }  
                  return false;  
                });  
          }};  
}

### **3.2. Pass 1: Bogus Code Generation (BogusControlFlow)**

The goal of this pass is to insert semantically null but computationally present code to confuse static analysis and increase the complexity of the Control Flow Graph (CFG). A naive implementation of junk code insertion is easily defeated by modern optimizers. The key is to make the inserted code appear "live" to the compiler's Dead Code Elimination (DCE) pass.21

#### **Implementation Strategy**

This pass will be implemented as a FunctionPass. The strategy is to insert bogus code that has an observable side effect, which the optimizer cannot safely remove. A common and effective technique is to perform operations that read from and write to a global variable or a volatile local variable.29

1. **Iterate Basic Blocks**: The pass will iterate through each BasicBlock in the Function.  
2. **Identify Insertion Points**: For each block, it will select a probabilistic insertion point (e.g., before the terminator instruction).  
3. **Create Opaque Predicate**: An opaque predicate is a conditional branch whose outcome is known at obfuscation time but is difficult for a static analyzer to determine. A simple example is if (x \* x \>= 0). We will generate the IR for such a condition.  
4. **Split Block and Insert Branch**: The original basic block will be split at the insertion point. A new conditional branch based on the opaque predicate will be inserted. One branch will lead to a new "bogus" block, while the other (the always-taken path) will lead to the remainder of the original block.  
5. **Populate Bogus Block**: The bogus block will contain irrelevant instructions, such as arithmetic operations on a volatile variable to prevent optimization. It will then unconditionally branch back to the main execution flow.

This approach not only adds junk instructions but also creates false edges in the CFG, significantly complicating manual and automated analysis.

### **3.3. Pass 2: String Encryption**

This pass aims to hide sensitive string literals within the binary, preventing them from being easily discovered with tools like strings. The most secure approach is to decrypt strings on the stack just before they are used and have them be automatically deallocated when the function returns, minimizing the time the plaintext exists in memory.15

#### **Implementation Strategy**

This pass is best implemented as a ModulePass, as it needs to operate on global variables and potentially inject new functions into the module.

1. **Identify Target Strings**: Iterate through all global variables in the Module (M.globals()). Identify constant string literals, which are typically represented as llvm::ConstantDataArray of type i8.30  
2. **Encrypt and Create New Global**: For each identified string, apply a simple encryption algorithm (e.g., a rolling XOR cipher). Create a new GlobalVariable in the module to store this encrypted byte array.  
3. **Inject Decryption Routine**: Create a helper function within the module that performs the decryption. This function will be generated as LLVM IR. It will take a pointer to the encrypted data, allocate a buffer on the stack using alloca, loop through the data to decrypt it into the buffer, and return a pointer to the now-decrypted string on the stack.  
4. **Replace All Uses**: Find every use of the original plaintext string global (GlobalVariable::users()). At each use site, insert a CallInst to the newly injected decryption routine, passing the encrypted global as an argument. Then, replace the original use with the result of this call using Value::replaceAllUsesWith().  
5. **Clean Up**: After all uses have been replaced, the original plaintext GlobalVariable is no longer needed and can be safely removed from the module using GlobalVariable::eraseFromParent().15

This method is highly effective. The original strings are completely absent from the final binary, and the plaintext versions only exist transiently on the stack during the execution of the functions that need them.

### **3.4. Pass 3: Fake Loop Insertion**

This pass is designed to increase the computational complexity and analysis time of a function by inserting loops that perform no useful work. These loops can thwart dynamic analysis by significantly increasing execution time and can confuse static analysis tools by increasing the cyclomatic complexity.

#### **Implementation Strategy**

This will be implemented as a FunctionPass.

1. **Select Target Block**: Similar to the Bogus Control Flow pass, iterate through the basic blocks of the function and select a target for loop insertion.  
2. **Split the Block**: Split the target basic block into two parts: Pre-Loop and Post-Loop.  
3. **Create Loop Blocks**: Create three new basic blocks: Loop.Header, Loop.Body, and Loop.Exit.  
4. **Construct the Loop**:  
   * The Pre-Loop block will initialize a loop counter and then unconditionally branch to Loop.Header.  
   * The Loop.Header will contain a PHINode for the loop counter and the comparison instruction that checks for the loop's exit condition (e.g., counter \< 10000). It will have a conditional branch to Loop.Body or Loop.Exit.  
   * The Loop.Body will contain the bogus instructions (e.g., complex arithmetic on variables local to the loop). It will increment the loop counter and unconditionally branch back to Loop.Header, forming the backedge.31  
   * The Loop.Exit block will be the target for the exit branch from the header. This block will then unconditionally branch to the Post-Loop block, resuming the original program flow.  
5. **Ensure Correctness**: Properly constructing the PHINode in the loop header is critical to ensure the loop variable is correctly updated from both the pre-header (initial value) and the loop body (incremented value).32

This process correctly injects a well-formed loop structure into the function's CFG, significantly increasing its structural complexity without altering its original semantics.

---

## **IV. The User-Facing Application: Interface and Reporting**

A powerful obfuscation engine requires an equally well-designed user interface to make its capabilities accessible and controllable. This section details the design of the command-line application that will serve as the front-end to our LLVM-based obfuscator, along with the mechanism for generating the detailed reports required by the problem statement.

### **4.1. Command-Line Interface (CLI) Design**

The application's CLI will be its primary user interface. A well-designed CLI is not merely a functional requirement; it is a hallmark of a professional tool, demonstrating a deep understanding of the tool's purpose and its user's needs. Providing granular control over each obfuscation technique transforms the tool from a simple on/off switch into a flexible framework for security research and application hardening.

The CLI will be implemented in C++ using LLVM's own robust CommandLine library, which is designed for creating complex command-line tools.33 This choice ensures seamless integration with the rest of the LLVM ecosystem. The design philosophy is to provide a powerful and scriptable interface that follows established conventions.34

The following table specifies the command-line arguments that will be supported. This level of parameterization allows the user to precisely control the extent and nature of the obfuscation, fulfilling a key requirement of the problem statement.

**Table 1: Command-Line Interface (CLI) Specification**

| Flag | Parameter | Default | Description |
| :---- | :---- | :---- | :---- |
| \-i, \--input | \<file\> | (Required) | Specifies the input C/C++ source file. |
| \-o, \--output | \<file\> | a.out / a.exe | Specifies the output executable file name. |
| \--target | \<triple\> | (Host Triple) | Sets the target platform (e.g., x86\_64-w64-mingw32). |
| \--report | \<file\> | report.txt | Specifies the path for the obfuscation report. |
| \--enable-bcf | None | false | Enables the Bogus Control Flow pass. |
| \--bcf-prob | \<int 0-100\> | 30 | Probability (%) of applying BCF to a basic block. |
| \--bcf-loop | \<int\> | 3 | Number of times to apply the BCF pass. |
| \--enable-str-enc | None | false | Enables the String Encryption pass. |
| \--enable-fake-loops | None | false | Enables the Fake Loop Insertion pass. |
| \--enable-cff | None | false | Enables the Control Flow Flattening pass. |
| \-h, \--help | None | N/A | Displays the help message. |

### **4.2. Automated Report Generation**

A key deliverable is the generation of a comprehensive report that logs the obfuscation process and its results. The naive approach of using manual global counters is brittle and unprofessional. Instead, we will leverage LLVM's powerful, built-in Statistic framework. This is a "force multiplier" for the project, allowing us to meet a core requirement with minimal custom code while demonstrating advanced knowledge of LLVM's APIs.

#### **Logging Input Parameters**

The main C++ application, after parsing the command-line arguments, will open the specified report file and log all the provided options and their values. This can be achieved using standard C++ file I/O (std::ofstream).36 This creates a record of the exact configuration used for a given obfuscation run.

#### **Collecting Obfuscation Metrics with llvm::Statistic**

The llvm::Statistic class is designed specifically for collecting metrics from compiler passes.37 The process is straightforward and robust:

1. **Define Statistics**: In the C++ source file for each custom pass, we define global Statistic objects using a macro. These objects are automatically registered by LLVM.  
   C++  
   \#**define** DEBUG\_TYPE "my-obfuscator"  
   STATISTIC(NumStringsEncrypted, "Number of strings encrypted");  
   STATISTIC(NumBogusBlocks, "Number of bogus basic blocks inserted");  
   STATISTIC(NumFakeLoops, "Number of fake loops inserted");

2. **Increment Counters**: Inside the run method of each pass, whenever a transformation is successfully applied, the corresponding counter is simply incremented.  
   C++  
   // Inside the String Encryption pass...  
   if (encryption\_successful) {  
     \++NumStringsEncrypted;  
   }

3. **Enable and Print Statistics**: In the main application, before invoking the opt tool to run the passes, we programmatically enable statistics collection. After the passes have run, we can then programmatically request LLVM to print all collected statistics into our report file.  
   C++  
   // In main application  
   llvm::EnableStatistics();  
   //... run the obfuscation pipeline...  
   std::error\_code EC;  
   llvm::raw\_fd\_ostream report\_stream("report.txt", EC);  
   llvm::PrintStatistics(report\_stream);

This mechanism provides thread-safe counters and standardized output formatting for free, representing the professional standard for metric collection within the LLVM framework.

#### **Report Structure**

The final report will be a plain text or Markdown file with a clear structure, directly addressing all points in the "Expected output" section of the problem statement:

* **Section A: Input Parameters**  
  * Logs all command-line flags and values used for the obfuscation run.  
* **Section B: Output File Attributes**  
  * Output filename.  
  * Final binary size in bytes.  
  * Method of obfuscation (a list of enabled passes).  
* **Section C: Obfuscation Metrics**  
  * **Bogus Code Generated**: The value from the NumBogusBlocks statistic.  
  * **Cycles of Obfuscation**: The values from parameters like \--bcf-loop.  
  * **String Obfuscation/Encryption**: The value from the NumStringsEncrypted statistic.  
  * **Fake Loops Inserted**: The value from the NumFakeLoops statistic.

---

## **V. Innovative Features for a Competitive Edge**

To secure a competitive advantage in the hackathon, the project must go beyond the baseline requirements and implement advanced, high-impact obfuscation techniques. This section details the theory and implementation plan for features that will significantly elevate the tool's sophistication and effectiveness. The most potent obfuscation techniques often incur the highest performance and size costs. A winning project must demonstrate an understanding of this trade-off. Real-world obfuscators are rarely applied to an entire program; instead, they target the most critical functions, such as those handling licensing or cryptography.14 Therefore, a highly advanced implementation of these features would involve using C++ attributes (e.g.,

\[\[obfuscate("cff")\]\]) or pragmas to allow developers to selectively apply these heavy transformations to specific functions, demonstrating a professional design that balances security with practicality.

### **5.1. High-Impact Innovation: Control Flow Flattening (CFF)**

Control Flow Flattening is a powerful obfuscation technique that fundamentally alters a function's structure to impede static analysis.40 It dismantles the natural Control Flow Graph (CFG)—the graph of basic blocks and their conditional jumps—and reconstructs it into a dispatcher-based state machine. All original basic blocks are placed at the same level within a large loop, and a state variable dictates which block is executed in each iteration. This transforms a complex, readable CFG into a simple "star" shape, where every block appears to be reachable from a central dispatcher, making it exceptionally difficult for a human analyst to trace the program's logic.16

#### **Implementation Plan**

Implementing CFF is a complex but achievable task for a FunctionPass.

1. **Collect Blocks**: Traverse the target Function and collect a list of all its BasicBlocks, setting aside the entry block.  
2. **Create State Variable**: In the function's entry block, insert an alloca instruction to create a stack variable that will serve as the state variable. Initialize it with a value corresponding to the first real block to be executed.  
3. **Create Dispatcher and Loop**: Create a new basic block to serve as the main dispatcher. This block will contain a load of the state variable followed by a large switch instruction. This dispatcher will be the header of a new loop.  
4. **Modify Original Blocks**: For each original basic block:  
   * Assign it a unique integer ID.  
   * Remove its original terminator instruction (e.g., br, ret).  
   * At the end of the block, add instructions to store the ID of the *next* logical block into the state variable.  
   * Add an unconditional br instruction that jumps back to the dispatcher block.  
5. **Populate the Dispatcher**: For each original basic block, add a case to the switch statement in the dispatcher that jumps to that block when the state variable matches its ID.  
6. **Handle Function Returns**: The original return instructions must be handled specially. The blocks containing them will not jump back to the dispatcher but will instead execute the ret instruction directly.

To enhance resilience against deobfuscation tools that target simple CFF patterns, the update to the state variable can itself be obfuscated using opaque predicates or MBA, making the transitions between states non-trivial to predict statically.16

### **5.2. Stretch Goal 1: Introduction to Virtualization-Based Obfuscation (VBO)**

Virtualization-Based Obfuscation is widely considered one of the most formidable software protection techniques available.41 It operates by translating sections of native machine code into a custom, proprietary bytecode format. A specially designed Virtual Machine (VM) or interpreter is then embedded within the application. At runtime, this VM fetches, decodes, and executes the custom bytecode.39 This forces an attacker into a multi-stage battle: they must first completely reverse engineer the custom VM's architecture and instruction set before they can even begin to analyze the logic of the protected code.44

#### **Simplified Hackathon Implementation: "Lightweight VBO"**

A full-scale VBO implementation is beyond the scope of a hackathon. However, a "Lightweight VBO" can be implemented as a FunctionPass to demonstrate the core concept and showcase a high level of technical ambition.45

1. **Define a Mini-VM Architecture**: Design a simple, stack-based VM with a handful of custom opcodes. For example:  
   * OP\_PUSH\_CONST \<val\>: Push a constant value onto the VM stack.  
   * OP\_ADD: Pop two values, add them, push the result.  
   * OP\_STORE\_REG \<reg\_id\>: Pop a value and store it in a virtual register.  
   * OP\_LOAD\_REG \<reg\_id\>: Push the value from a virtual register.  
   * OP\_HALT: Stop VM execution.  
2. **Implement the Pass**:  
   * The pass will identify very simple, linear sequences of LLVM IR instructions (e.g., loading two variables, adding them, and storing the result).  
   * It will "lift" this sequence, translating it into the equivalent mini-bytecode (e.g., LOAD\_REG 0, LOAD\_REG 1, ADD, STORE\_REG 2, HALT).  
   * This bytecode sequence is stored in a global constant array.  
3. **Inject the Interpreter**: A generic interpreter function, written in C and compiled to LLVM IR on-the-fly (or pre-written), is injected into the module. This interpreter is essentially a large switch statement inside a while loop that fetches and executes opcodes.  
4. **Replace Original Code**: The original sequence of LLVM IR instructions is replaced with a single call instruction to the interpreter, passing it a pointer to the bytecode array.

This simplified implementation effectively demonstrates the principle of replacing native code with interpreted bytecode, a major achievement for a hackathon project.

### **5.3. Stretch Goal 2: Advanced Instruction Substitution with MBA**

Standard instruction substitution might replace a \= b \+ c with a \= b \- (-c). While this adds minor confusion, it is trivial to simplify. Mixed Boolean-Arithmetic (MBA) offers a far more resilient alternative.46 MBA rewrites simple arithmetic expressions into complex, but mathematically equivalent, forms that interleave standard arithmetic (

\+, \*) with bitwise/boolean operators (&, |, ^, \~).47 These expressions are notoriously difficult for automated tools, such as SMT solvers, to simplify, thus providing strong resilience.48

#### **Implementation Plan**

This pass can be implemented using the InstVisitor pattern within a FunctionPass to target specific instructions.

1. **Build an MBA Library**: Create a collection of proven MBA identities for various operations. For example:  
   * x \+ y can be replaced with (x ^ y) \+ 2 \* (x & y)  
   * x \+ y can also be (x | y) \+ (x & y)  
   * x ^ y can be (x | y) \- (x & y)  
2. **Visit and Replace**: The pass will traverse the instructions of a function. When it encounters a BinaryOperator (e.g., Add, Sub, Xor), it will:  
   * Use an IRBuilder to construct the LLVM IR for one of the corresponding MBA expressions from the library.  
   * Replace the original instruction with the newly generated sequence of instructions using Instruction::replaceAllUsesWith().  
3. **Increase Complexity**: To be effective, the transformation must be applied iteratively. A single MBA substitution can sometimes be simplified by compiler optimizations.48 By applying the pass multiple times, expressions become deeply nested and polynomially complex, making them significantly more robust against simplification.24

The following table provides a strategic comparison of these techniques, which can be used to prioritize development efforts during the hackathon and to articulate the project's value during the final presentation.

**Table 2: Comparison of Obfuscation Techniques**

| Technique | Potency (Difficulty to Analyze) | Resilience (vs. Automated Tools) | Performance Cost | Binary Size Cost | Implementation Complexity |
| :---- | :---- | :---- | :---- | :---- | :---- |
| Bogus Control Flow | Low | Low-Medium | Low | Low-Medium | Low |
| String Encryption | Medium | Medium | Low | Low | Medium |
| Control Flow Flattening | High | Medium | Medium-High | Medium | High |
| MBA Substitution | High | High | Medium | Medium-High | High |
| Virtualization (VBO) | Very High | Very High | Very High | Very High | Very High (Hackathon: High) |

---

## **VI. Cross-Platform Binary Generation**

A core requirement of the problem statement is the ability to generate obfuscated binaries for both Windows and Linux. A naive approach would involve maintaining two separate build environments, which would be unmanageable in a hackathon setting. The correct and vastly more efficient strategy is to leverage the native cross-compilation capabilities of the LLVM/Clang toolchain. This turns a potential major roadblock into a straightforward configuration problem, demonstrating astute architectural planning.

### **6.1. The Power of LLVM/Clang for Cross-Compilation**

Unlike traditional compiler toolchains like GCC, which are typically built for a specific host-target pair, Clang is designed from the ground up as a cross-compiler.49 A single installation of Clang can generate code for any architecture and operating system that LLVM supports. This is controlled via the

\--target command-line flag, which accepts a "target triple" string.51

A target triple is a standardized string that specifies the target platform's architecture, vendor, operating system, and application binary interface (ABI). For this project, the relevant triples are:

* **Linux (64-bit)**: x86\_64-unknown-linux-gnu  
* **Windows (64-bit)**: x86\_64-w64-mingw32

By simply providing the appropriate triple to Clang, we instruct the entire LLVM backend to generate code, object files, and executables for that specific platform.

### **6.2. Setting Up the Toolchain**

While Clang can generate the machine code for Windows, it does not inherently contain the Windows-specific headers (windows.h, etc.) or the libraries (kernel32.lib, etc.) needed to link a functional executable. The llvm-mingw project provides a complete MinGW-w64 toolchain built with LLVM, including all necessary headers, libraries, and a Windows-compatible C++ standard library (libc++).52 This is the recommended solution for providing the Windows "sysroot" (system root) on a Linux development host.

The setup process is as follows:

1. **Development Environment**: A Linux-based OS (e.g., Ubuntu) is recommended as the primary development host.  
2. **Install Native Toolchain**: The build of LLVM/Clang from Section II will serve as the native compiler for the Linux target.  
3. **Download and Extract llvm-mingw**: Download the latest x86\_64 Linux host release of llvm-mingw from its GitHub page and extract it to a known location (e.g., /opt/llvm-mingw).53 This directory contains the complete sysroot needed for Windows cross-compilation.

### **6.3. Build Automation with CMake**

CMake is the de facto build system for C++ projects and has excellent support for cross-compilation through the use of toolchain files.55 A toolchain file is a script that tells CMake which compiler to use and how to configure it, overriding its default behavior of searching for a native compiler.

We will create a toolchain file specifically for cross-compiling to Windows from our Linux host.

**windows-toolchain.cmake:**

CMake

\# The target system name  
set(CMAKE\_SYSTEM\_NAME Windows)

\# Specify the cross-compiler  
set(CMAKE\_C\_COMPILER clang)  
set(CMAKE\_CXX\_COMPILER clang++)

\# Configure the compiler for the target  
set(TARGET\_TRIPLE x86\_64-w64-mingw32)  
set(CMAKE\_C\_COMPILER\_TARGET ${TARGET\_TRIPLE})  
set(CMAKE\_CXX\_COMPILER\_TARGET ${TARGET\_TRIPLE})

\# Set the sysroot to our llvm-mingw installation  
set(CMAKE\_SYSROOT /opt/llvm-mingw)

\# Configure search paths  
set(CMAKE\_FIND\_ROOT\_PATH\_MODE\_PROGRAM NEVER)  
set(CMAKE\_FIND\_ROOT\_PATH\_MODE\_LIBRARY ONLY)  
set(CMAKE\_FIND\_ROOT\_PATH\_MODE\_INCLUDE ONLY)  
set(CMAKE\_FIND\_ROOT\_PATH\_MODE\_PACKAGE ONLY)

With this file, building for either platform becomes a simple, repeatable process:

**To build for Linux (native):**

Bash

mkdir build-linux && cd build-linux  
cmake.. \-DCMAKE\_C\_COMPILER=/opt/llvm-obfuscator/bin/clang \-DCMAKE\_CXX\_COMPILER=/opt/llvm-obfuscator/bin/clang++  
make

**To build for Windows (cross-compile):**

Bash

mkdir build-windows && cd build-windows  
cmake.. \-DCMAKE\_TOOLCHAIN\_FILE=../windows-toolchain.cmake  
make

This automated workflow is robust, efficient, and perfectly suited for the rapid iteration cycles of a hackathon.

---

## **VII. Strategic Roadmap for Hackathon Success**

A successful hackathon project requires not only a strong technical vision but also disciplined time management. This roadmap divides the project into three distinct phases, prioritizing a functional core before moving on to advanced, high-impact features.

### **Phase 1: Minimum Viable Product (MVP) \- The First 8 Hours**

The primary goal of this phase is to establish a working end-to-end toolchain and development environment. This de-risks the project early by ensuring all foundational components are correctly configured.

* **Tasks**:  
  1. **Build LLVM Toolchain**: Complete the full build of LLVM, Clang, and LLD from source as detailed in Section II. This should be started immediately as it is time-consuming.  
  2. **Develop CLI Wrapper**: Create the basic C++ application that can parse command-line arguments (--input, \--output). This application should be able to orchestrate the basic compilation flow: call clang to get IR, then llc to get an object file, and finally lld to produce a native executable for the host platform.  
  3. **Implement "Hello World" Pass**: Create the simplest possible FunctionPass that does nothing but print the names of functions it visits. Compile it as a plugin (.so) and successfully load and run it using the opt tool invoked from the CLI wrapper. This step validates the entire custom pass development workflow.

### **Phase 2: Core Features \- The Next 12 Hours**

With the foundation in place, this phase focuses on implementing the core requirements of the problem statement to ensure a complete and compliant submission.

* **Tasks**:  
  1. **Implement Bogus Code Generation**: Develop the BogusControlFlow pass. Start with simple instruction insertion and then add opaque predicates to create false CFG edges.  
  2. **Implement String Encryption**: Develop the StringEncryption pass. Focus on the core logic of identifying, encrypting, replacing, and cleaning up global strings.  
  3. **Integrate llvm::Statistic Framework**: Add STATISTIC counters to both new passes to track the number of transformations performed.  
  4. **Implement Report Generation**: Enhance the CLI wrapper to enable and capture the statistics from LLVM and format them into the required report file.  
  5. **Enable Cross-Compilation**: Create the CMake toolchain file for Windows and verify that the application can successfully generate a working Windows executable from the Linux host.

### **Phase 3: Innovative Features & Polish \- The Final 16 Hours**

This final phase is dedicated to implementing the standout features that will differentiate the project and preparing a polished final presentation.

* **Tasks**:  
  1. **Implement Control Flow Flattening**: This is the highest-priority innovative feature. Its visual impact on the CFG is dramatic and makes for a compelling demonstration.  
  2. **Attempt Stretch Goal**: If CFF is completed and time permits, begin work on either the Lightweight VBO or the MBA substitution pass. Even a partial or proof-of-concept implementation demonstrates significant technical depth and ambition.  
  3. **Refine and Test**: Thoroughly test the tool with various C++ source files. Refine the CLI options and ensure the generated report is clear and well-formatted.  
  4. **Prepare Presentation**: Create a compelling presentation and live demo. The demo should clearly show a "before" and "after" view of a program's CFG (using a tool like Ghidra or IDA Pro) to visually demonstrate the power of the obfuscations, particularly CFF. The presentation should articulate the architectural decisions and highlight the innovative aspects of the solution.

---

## **VIII. Conclusions**

This report has outlined a comprehensive and technically rigorous plan for developing a state-of-the-art software obfuscation tool using the LLVM compiler infrastructure. The proposed solution is designed not only to meet but to exceed the requirements of the problem statement, providing a clear path to creating a competitive and innovative project for the Smart India Hackathon.

The architectural foundation, based on a modern LLVM version and a modular plugin design, ensures the project is both robust and extensible. The detailed implementation plans for core obfuscation passes—Bogus Code Generation, String Encryption, and Fake Loop Insertion—provide a solid baseline of functionality. The integration of LLVM's native statistics framework for automated reporting demonstrates a professional and efficient approach to meeting the project's deliverables.

Crucially, the plan identifies and details several high-impact innovative features, such as Control Flow Flattening, Mixed Boolean-Arithmetic, and a lightweight form of Virtualization-Based Obfuscation. These advanced techniques, when implemented, will serve as powerful differentiators, showcasing a deep understanding of advanced software protection paradigms. The inclusion of a robust, automated cross-compilation strategy using Clang and CMake addresses a key technical challenge efficiently, allowing the development team to focus their efforts on the core obfuscation logic.

By following the strategic, phased roadmap, a development team can systematically build from a functional MVP to a feature-rich, competition-winning application. The final product will be a testament to sophisticated compiler engineering and a significant contribution to the field of software security.

#### **Works cited**

1. LLVM \- Wikipedia, accessed September 24, 2025, [https://en.wikipedia.org/wiki/LLVM](https://en.wikipedia.org/wiki/LLVM)  
2. The LLVM Compiler Infrastructure Project, accessed September 24, 2025, [https://llvm.org/](https://llvm.org/)  
3. Obfuscation (software) \- Wikipedia, accessed September 24, 2025, [https://en.wikipedia.org/wiki/Obfuscation\_(software)](https://en.wikipedia.org/wiki/Obfuscation_\(software\))  
4. Getting Started with the LLVM System — LLVM 22.0.0git documentation \- LLVM.org, accessed September 24, 2025, [https://llvm.org/docs/GettingStarted.html](https://llvm.org/docs/GettingStarted.html)  
5. The Architecture of Open Source Applications (Volume 1)LLVM, accessed September 24, 2025, [https://aosabook.org/en/v1/llvm.html](https://aosabook.org/en/v1/llvm.html)  
6. A Deep Dive into LLVM: The Future of Compiler Technology | by Aastha Jain | Medium, accessed September 24, 2025, [https://medium.com/@aastha.j901/a-deep-dive-into-llvm-the-future-of-compiler-technology-aa6ceaa4f761](https://medium.com/@aastha.j901/a-deep-dive-into-llvm-the-future-of-compiler-technology-aa6ceaa4f761)  
7. Understanding LLVM Passes and Their Types \- CompilerSutra, accessed September 24, 2025, [https://compilersutra.com/docs/llvm/intermediate/what\_is\_llvm\_passes/](https://compilersutra.com/docs/llvm/intermediate/what_is_llvm_passes/)  
8. Understanding LLVM Passes \- CompilerSutra, accessed September 24, 2025, [https://compilersutra.com/docs/llvm/llvm\_basic/pass/understanding\_llvm\_pass/](https://compilersutra.com/docs/llvm/llvm_basic/pass/understanding_llvm_pass/)  
9. Can Large Language Models Understand Intermediate Representations? \- arXiv, accessed September 24, 2025, [https://arxiv.org/html/2502.06854v1](https://arxiv.org/html/2502.06854v1)  
10. Home · obfuscator-llvm/obfuscator Wiki \- GitHub, accessed September 24, 2025, [https://github.com/obfuscator-llvm/obfuscator/wiki](https://github.com/obfuscator-llvm/obfuscator/wiki)  
11. Obfuscator-LLVM — Software Protection for the Masses \- Pascal Junod, accessed September 24, 2025, [https://crypto.junod.info/spro15.pdf](https://crypto.junod.info/spro15.pdf)  
12. Malware development part 6 \- advanced obfuscation with LLVM and template metaprogramming \- 0xPat blog, accessed September 24, 2025, [https://0xpat.github.io/Malware\_development\_part\_6/](https://0xpat.github.io/Malware_development_part_6/)  
13. obfuscator-llvm/obfuscator \- GitHub, accessed September 24, 2025, [https://github.com/obfuscator-llvm/obfuscator](https://github.com/obfuscator-llvm/obfuscator)  
14. IOLLVM: ENHANCED VERSION OF OLLVM \- arXiv, accessed September 24, 2025, [https://arxiv.org/pdf/2203.03169](https://arxiv.org/pdf/2203.03169)  
15. Extending LLVM for Code Obfuscation (2 of 2\) | Praetorian, accessed September 24, 2025, [https://www.praetorian.com/blog/extending-llvm-for-code-obfuscation-part-2/](https://www.praetorian.com/blog/extending-llvm-for-code-obfuscation-part-2/)  
16. Dissecting LLVM Obfuscator Part 1 \- RPISEC, accessed September 24, 2025, [https://rpis.ec/blog/dissection-llvm-obfuscator-p1/](https://rpis.ec/blog/dissection-llvm-obfuscator-p1/)  
17. Deobfuscation: recovering an OLLVM-protected program \- Quarkslab's blog, accessed September 24, 2025, [https://blog.quarkslab.com/deobfuscation-recovering-an-ollvm-protected-program.html](https://blog.quarkslab.com/deobfuscation-recovering-an-ollvm-protected-program.html)  
18. eshard/obfuscator-llvm \- GitHub, accessed September 24, 2025, [https://github.com/eshard/obfuscator-llvm](https://github.com/eshard/obfuscator-llvm)  
19. A curated list of awesome LLVM (including Clang, etc) related resources. \- GitHub, accessed September 24, 2025, [https://github.com/learn-llvm/awesome-llvm](https://github.com/learn-llvm/awesome-llvm)  
20. Writing an LLVM Pass (legacy PM version) — LLVM 22.0.0git documentation \- LLVM.org, accessed September 24, 2025, [https://llvm.org/docs/WritingAnLLVMPass.html](https://llvm.org/docs/WritingAnLLVMPass.html)  
21. Global Dead Code Elimination for LLVM, revisited \- Quarkslab's blog, accessed September 24, 2025, [https://blog.quarkslab.com/global-dead-code-elimination-for-llvm-revisited.html](https://blog.quarkslab.com/global-dead-code-elimination-for-llvm-revisited.html)  
22. lib/Transforms/Scalar/DCE.cpp Source File \- LLVM.org, accessed September 24, 2025, [https://llvm.org/doxygen/DCE\_8cpp\_source.html](https://llvm.org/doxygen/DCE_8cpp_source.html)  
23. Challenges when building an LLVM-based obfuscator, accessed September 24, 2025, [https://llvm.org/devmtg/2017-10/slides/Guelton-Challenges\_when\_building\_an\_LLVM\_bitcode\_Obfuscator.pdf](https://llvm.org/devmtg/2017-10/slides/Guelton-Challenges_when_building_an_LLVM_bitcode_Obfuscator.pdf)  
24. Arithmetic Obfuscation | O-MVLL Documentation, accessed September 24, 2025, [https://obfuscator.re/omvll/passes/arithmetic/](https://obfuscator.re/omvll/passes/arithmetic/)  
25. Writing an LLVM Pass \- ROCm Documentation, accessed September 24, 2025, [https://rocm.docs.amd.com/projects/llvm-project/en/latest/LLVM/llvm/html/WritingAnLLVMNewPMPass.html](https://rocm.docs.amd.com/projects/llvm-project/en/latest/LLVM/llvm/html/WritingAnLLVMNewPMPass.html)  
26. Writing an LLVM Pass — LLVM 22.0.0git documentation \- LLVM.org, accessed September 24, 2025, [https://llvm.org/docs/WritingAnLLVMNewPMPass.html](https://llvm.org/docs/WritingAnLLVMNewPMPass.html)  
27. llvm \- How to automatically register and load modern Pass in Clang? \- Stack Overflow, accessed September 24, 2025, [https://stackoverflow.com/questions/54447985/how-to-automatically-register-and-load-modern-pass-in-clang](https://stackoverflow.com/questions/54447985/how-to-automatically-register-and-load-modern-pass-in-clang)  
28. How to remove dead code in this code? : r/Compilers \- Reddit, accessed September 24, 2025, [https://www.reddit.com/r/Compilers/comments/176b5oi/how\_to\_remove\_dead\_code\_in\_this\_code/](https://www.reddit.com/r/Compilers/comments/176b5oi/how_to_remove_dead_code_in_this_code/)  
29. Extending LLVM for Code Obfuscation (1 of 2\) | Praetorian, accessed September 24, 2025, [https://www.praetorian.com/blog/extending-llvm-for-code-obfuscation-part-1/](https://www.praetorian.com/blog/extending-llvm-for-code-obfuscation-part-1/)  
30. GlobalVariable Class Reference \- LLVM.org, accessed September 24, 2025, [https://llvm.org/doxygen/classllvm\_1\_1GlobalVariable.html](https://llvm.org/doxygen/classllvm_1_1GlobalVariable.html)  
31. LLVM Loop Terminology (and Canonical Forms), accessed September 24, 2025, [https://llvm.org/docs/LoopTerminology.html](https://llvm.org/docs/LoopTerminology.html)  
32. How to create Loop object on the LLVM? \- Stack Overflow, accessed September 24, 2025, [https://stackoverflow.com/questions/34417490/how-to-create-loop-object-on-the-llvm](https://stackoverflow.com/questions/34417490/how-to-create-loop-object-on-the-llvm)  
33. LibTooling — Clang 22.0.0git documentation, accessed September 24, 2025, [https://clang.llvm.org/docs/LibTooling.html](https://clang.llvm.org/docs/LibTooling.html)  
34. Command Line Interface (CLI) \- Visual Studio Code, accessed September 24, 2025, [https://code.visualstudio.com/docs/configure/command-line](https://code.visualstudio.com/docs/configure/command-line)  
35. Command-line interface \- Wikipedia, accessed September 24, 2025, [https://en.wikipedia.org/wiki/Command-line\_interface](https://en.wikipedia.org/wiki/Command-line_interface)  
36. Logging System in C++ \- GeeksforGeeks, accessed September 24, 2025, [https://www.geeksforgeeks.org/cpp/logging-system-in-cpp/](https://www.geeksforgeeks.org/cpp/logging-system-in-cpp/)  
37. include/llvm/ADT/Statistic.h File Reference \- LLVM.org, accessed September 24, 2025, [https://llvm.org/doxygen/Statistic\_8h.html](https://llvm.org/doxygen/Statistic_8h.html)  
38. LLVM \-stats option \- Stack Overflow, accessed September 24, 2025, [https://stackoverflow.com/questions/34804240/llvm-stats-option](https://stackoverflow.com/questions/34804240/llvm-stats-option)  
39. Code Virtualizer Overview \- Oreans Technologies : Software Security Defined., accessed September 24, 2025, [https://www.oreans.com/CodeVirtualizer.php](https://www.oreans.com/CodeVirtualizer.php)  
40. Breaking Control Flow Flattening: A Deep Technical Analysis ..., accessed September 24, 2025, [https://zerotistic.blog/posts/cff-remover/](https://zerotistic.blog/posts/cff-remover/)  
41. Deobfuscation of Virtualization-obfuscated Code through Symbolic Execution and Compilation Optimization \- GMU CS Department, accessed September 24, 2025, [https://cs.gmu.edu/\~zeng/papers/deobfuscation-icics2017.pdf](https://cs.gmu.edu/~zeng/papers/deobfuscation-icics2017.pdf)  
42. Source code level example of basic virtualization obfuscation. \- ResearchGate, accessed September 24, 2025, [https://www.researchgate.net/figure/Source-code-level-example-of-basic-virtualization-obfuscation\_fig2\_343317567](https://www.researchgate.net/figure/Source-code-level-example-of-basic-virtualization-obfuscation_fig2_343317567)  
43. \[Research\] LLVM based VMProtect Devirtualization: Part 1 (EN) \- hackyboiz, accessed September 24, 2025, [https://hackyboiz.github.io/2025/09/11/banda/LLVM\_based\_VMP/en/](https://hackyboiz.github.io/2025/09/11/banda/LLVM_based_VMP/en/)  
44. LLVM-powered deobfuscation of virtualized binaries \- THALIUM, accessed September 24, 2025, [https://blog.thalium.re/posts/llvm-powered-devirtualization/](https://blog.thalium.re/posts/llvm-powered-devirtualization/)  
45. LLVM Obfuscator Based on Virtual Machines with Custom Opcodes and String Encryption, accessed September 24, 2025, [https://dspace.cvut.cz/bitstream/handle/10467/82300/F8-DP-2019-Turcan-Lukas-thesis.pdf?sequence=-1\&isAllowed=y](https://dspace.cvut.cz/bitstream/handle/10467/82300/F8-DP-2019-Turcan-Lukas-thesis.pdf?sequence=-1&isAllowed=y)  
46. MBA-Blast: Unveiling and Simplifying Mixed Boolean-Arithmetic Obfuscation \- USENIX, accessed September 24, 2025, [https://www.usenix.org/system/files/sec21fall-liu-binbin.pdf](https://www.usenix.org/system/files/sec21fall-liu-binbin.pdf)  
47. Improving MBA Deobfuscation using Equality Saturation \- secret club, accessed September 24, 2025, [https://secret.club/2022/08/08/eqsat-oracle-synthesis.html](https://secret.club/2022/08/08/eqsat-oracle-synthesis.html)  
48. Inspecting Compiler Optimizations on Mixed Boolean Arithmetic ..., accessed September 24, 2025, [https://www.ndss-symposium.org/wp-content/uploads/bar2025-final7.pdf](https://www.ndss-symposium.org/wp-content/uploads/bar2025-final7.pdf)  
49. Cross-compilation using Clang — Clang 22.0.0git documentation, accessed September 24, 2025, [https://clang.llvm.org/docs/CrossCompilation.html](https://clang.llvm.org/docs/CrossCompilation.html)  
50. Cross compiling made easy, using Clang and LLVM \- mcilloni's blog, accessed September 24, 2025, [https://mcilloni.ovh/2021/02/09/cxx-cross-clang/](https://mcilloni.ovh/2021/02/09/cxx-cross-clang/)  
51. Cross compilation with Clang and LLVM tools \- Linaro, accessed September 24, 2025, [https://static.linaro.org/connect/bkk19/presentations/bkk19-210.pdf](https://static.linaro.org/connect/bkk19/presentations/bkk19-210.pdf)  
52. Pre-built Toolchains \- mingw-w64, accessed September 24, 2025, [https://www.mingw-w64.org/downloads/](https://www.mingw-w64.org/downloads/)  
53. Cross-compiling for Windows using llvm-mingw \- ROllerozxa, accessed September 24, 2025, [https://voxelmanip.se/2024/12/10/cross-compiling-for-windows-using-llvm-mingw/](https://voxelmanip.se/2024/12/10/cross-compiling-for-windows-using-llvm-mingw/)  
54. Releases · mstorsjo/llvm-mingw \- GitHub, accessed September 24, 2025, [https://github.com/mstorsjo/llvm-mingw/releases](https://github.com/mstorsjo/llvm-mingw/releases)  
55. Cross Compiling With CMake, accessed September 24, 2025, [https://cmake.org/cmake/help/book/mastering-cmake/chapter/Cross%20Compiling%20With%20CMake.html](https://cmake.org/cmake/help/book/mastering-cmake/chapter/Cross%20Compiling%20With%20CMake.html)