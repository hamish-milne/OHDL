# Background

To develop for ASIC or FPGA platforms there are, broadly speaking, two options: code in a low-level HDL which is passed directly to the platform-specific synthesiser--essentially Verilog or VHDL--or use a High-Level Synthesis tool such as LegUp or Vivado to translate C/C++ into one of these languages. While the latter is ideal for implementing existing sequential algorithms in hardware or allowing software developers to work with these platforms with little additional training, they suffer from the drawbacks that typically arise from using a language far removed from the bare metal: long compile times, decreased performance, and difficulty in debugging the compiled code.

As a result, many hardware developers choose to, or are required to, eschew HLS and code directly in a low-level language. In the software world this would be like being unable to code in Prolog, so deciding to use C or BASIC. My goal is to create a third option, a high-level hardware description language that gives the programmer as much control as Verilog or VHDL, while providing features and abstractions that are typically found only in HLS.

My goals for the language are:

* That any non-trivial program written in Verilog or VHDL can be expressed in the same or fewer lines, in almost all cases
* That any non-trivial program written for HLS can be expressed with minimal extra effort and reduced area cost, in most cases
* That, for a given task, an engineer with basic understanding of all three systems will be able to complete it in the new language as fast as in HLS, and faster than Verilog/VHDL
* That a pure software engineer can gain familiarity with digital design using the new language as quickly as with HLS, and quicker than Verilog/VHDL
* That the use of the new language has minimal impact on the overall compile time

I plan to accomplish this by:

* Implementing object-oriented functionality, a paradigm which is familiar to software engineers, and closely relates to the module structure of most digital designs
* Adding additional zero-cost abstractions such as generics
* The ability to easily switch between different modes of operation: combinatorial, edge-triggered, and multi-cycle
* Including as little boilerplate as possible
* Compiling to Verilog and/or VHDL, which should be relatively fast and integrates neatly into the environment of whatever platform is being used.

Throughout this project I will be asking myself and others what they like about some languages, what they dislike about others, and what would influence their decision to use one language over another

## Introducing OHDL - Object-Oriented Hardware Description Language

This is an idea I had a while back, but never got around to anything more than an initial specification. It's the answer to the question "What if C# was a HDL?". What follows is the aforementioned spec, which gives an idea of my thoughts at the time, but of course it's all subject to change.

# Basic concepts
OHDL includes many common language elements, which are briefly listed here for clarity:

## Style
OHDL is a C-style language:

* Identifiers are strings of any alphanumeric character or the underscore character, but cannot start with a number.
* Blocks of code are delimited with curly braces. They can be omitted in control blocks when they control only a single statement.
* Every statement ends with a semicolon
* Whitespace is only needed to separate identifiers. It is otherwise ignored, and the amount of whitespace used is irrelevant.
* Order of operations can be made explicit with parentheses
* Functions are called with the standard parenthesis syntax.
* Comments are // for single lines and /* */ for blocks.

## Function definitions
```
<keywords> <return type> <function name>
        ( <parameter type> <parameter name> , … )
{
    <body>
    return <return value>;  // Can be omitted for void-return functions
}
```

## Conditional statement
```
if ( <conditional expression> )
{
    <expression is true>
} else if( <conditional expression> )
{
    <other expression is true>
} else
{
    <none of the expressions are true>
}
```

## While loop
```
while ( <conditional expression> )
{
    <execute repeatedly while the expression is true>
}

do
{
    <execute once, then repeatedly while the expression is true>
} while ( <conditional expression );
```

## For loop
```
for( <execute first> ; <conditional expression> ; <execute after every iteration> )
{
    <execute while condition is true>
}

// The usual pattern is:
for ( INT i = 0; i < 5; i++ )
{
    <loop body>
}
```

## Switch statement
```
switch ( <expression> )
{
case <value 1>:
    <executed when expression == value 1>
case <value 2>
    <executed when expression == value 2>
...
default:
    <executed in all other cases>
}
```

# Data types

## Bit
OHDL, like other similar languages, has an ultimate root value type, bit. A bit is a single digital wire that can take a value of `0` or `1`. Multiple bits in an array can represent a greater range of numbers.
The bit type is implied if type information is omitted in any statement, including array definitions.

