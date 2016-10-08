# Project proposal

## Background

To develop for ASIC or FPGA platforms there are, broadly speaking, two options: code in a low-level HDL which is passed directly to the platform-specific synthesiser--essentially Verilog or VHDL--or use a High-Level Synthesis tool such as LegUp or Vivado to translate C/C++ into one of these languages. While the latter is ideal for implementing existing sequential algorithms in hardware or allowing software developers to work with these platforms with little additional training, they suffer from the drawbacks that typically arise from using a language far removed from the bare metal: long compile times, decreased performance, and difficulty in debugging the compiled code.

As a result, many hardware developers choose to, or are required to, eschew HLS and code directly in a low-level language. In the software world this would be like being unable to code in Prolog, so deciding to use C or BASIC. My goal is to create a third option, a high-level hardware description language that gives the programmer as much control as Verilog or VHDL, while providing features and abstractions that are typically found only in HLS.

## My goals for the language are:

* That any non-trivial program written in Verilog or VHDL can be expressed in the same or fewer lines, in almost all cases
* That any non-trivial program written for HLS can be expressed with minimal extra effort and reduced area cost, in most cases
* That, for a given task, an engineer with basic understanding of all three systems will be able to complete it in the new language as fast as in HLS, and faster than Verilog/VHDL
* That a pure software engineer can gain familiarity with digital design using the new language as quickly as with HLS, and quicker than Verilog/VHDL
* That the use of the new language has minimal impact on the overall compile time

The above points define metrics which I can use to evaluate the success of the project.

## I plan to accomplish this by:

* Implementing object-oriented functionality, a paradigm which is familiar to software engineers, and closely relates to the module structure of most digital designs
* Adding additional zero-cost abstractions such as generics
* The ability to easily switch between different modes of operation: combinatorial, edge-triggered, and multi-cycle
* Including as little boilerplate as possible
* Compiling to Verilog and/or VHDL, which should be relatively fast and integrates neatly into the environment of whatever platform is being used
* Keep the compiled code as similar as possible to its input, including the high-level code as comments, keeping identifiers the same and where possible preserving line numbers

Throughout this project I will be continually asking myself and others what they like about some languages, what they dislike about others, and what would influence their decision to use one language over another

## Introducing OHDL - Object-Oriented Hardware Description Language

This is an idea I had a while back, but never got around to anything more than an initial specification, [linked here](Spec.md). It's essentially the answer to the question "What if C# was a HDL?". I plan to develop the spec further, re-thinking some parts and gathering feedback to ensure I'm proceeding down the right path.

## Scope and risks

Wherever possible, I will be leveraging existing technology to allow me to spend time implementing features rather than the compiler framework. To do this, I will be using Microsoft's open-source C# compiler [Roslyn](https://github.com/dotnet/roslyn) as a base, modifying the parsing stage as necessary and swapping the MSIL emitter for Verilog, or VHDL if I have time. This is particularly useful because the language is syntactically very similar to C#, so much of the existing code can be re-used.

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

I plan to start work on the project as soon as possible, so any potential complications can be identified and dealt with quickly.

## Costs and equipment

Since the compiler's Verilog output can be accepted by, for example, Quartus, without any attached hardware and still show all the necessary metrics, the only equipment needed is a compatible PC, such as mine at home or any college computer. For a final demonstration, I may decide to use a DE0 board or similar; any FPGA development kit should do. The software I'll be using (Visual Studio, ReSharper and Quartus) can all be acquired for free as a student. Therefore the expenditure should be zero, however at the evaluation stage it may be prudent to compensate the in-person test subjects for their time, to the tune of £10-20 each.
