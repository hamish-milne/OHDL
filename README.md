# Creating a modern HDL with mixed-mode programming and object-oriented syntax

## Background

To develop for ASIC or FPGA platforms there are, broadly speaking, two options: code in a low-level HDL (essentially Verilog or VHDL) which is passed directly to the platform-specific synthesiser, or use a High-Level Synthesis tool such as LegUp or Vivado to translate C/C++ into one of these languages. While the latter is ideal for implementing existing sequential algorithms in hardware or allowing software developers to work with these platforms with little additional training, they suffer from the drawbacks that typically arise from using a language far removed from the bare metal: long compile times, decreased performance, and difficulty in debugging the compiled code.

As a result, many hardware developers choose to, or are required to, eschew HLS and code directly in a low-level language. In the software world this would be like being unable to code in Prolog, so deciding to use C or BASIC. My goal is to create a third option, a high-level hardware description language that gives the programmer as much control as Verilog or VHDL, while providing features and abstractions that are typically found only in HLS.

## Relation to existing technologies

Making digital design more accessible has long been the goal of many research projects. The most similar existing technologies I know of are [Chisel](https://chisel.eecs.berkeley.edu/), which uses the Scala language, and Kiwi (https://www.microsoft.com/en-us/research/project/kiwi/), which uses .NET languages. Both translate their respective inputs directly into hardware, rather than using a more complex HLS framework.

The primary difference between these projects and my own is that they add hardware design libraries into existing software languages. While this can improve familiarity to software developers, the resultant code isn't as elegant as it could be with a purpose-designed language, in terms of the number of symbols required to type, and the understanding of how the code translates into hardware.

## My vision for the language is:

* That any non-trivial program written in Verilog or VHDL can be expressed in the same or fewer lines, in almost all cases
* That any non-trivial program written for HLS can be expressed with minimal extra effort and reduced area cost, in most cases
* That, for a given task, an engineer with basic understanding of all three systems will be able to complete it in the new language as fast as in HLS, and faster than Verilog/VHDL
* That a pure software engineer can gain familiarity with digital design using the new language as quickly as with HLS, and quicker than Verilog/VHDL
* That the use of the new language has minimal impact on the overall compile time

## Method

I will primarily explore the following features:

* Supporting combinatorial, edge-triggered and multi-cycle functions in the same language
* `async`/`await` task-based functions or the coroutine pattern as a way for building concurrent state machines
* Supporting system clock, multi-clocked and Double Data Rate designs
* Object-oriented syntax, such as inheritance, composition and abstract interfaces, that relates directly to the hardware being synthesised

## Metrics

I can quantitively evaluate the success of the project, therefore, in the following ways:

* The **number of lines** of code required to write a given program in the new language
* The **area** used by the synthesiser, when comparing to the same program written in Verilog or HLS
* The extra **compile time** added by the new language, with pure Verilog as the baseline

Throughout this project I will be continually asking myself and others what they like about some languages, what they dislike about others, and what would influence their decision to use one language over another

## Applications

I believe that the language will be useful in a wide range of digital design applications, but particularly in large systems where abstraction and readability are important, but also determinism, performance or low-level control. This could include signal processing, control logic, or even finance.

## Scope and risks

Wherever possible, I will be leveraging existing technology to allow me to spend time implementing features rather than the compiler framework. To do this, I will be using Microsoft's open-source C# compiler [Roslyn](https://github.com/dotnet/roslyn) as a base, modifying the parsing stage as necessary and swapping the MSIL emitter for Verilog, or VHDL if I have time.

To manage scope, I will first separate tasks into 'required' and 'optional' groups, the former defining the minimum required functionality to produce any valid output, the latter ordered by priority. My hope is that near the end of the project, the vast majority of the 'optional' features will be complete, leaving enough time for an in-depth final evaluation. Managing the project this way allows me to alter the scope as the relevant complications arise.

## Milestones

* Finalise specification, mark features as 'required' or 'optional' with a given priority
* Create a plan of attack for modifying Rosyln's parsing and emitting stages - an internal reference of how the compiler works from the perspective of modifying it for my needs
* Create the basic compiler framework, marking in the Roslyn code the points where I need to change it to implement a given feature
* Implement the 'required' functionality in the parsing stage, allowing a subset of the OHDL spec to be transformed into a syntax tree
* Implement the Verilog emitter to the level required to output the aforementioned syntax tree, verifying its correctness
* Create a suitable evaluation plan, which measures all the metrics defined in the goals above
* Write the Interim Report
* Successively implement each 'optional' feature in the order listed, refining and refactoring existing code where appropriate. This is an open-ended task, as I can spend all my remaining time up to the point where I need to start evaluating and reporting
* Clean up the code and program interface, packaging and documenting it appropriately for release
* Perform the evaluation, using as many willing test subjects as I can find. This will probably be a combination of in-person evaluation and remotely via a web interface
* Write my final report

## Costs and equipment

Since the compiler's Verilog output can be accepted by, for example, Quartus, without any attached hardware and still show all the necessary metrics, the only equipment needed is a compatible PC, such as mine at home or any college computer. For a final demonstration, I may decide to use a DE0 board or similar; any FPGA development kit should do. The software I'll be using (Visual Studio, ReSharper and Quartus) can all be acquired for free as a student.