## Tri
The tri type can take one of three values: `0`, `1` or `X`, with `X` being high impedance. The reason for the distinction between tri and bit is for type safety; with bit you can be assured that your signal has a definite value. Converting between the two types is very simple; bit can be converted to tri freely, but to convert tri to bit requires either an explicit cast, or use of the `Pullup` or `Pulldown` properties, which will convert `X` values to `1` and `0` respectively. Using a cast is equivalent to using `Pulldown`.

```
tri[8] myTri = 0b0011XXXX;

//var b1 = myTri;        // Error. Cannot convert tri to bit without a cast
var b2 = (bit[8])myTri;  // b2 = 0b00110000;
var b3 = myTri.Pulldown; // b3 = 0b00110000;
var b4 = myTri.Pullup;   // b4 = 0b00111111;
```

Note: OHDL does not have separate values for high impedance, unknown values, unset values or weak signals, as these are the same value for all practical purposes and could be interpreted differently by synthesisers.

## Arrays
Multiple values of the same type can be defined with the familiar array syntax, in either pre or postfix forms; both have the same function, though the postfix form only applies to a single variable on the line, while the prefix one applies to the whole line. Array elements are accessed, again, with the familiar indexer syntax, and can have their net taken as well.
Uniquely to OHDL, rather than allowing for specific indexer patterns to indicate endianness, the ‘end’ of the array depends on the position of the indexer operator. If it is before the identifier, the index is taken from the end of the array rather than the beginning.

```
bit[8] myByte;
secondByte[8];  // This has the same effect
                // Note however:
bit[8] a, b;    // Both a and b are arrays
bit c[8], d;    // c is an array, d is a single bit

myByte[0] = 1;  // This sets bit 0 of myByte to be permanently 1
[1]myByte = 0;  // This sets bit 7 - 1 = 6 of myByte to be permanently 0
```

Arrays can be created with (and represented by) a ‘collection initializer’, in the C style, as shown below. They can be ‘spliced’, selecting a range of elements and returning an array of the length selected, by specifying the start and end element in the indexer, separated by a colon. They can also be reversed by prepending ‘[]’ to the variable, which returns the same type as it accepts, net or value.
Note: Reversing, for example, a byte array will reverse the byte order, not the bit order. Casting to a bit array first causes a full bitwise reversal.

```
bit[] myByte = { 0, 1, 0, 0, 1, 1, 0, 1 }; 
// Since we have an initializer, we can omit the length
// 0b10110010 would also work here - but note the reversal
bit[] reversed = []myByte;
bit[] splice = myByte[1:4];    // splice = { 1, 0, 0, 1 }
```

Note that in these cases, the length of the array is omitted in the definition, because it is initialized on the same line, which gives the compiler the same information.
When the index of an array is taken, a constant or variable argument can be used. Constant indices can be used anywhere and as many times as needed, since it is just a single wire. For variable indices, however, there can only be one calling point, since a multiplexer needs to be synthesised. This restriction can be lifted by applying the inline keyword.

## Boolean type
A boolean value can be declared with the ‘bool’ keyword, and is physically identical to a single bit in registers or nets. However, the way they are compared with other numbers is quite different. A number is equal to true when it is non-zero, and false otherwise. This comparison can be disabled by casting explicitly to bit.

```
bool myBool = true;
bool b1 = (0b10100110 == myBool);        // b1 = true
bool b2 = (0b10100110 == (bit)myBool);   // b2 = false
```

## Signedness and representation
Values are presumed unsigned by default, but the signed and unsigned keywords can be prefixed to change this. When casting an unsigned value to a signed value, the its width must be increased by 1 to allow room for the sign bit. Signed values will also have their sign bit extended when cast to a higher width type.

Note: All values in OHDL are little-endian, so the value of the bit at index i is 2^i. This also simplifies casting logic.

## Real numbers
Fixed-point real numbers are supported. The fixed type is parameterised for the number of integer bits, and the number of fractional bits, to allow for greater precision and range as required, while being able to add and subtract normally.

```
fixed<4, 4> foo;        // Range: 0 to 15.9375, precision 0.0625
signed fixed<4, 4> bar; // Range: -8 to 7.9375, precision 0.0625
```

Floating-point numbers are also supported with the float type. By default it implements the IEEE-754 binary32 format with 8 exponent bits and 23 mantissa bits, but a templated form also exists that can accept arbitrary widths for both parameters.
Note: Floating-point operations are available, and yes, they are expensive. One can, of course, derive from these built-in types and override their operators with an external FPU, multi-cycle steps etc.

## Casting
In general, casting a value to a type with a similar/higher range and a similar/higher precision does not require a cast, because no data can be lost. For example, casting `short` to `int`, or `fixed<4, 4>` to `fixed<8, 8>`. If this is not the case, some data could be lost, or the real representation of the number could change, so the casts must be explicit. When casting, fixed type numbers are shifted appropriately, and signed numbers have their sign bit extended.

```
sbyte foo = -10; // 0b11110110
short bar = foo; // bar is -10, or 0b11111111_11110110
```

In addition, it is sometimes useful to transform a number to its bitwise equivalent in another type, ignoring any conversion logic. For this, the raw keyword is provided, and can be prefixed to the casting type. Raw casts will discard any excess input bits (the first/lower ones taking priority), and fill any remaining ones with zeroes. Casts between signed and unsigned types are not affected either way.

```
fixed<4, 4> foo = 10.5;
//short bar = foo; // Error. fixed<4, 4> has fractional values, which byte cannot represent
short baz = (short) foo;     // baz is now 10
short bat = (raw short) foo; // bat is now 0b00000000_10101000 = 168
```

## Type aliases

Types can be aliased with one form of the ‘using’ directive, and are scoped to the current file.

```
using MyType = bit[32];  // This is based on C# syntax
```

Some (global) type aliases exist in the language by default:

```
	byte		OHDL.UInt8 : unsigned bit[8]
	char		OHDL.Char : unsigned bit[8]
	wchar		OHDL.WChar : unsigned bit[16]
	sbyte		OHDL.Int8 : signed bit[8]
	short		OHDL.Int16 : signed bit[16]
	ushort		OHDL.UInt16 : unsigned bit[16]
	int		OHDL.Int32 : signed bit[32]
	uint		OHDL.UInt32 : unsigned bit[32]
	long		OHDL.Int64 : signed bit[64]
	ulong		OHDL.UInt64 : unsigned bit[64]
	fixed<I, F>	OHDL.Fixed<I, F>
	fixed		OHDL.Fixed32 : fixed<16, 16>
	float<E, M>	OHDL.Float<E, M>
	half		OHDL.Float16 : float<5, 10>
	float		OHDL.Float32 : float<8, 23>
	double		OHDL.Float64 : float<11, 52>
	quad		OHDL.Float128 : float<15, 112>
```

Since these are keywords, they cannot be class or variable names. However, they can be ‘overridden’ by an explicit using statement, which is scoped at a maximum to the current file.

# Literals

OHDL has a number of forms for literals:

| Example                | Name                   | Meta type      |
|------------------------|------------------------|----------------|
| `0`                    | Zero                   | INT, FLOAT     |
| `123456789`            | Normal decimal integer | INT, FLOAT     |
| `0b1010_0110`          | Binary pattern         | INT            |
| `0c1234567`            | Octal integer          | INT            |
| `0x12345678_9ABCDEF0`  | Hexadecimal integer    | INT            |
| `1234.5678`            | Real decimal number    | FLOAT          |
| `"abcde"`              | UTF-8 string literal   | STRING         |
| `L"abcde"`             | UTF-16 string literal  | STRING         |
| `true`                 | Boolean true           | BOOL           |
| `false`                | Boolean false          | BOOL           |

Note the underscores in the examples, which can be used in the binary, octal or hexadecimal patterns (collectively known as bit-pattern literals) as separators. They can be used as many times as is needed after the ‘c’, ‘b’ or ‘x’ in the literal, and are ignored by the synthesiser.

Whenever a literal is used in code, it is coerced into whatever type is expected at that point to ensure that the representation of the value is consistent with what is given. For bit-pattern literals, their representation is just that, so the only change that is made is to zero-pad the number if necessary. The same is true for strings. For signed decimal numbers, this changes slightly as to keep representation, the number must be sign-extended if it is negative. Real numbers are synthesised as their closest binary approximation for that type, though a warning will be shown if any precision is being lost.
In the case where a value is entirely out of range for that type, or the assignment is nonsensical (e.g. string to integer) an error occurs. Explicit casts can be used in this case to force the value to be accepted.

One important point is that only decimal literals can be negated without a cast. This is because the hex, octal and binary forms should represent the exact bit pattern, and the result of a negation is therefore undefined. You should cast to a signed type and then perform the negation in these circumstances (it will still be a constant literal). Note that the negation is performed as if the literal were of the type it has been cast to, so if the width is different from the target type, the result may not be as expected:

```
short a = 0b11111111_11111111;   // Valid. a = -1
short b = 0xFFFF;                // Valid. b = -1
//short c = -0x1;                // Error
//short c = -(sbyte)0x1;         // Valid, but c = 0b00000000_11111111
short c = -(short)0x1;           // Valid. c = -1
```

# Meta types
These data types are used in compilation to generate the output, and can be used in (almost) all of the same constructs. For obvious reasons, they can only be literal values (or combinations of literal values). Meta types can be freely cast to one another.

## INT
An integer number that can be used for array length, indexing and counting as well as a normal literal.
Consider the following examples:

```
async void Method1()
{
   for(INT i = 0; i < 5; i++)
        on (clk);
}

async void Method2()
{
    for(int i  = 0; i < 5; i++)
        on (clk);
}
```

How do Method1 and Method2 differ? In Method1, the body of the loop is synthesised 5 times in a row. In Method2, ‘i’ is a real register that changes its value on each iteration. Now consider these examples:

```
int count;

edge void Method3()
{
    for(INT i = 0; i < 5; i++)
        count++;
}

edge void Method4()
{
    //for(int i = 0; i < 5; i++) // This will cause a compiler error
    //    count++;
}
```

In any block that occurs in a single cycle, including all combinatorial and edge-triggered methods, the number of loop iterations must be determinable, which means the iterators must use only meta-data. Method4 will therefore fail to compile.

## BOOL
A boolean value, returned by any comparison between two meta-types. Can be compared as per the normal boolean comparisons.

## FLOAT
A number that can have a fractional component. Floats cannot be used for indexing, but they can be compared with and used as literals.

## STRING
A string of characters, used for compiler and debug output. They can have other metadata concatenated to them with the ‘+’ operator. They can also use the ‘$’ symbol to have other variables embedded in them.

```
INT i = 5;
STRING s = "a string"

print("i: $i");                        // i: 5
print("This is a string: $s");         // This is a string: a string
print("This is a dollar sign: $$");    // This is a dollar sign: $
```

Data concatenated with ‘+’ is evaluated once and saved as a string. Strings with embedded values, as shown above, are evaluated each time as necessary.

## TYPE
A type name, generally used in templated classes, but can also be compared for equality with `==` and `!=`, or for inheritance with `->`.

```
module Example0 { }
module Example1 { }
module Example2 : Example1 { }
module Example3 : Example2, Example0 { }

print(Example2 -> Example1);   // true
print(Example2 -> Example0);   // false
print(Example3 -> Example1);   // true
```

# Signal representation

## Nets
OHDL is explicit in how wires are connected in the system and what logic primitives are synthesised.

By default, all fields are synthesised as registers, however by prefixing the definition with `&` one can define a 'net' or 'reference' that represents a connection to a value of that type, so no registers are placed at all. Local variables and parameters are nets by default, but one can prefix method arguments with reg to indicate that the argument is latched in. All nets must be assigned a value at some point in code, and can only be assigned once.

## Tristates
By applying the tri keyword to a register or net type (only value types, not modules), one can synthesise a tristate buffer or register. A tristate register can be assigned the X value, whereas a tristate buffer has 3 nets: Input, Output and Enable. When the specific net is omitted during assignment, the compiler will use Input and Output appropriately depending on which side of the operator the tristate is on.

```
module TristateExample
{
    &int input;
    &bit en;
    tri &int tristate;

    public TristateExample()
    {
        tristate.Input = input;
        tristate.Enable = en;
    }

    public int Output
    {
        get { return tristate.Output; }
    }
}
```

## Input and output
All fields, arguments etc. are normally inputted values, unless a register is synthesised (in which case the register is used directly). Prefixing the definitions with ‘out’ will define output nets as necessary, but these are generally only useful for certain splicing operations; use properties for anything complex. One can use an explicit ‘in’ for nets as well.

## Bidirectional nets
Direct access to bidirectional ports is generally a bad idea, as it is easily possible to create power-ground shorts. To accommodate external bidirectional ports, OHDL provides the `OHDL.IOPort` module which allows safe reading and writing to a single port using tristate drivers. 

However, to give the programmer absolute control, raw bidirectional nets can be defined with the `unsafe` keyword. These nets are can have either a single bit driver, or any number of tri drivers.
Note: Having more than one bit driver will definitely cause a power-ground short, so it is explicitly forbidden.

## Register pointers
By prefixing a definition with `&reg`, one defines a ‘register pointer’, which is compiled as read and write buses and a clock input. This configuration allows data to be both read from and written to an external register. They can also be converted to normal nets which point to the ‘read’ bus.

Note that to set the value of the register pointed to by the pointer, we use the standard dereference operator, `*`.

## Writeable nets
Normal nets have their value fixed at compile time. However, if a net is defined with the `*` operator on the type, its value can be set at runtime. This is accomplished by synthesising extra multiplexers and registers.

# Operators
OHDL includes the standard set of operators, plus some specific to the language.

| Example | Assignment | Name       | Notes |
|--------|---------|----------------|-------|
|`x + y` |`x += y` | Addition       |  |
|`x - y` |`x -= y` | Subtraction    |  |
|`x * y` |`x *= y` | Multiplication | Expensive |
|`x / y` |`x /= y` | Division       | Very expensive |
|`x & y` |`x &= y` | Bitwise AND    |  |
|`x | y` |`x |= y` | Bitwise OR     |  |
|`x ^ y` |`x ^= y` | Bitwise XOR    |  |
|`~x`    |         | Bitwise NOT    |  |
|`x ~& y`|`x ~&= y`| Bitwise NAND   |  |
|`x ~| y`|`x ~|= y`| Bitwise NOR    |  |
|`x ~^ y`|`x ~^= y`| Bitwise XNOR   |  |
|`x << y`|`x <<= y`| Left shift     |  |
|`x >> y`|`x >>= y`| Right shift    |  |
|        |`x++`    | Post-increment | Post: Returns the original value |
|        |`++x`    | Pre-increment  | Pre: Returns the modified value |
|        |`x--`    | Post-decrement |  |
|        |`--x`    | Pre-decrement  |  |
|        |`x~~`    | Post-negation  |  |
|        |`~~x`    | Pre-negation   |  |
|`x && y`|         | Logical AND    | Logical: returns true or false |
|`x || y`|         | Logical OR     |  |
|`!x`    |         | Logical NOT    |  |
|        |`x = y`  | Assignment     |  |
|        |`x -> y` | Shift-into     | See ‘Parallel assignment’ |
|        |`x <-> y`| Swap           | As above |

If a given platform lacks certain operators (particularly multiply and divide), OHDL can provide code to perform the same functions with appropriate compiler options. Note, however, that the area cost may be prohibitive.

## Comparison with X
Tri type variables can be compared for equality and inequality, but not for order (without being first converted to bit). This is accomplished by comparing both the `Pullup` and `Pulldown` properties of the arguments.

## Multi-clock registers
Many platforms only offer single-edge registers. OHDL allows registers with any number of clocks, at the cost of area. For that reason, all registers are defined by default as single-edge. To allow multiple edges, the `inline` keyword can be applied, which allows as many edges as are required for the program. The total number of clock inputs required for synthesis is calculated on compilation; this includes every time the register is set, or a pointer to the register is taken.

```
int a;			// Normal, single-edge register
inline int b;		// N-edge register
ddr int b;		// Both edges of the same clock
```

The `ddr` keyword can also be applied to methods and modules, making all registers within them dual-edged - allowing clocking on both the rising and falling edge of a single signal. Since some platforms will allow this type of register in hardware by default, a compiler option can be selected that will make use of them where available. Unlike normal multi-clock members, a DDR member is defined as such even when dealing with external nets.

## Length and width
All array types have a read-only property `Length`, which returns the number of elements in the array as an `INT`. To find the number of bits in a given array or struct, however, the `sizeof` keyword is provided, which also returns an `INT`. It can take either a type or variable name, preferring a type name if both exist.

```
bit[23] array;
print(sizeof(array));    // 23
print(array.Length);     // 23

int[2] array2;
print(sizeof(int[2]));   // 64
print(array2.Length);    //  2
```

# Structs and Modules
A module defines a set of registers, nets, properties and methods which represent hardware building blocks. Uniquely to OHDL, modules can be inherited, with multiple inheritance, abstract and virtual methods allowed, as well as access modifiers to all the above. References to modules, as well as embedded sub-modules, can be used. All module members are private by default.

A struct is a collection of related data elements that acts like a primitive type, and can therefore be converted to and from a bit array, have a tristate applied to it, etc. Unlike a module, struct members are public by default, and all properties and methods are inline.  Whether the fields are nets or registers is unspecified in the struct definition; this is decided when a member of this type is declared. Struct members are sequentially stored, making them ideal for use in parsing large, external buses. Structs can be inherited, but they can only derive from one other value type and any number of interfaces.

Note: Because a struct can be easily converted to and from a bit array, it is possible to work around access modifiers. This is similar to C++, where one can use pointer addition to access class members.

Modules and structs - collectively referred to as ‘classes’ for simplicity - are defined with the `module` and `struct` keywords respectively, optional keywords such as access modifiers, a name and finally an open brace, which must be closed at the end of the class definition. Class members are always accessed with the dot operator.

## Access modifiers
Like other object-oriented languages, OHDL includes a number of access modifier keywords. The default visibility for members is `private`, and for modules it is `internal`. Modifier keywords are applied to each member individually, like Java and C#, and unlike C++.

```
	private	Visible only to the containing module.
		protected	Visible to the containing module and modules derived from it
	internal	Visible to the current file
	public	Visible everywhere
		protected internal	Additive combination of protected and internal
```

## Fields
A field is a named item of data that is normally unique to each instance of a module. A field can be of any data type: registers, nets, primitive types, modules or module references. Fields are defined by their access modifiers, their full data type, followed by an identifier and a semicolon.

```
public module CPUState
{
    public uint pc;
    protected int eax, ebx;
    internal bit[24] flags;
}

public struct CommandBus
{
    byte instr;
    short opA, opB;
    uint imm;
}
```

## Constructors
Constructors are used to initialize module instances. They have a similar syntax to methods, but can only take references and meta-types as parameters. Only in constructors can references be assigned to external inputs and outputs.

```
module MyModule
{
    &int a, b, c;

    public MyModule()
    {
        a = 5;
        b = 9;
        c = a + b;
    }

    public MyModule(int a, int b, out int o)
    {
        this.a = a;
        this.b = b;
        c = a + b;
        o = c;
    }
}

MyModule m;  // Created with default constructor (no arguments)
MyModule n(1, 2, null);  // Constructor is used
MyModule(3, 4, myOutput);  // Anonymous member - warning if there are no outputs
```

## Nested modules and references
There are two ways to include a module in another module: an embedded field, and a reference. In the former case, the module name is used as the field’s type with no prefix. This creates an embedded instance in every instance of the containing module, and additional logic is synthesised. Alternatively, a reference can be defined by prefixing the type name with ‘&’, which creates the necessary input and output wires to access the public members of the given module.

Note: Module references are language features; the number of physical wires defined is dependant on what module members are used and in what way, so as such they cannot be converted to bit arrays. Switching module references is supported, however; the compiler will work out what wires need to be assigned where.

Module references can be freely assigned to local variables, and used as return types. The necessary multiplexers are synthesised automatically.

## Static members
Members can be declared with the static keyword, which makes them independent of the module instance and causes them to be synthesised only once. Static methods and properties are considered class members in terms of access, however since they don’t have a reference to a specific module instance, they cannot access any instance members without one.

The static keyword can also be applied to modules, which prevents them from being inherited and instantiated, only allowing static members, which become the default.

# Properties
A property is a set of logic defined like one or two methods, but accessed like a field. Properties have up to two ‘accessors’, get and set, which are defined in the body of the property. The get accessor must return a value of the property’s type using combinational logic, and the set accessor takes as input a value of the property’s type (accessed with the value keyword) and can set registers, being equivalent to an ‘edge’ method. Alternatively, if a read-only property is desired, the body of a ‘get’ accessor can be written in the main body of the property.

```
int a, b, c;

int Output
{
    get { return a & ( b | c ); }
}
```

A property address can be used as a full register pointer if it has both accessors (since the wires are identical), and to a normal net for the property type if it has at least the get accessor. Since property setters require an edge, they must be declared inline if they are to have multiple calling points.

# Indexers
Indexers have similar syntax to properties

# Methods
A method represents a clocked action on a particular module. They can take a number of arguments of any type, which are nets by default, though both input and output nets are allowed. They can also return a value. There are four main categories of methods:

## Combinatorial
This is the default type of method declared. All arguments must be nets, and the body can only contain combinational logic, so no registers or properties can be set, nor edge or async methods called. Register values can still be read, and other combinatorial methods can be caled.

Note: Most combinatorial methods should be declared as ‘inline’, as a normal one will only allow one set of arguments. If there are no arguments, an inline declaration will have no effect.

```
int val;

public int SomeBooleanExpression(int a, int b, int c)
{
    int q = a ^ (b | c);
    return (q + a) | (q + val);
}
```

## Macros
A macro method is one that is computed in its entirety at compile time. It can only deal with meta types, but can include any control structures that do not result in an infinite loop. For obvious reasons, they cannot deal with any non-macro members. Macros are defined with the macro keyword, which is implicit if the return value is a meta-type (i.e. it is required for void-return functions). Macros have no limits on when they are called.

```
module Array<T, INT L>
{
    private T[L] internalArray;

    public INT GetSize()
    {
        return L * sizeof(T);
    }

    // Macros can be get-only properties as well:
    macro INT Length { return L; }
}
```

## Edge triggered
These methods are defined with the edge keyword. Under the hood, an extra wire is synthesised as a clock input, which allows registers to be set and other edge-triggered methods to be called. Async methods can be started, but their return value cannot be checked, nor can they be enumerated, since an edge-triggered method must be completed in a single clock cycle.

```
int myReg;

public edge int SetValues(int a, int b)
{
    myReg = a + b;
    return myReg ^ b;
}
```

Edge methods can have the clocks keyword applied to them to allow multiple calling points, but cannot be inline.

## Asynchronous
The ‘asynchronous’ designation and keyword ‘async’ don’t refer to a lack of clocking - they are very much edge-triggered - but instead mean that these methods can take an arbitrary amount of time from when they are first called to when they return - if they ever return. The term is taken from software languages that start separate threads for these methods, and the analogy is reasonably accurate here.

Async methods can include delays (not fixed time delays, but delays until a clock edge) in their body. Their arguments, unlike other methods, are latched in by default, though a reference can be defined as well.

## Local variables
Methods can define local variables, which are synthesised as nets or registers depending on the method type (nets for combinatorial and edge-triggered, registers for async).

```
int myVar;

edge int MyMethod(byte a, byte b)
{
    int tmp = a * 3;
    myVar = tmp + tmp >> 4 + b;
    return myVar + 1;
}
```

If a local variable is re-assigned (in the same block, for async methods), it is synthesised as a separate wire which uses the old variable as an input. The name will now refer to this new wire.

## Parallel assignment
Consider the following code:

```
int v1, v2, v3;

edge void Shift(int input)
{
    v1 = input;
    v2 = v1;
    v3 = v2;
}
```

OHDL synthesises methods as to keep their operation predictable. So in this case, all `v1`, `v2` and `v3` are set to `input`, because that is what is implied by the code. To create a shift register, one could either reverse the order of the statements, or use the shift-into operator, `->`.

```
int v1, v2, v3;

edge void Shift(int input)
{
    input -> v1 -> v2 -> v3;
}
```

With this operator, the left hand expression is assigned into the right hand variable, and returns the old value of the right hand variable.

We can also define ring-buffers and swap statements. Consider the following:

```
int a, b, c, d, e;

edge void Shift()
{
    a -> b -> c -> d -> e -> a;
}

edge void Swap()
{
    a <-> b;   // Equivalent to: a -> b -> a; or b -> a -> b;
}
```

## Inline and local methods
By default, module methods are declared as ‘local’, which means that a maximum of one set of logic is synthesised. Since local methods are single blocks, they can connect to output nets and write to standard registers, however they can only be called once in code. Alternatively, methods can be declared inline, which synthesises individual blocks at each point in code where they are called. This requires more thought in optimizing area, and removes the ability to connect to outputs, but allows multiple calls without using multi-clocked registers.

# Anonymous members
Methods, properties, events and so on do not have to be named in order for them to compile and function. Consider these three counter implementations:

```
module Counter1
{
    public &reset;
    int count;
    
    public Count { get { return count; } }

    public edge void Increment()
    {
        if(reset)
            count = 0;
        else
            count++;
    }
}

module Counter2
{
    public &clk;
    public &reset;
    int count;
    
    public out Count = { return count; }
    event Increment on(clk -> 1) = { count = reset ? 0 : count + 1; }
}

module Counter3
{
    public & clk, reset;
    int count;
    
    public out Count = count;
    on(clk -> 1) reset ? count = 0 : count++;
}
```

All three perform essentially the same function, but in different styles, becoming progressively more terse. Notice that if a member does not need to be accessed by name, it does not require a name in code. This includes module instances, events, and certain methods (when assigned to events).

## Function pointers
A function pointer is nothing more than a net wired to a specific part of a module. They can be called

# Interfaces
An interface is an abstract group of methods, properties and indexers. In principle, an interface can be defined as a normal abstract module; it’s just a different syntax. All interface members must be public and abstract, being declared as such by default, and they cannot be registers.

```
interface ICollection<T>
{
    int Count { get; }
    T this[int i] { get; set; }
    foreach(T item in int start, end);
}
```

# Events
An event defines the entry points to the code - the moment when your actions are executed. A single entry point can be defined with the on keyword. What follows is a combination of applicable states defining the clock that causes the action to take place. OHDL uses ‘->’ in this sense to mean ‘becomes’, and allows logical operators to combine multiple expressions (or, in fact, any combinatorial expression).

```
public &clk;
int a;

on (clk -> 1) a++;
```

We can also define named events which can have methods added to them at other points in the code. Events can either be called explicitly, or can have ‘on’ expressions in their declaration.

```
public &clk;
int a;

event Increment on (clk -> 1);
Increment += { a++ };
```

The transition state can be any value appropriate for the variable type. The operator can also be omitted completely omitted, which will synthesise differently depending on whether the ddr keyword is present, as shown below.

```
using OHDL.Gates;
&bit[8] clk;

event NormalEvent on (clk);   // Equivalent to (OR(clk) -> 1)
ddr event DDREvent on (clk);   // Equivalent to (OR(clk) -> 1 or OR(clk) -> 0)
```

## Multiple clocks
Clock events can be combined in a single statement using the ‘or’ keyword. Consider the following two snippets:

```
public & clk1, clk2;
inline int a, b;

on (clk1 | clk2 -> 1) a++;
on (clk1 -> 1 or clk2 -> 1) b++;
```

These signals cause `a` to increment by 1, and `b` to increment by 2, because the former uses combinational logic to combine the clocks, whereas the latter uses dual-clocked logic. Be aware of the extra area and (small) delay caused by using multi-clocked registers.

# Errors and warnings
It is very easy to create custom errors and warnings for the use of specific methods and parameters. These are displayed with the error and warning keywords.

```
public override error VirtualMethod();
// Applied to a method, it displays a 'not implemented' error
// if the method is ever used in code.

int c;

public void Method2(INT parameter)
{
    if(parameter < 0)
        error("Method2 parameter ($parameter) is less than zero");
        // Error statements are shown whenever code is synthesised, so a non-meta type
        // for the parameter will cause the error to always be shown.
    c = parameter;
}
```

After an error is displayed, compilation of the block in which it is found stops. Warnings are very much like errors, but their presence will not stop the compilation.

# Reset method
All modules in OHDL have a virtual method, Reset. This method will set all registers to the values specified in their constructors and initializers. The method can then be assigned to an event, such as a reset input going high or low, or a clock input that checks another pin. If the Reset method for a given module type is never called or exposed, setting registers in constructors (or calling edge-triggered methods) is not permitted, and will cause an error.

# Nullable references
It is sometimes helpful to make a reference or pointer optional as a parameter - either to a module, or a method. For this purpose, nullable references can be defined by prepending `?` to the type name, and can then be assigned and checked against the `null` value.

```
?OutputInterface output;

void edge ProcessOutput(int data)
{
    //output.Init();    // Nullable references can only be accessed in checked blocks
    if(output != null)
    {
        output.Output(data[0:8]);
        output.Output(data[8:8]);
    }
}
```

Whether a reference is null or not is compiled statically, so any time such a reference is accessed, it must be within a conditional block that ensures that the reference is not null.

# Operator overloading

# Iterators

# Templates

# Input and output

## Extern and export
The extern and export keywords can be applied to any module field to mean ‘input’ and ‘output’ respectively (extern out has the same effect as export for nets). The usual pattern is to define a single static module with all the external nets as the ‘entry point’ for the overall program, but external nets can be added to module instances too, and in multiple locations.

External nets have some restrictions when compared with normal nets:
They cannot be nullable; they must always, by nature, have a value
All methods and registers for that reference are implicitly local and singly-clocked (or DDR, if the member is defined as such)
