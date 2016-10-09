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
* Comments are `//` for single lines and `/* */` for blocks.

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

## Tri
The tri type can take one of three values: `0`, `1` or `X`, with `X` being high impedance. The reason for the distinction between tri and bit is for type safety; with bit you can be assured that your signal has a definite value. Converting between the two types is very simple; bit can be converted to tri freely, but to convert tri to bit requires either an explicit cast, or use of the `Pullup` or `Pulldown` properties, which will convert `X` values to `1` and `0` respectively. Using a cast is equivalent to using `Pulldown`.

```
tri[8] myTri => 0b0011XXXX;

//var b1 => myTri;        // Error. Cannot convert tri to bit without a cast
var b2 => (bit[8])myTri;  // b2 = 0b00110000;
var b3 => myTri.Pulldown; // b3 = 0b00110000;
var b4 => myTri.Pullup;   // b4 = 0b00111111;
```

OHDL does not have separate values for high impedance, unknown values, unset values or weak signals, as these are the same value for all practical purposes and could be interpreted differently by synthesisers.

## Arrays
Multiple values of the same type can be defined with the familiar array syntax, by appending the length of the array surrounded by square brackets to the type. Note that the postfix syntax found in Verilog or C is not supported. Array elements are accessed, again, with the familiar indexer syntax, and can have their net taken as well.

Multiple array elements can be selected with the slice operator:
```
<array> [ <index> ] // Selects a single element
<array> [ <index> : ] // Selects all elements from <index> to the end
<array> [ : <length> ] // Selects the first <length> elements
<array> [ <index> : <length> ] // Selects the first <length> elements, starting at <index>

// If <index> is negative, the position is taken from the end of the array
// If <length> is negative, the end index moves behind the start

int[] array => {1, 2, 3, 4, 5, 6, 7, 8};
array[1]; // 2
array[2:3]; // 3, 4, 5
array[-1]; // 8
array[-2:2]; // 7, 8
array[-1:-2]; // 8, 7
array[-4:]; // 5, 6, 7, 8
```

Arrays can be created with (and represented by) a ‘collection initializer’, in the C style, as shown below.

```
bit[] myByte => { 0, 0, 0, 0, 1, 1, 1, 1 }; 
// Since we have an initializer, we can omit the length
// 0b11110000 would also work here - but note the reversal
```

Note that in these cases, the length of the array is omitted in the definition, because it is initialized on the same line, which gives the compiler the same information.

When the index of an array is taken, a constant or variable argument can be used. Constant indices are essentially free, but variable indices will use an additional multiplexer.

Slices must have a constant length, and a constant index if the length is omitted. For variable indices, a barrel shift is synthesised.

## Boolean type
A boolean value can be declared with the ‘bool’ keyword, and is physically identical to a single bit in registers or nets. However, the way they are compared with other numbers is quite different. A number is equal to true when it is non-zero, and false otherwise. This comparison can be disabled by casting explicitly to bit.

```
bool myBool => true;
bool b1 => (0b10100110 == myBool);        // b1 = true
bool b2 => (0b10100110 == (bit)myBool);   // b2 = false
```

## Signedness and representation
Values are presumed unsigned by default, but the signed and unsigned keywords can be prefixed to change this. When casting an unsigned value to a signed value, the its width must be increased by 1 to allow room for the sign bit. Signed values will also have their sign bit extended when cast to a higher width type.

All values in OHDL are little-endian, so the value of the bit at index i is 2^i. This also simplifies casting logic.

## Real numbers
Fixed-point real numbers are supported. The fixed type is parameterised for the number of integer bits, and the number of fractional bits, to allow for greater precision and range as required, while being able to add and subtract normally.

```
fixed<4, 4> foo;        // Range: 0 to 15.9375, precision 0.0625
signed fixed<4, 4> bar; // Range: -8 to 7.9375, precision 0.0625
```

Floating-point numbers are also supported with the float type. By default it implements the IEEE-754 binary32 format with 8 exponent bits and 23 mantissa bits, but a templated form also exists that can accept arbitrary widths for both parameters.

Floating-point operations are available, but are normally quite expensive. One can, of course, derive from these built-in types and override their operators with an external FPU, multi-cycle steps etc.

## Casting
In general, casting a value to a type with a similar/higher range and a similar/higher precision does not require a cast, because no data can be lost. For example, casting `short` to `int`, or `fixed<4, 4>` to `fixed<8, 8>`. If this is not the case, some data could be lost, or the real representation of the number could change, so the casts must be explicit. When casting, fixed type numbers are shifted appropriately, and signed numbers have their sign bit extended.

```
sbyte foo => -10; // 0b11110110
short bar => foo; // bar is -10, or 0b11111111_11110110
```

In addition, it is sometimes useful to transform a number to its bitwise equivalent in another type, ignoring any conversion logic. For this, the raw keyword is provided, and can be prefixed to the casting type. Raw casts will discard any excess input bits (the first/lower ones taking priority), and fill any remaining places with zeroes.

```
fixed<4, 4> foo => 10.5;
//short bar => foo; // Error. fixed<4, 4> has fractional values, which byte cannot represent
short baz => (short) foo;     // baz is now 10
short bat => (raw short) foo; // bat is now 0b00000000_10101000 = 168
```

## Type aliases

Types can be aliased with one form of the `using` directive, and are scoped to the current file.

```
using MyType = bit[32];
```

Some global type aliases exist in the language by default:

```
	byte		Sys.UInt<8>
	char		Sys.Char : Sys.UInt<8>
	wchar		Sys.WChar : Sys.UInt<16>
	sbyte		Sys.Int<8>
	short		Sys.Int<16>
	ushort		Sys.UInt<16>
	int		Sys.Int<32>
	uint		Sys.UInt<32>
	long		Sys.Int<64>
	ulong		Sys.UInt<64>
	fixed		Sys.Fixed<16, 16>
	half		Sys.Float<5, 10>
	float		Sys.Float<8, 23>
	double		Sys.Float<11, 52>
	quad		Sys.Float<15, 112>
	tri		Sys.Tri<T>
```

These type names can be ‘overridden’ by an explicit using statement, which is scoped at a maximum to the current file.

# Literals

OHDL has a number of forms for literals:

| Example                | Name                   | Meta type      |
|------------------------|------------------------|----------------|
| `0`                    | Zero                   | INT, FLOAT     |
| `123456789`            | Normal decimal integer | INT, FLOAT     |
| `0b1010_0110`          | Binary pattern         | INT            |
| `0o1234567`            | Octal integer          | INT            |
| `0x12345678_9ABCDEF0`  | Hexadecimal integer    | INT            |
| `1234.5678`            | Real decimal number    | FLOAT          |
| `"abcde"`              | UTF-8 string literal   | STRING         |
| `L"abcde"`             | UTF-16 string literal  | STRING         |
| `true`                 | Boolean true           | BOOL           |
| `false`                | Boolean false          | BOOL           |

Note the underscores in the examples, which can be used in the binary, octal or hexadecimal patterns (collectively known as bit-pattern literals) as separators. They can be used as many times as is needed after the `o`, `b` or `x` in the literal, and are ignored by the synthesiser.

To avoid confusion, C-style octal literals e.g. `01234` are explicitly disallowed.

Whenever a literal is used in code, it is coerced into whatever type is expected at that point to ensure that the representation of the value is consistent with what is given. For bit-pattern literals, their representation is just that, so the only change that is made is to zero-pad the number if necessary. The same is true for strings. For signed decimal numbers, this changes slightly as to keep representation, the number must be sign-extended if it is negative. Real numbers are synthesised as their closest binary approximation for that type, though a warning will be shown if any precision is being lost.

In the case where a value is entirely out of range for that type, or the assignment is nonsensical (e.g. string to integer) an error occurs. Explicit casts can be used in this case to force the value to be accepted.

One important point is that only decimal literals can be negated without a cast. This is because the hex, octal and binary forms should represent the exact bit pattern, and the result of a negation is therefore undefined. You should cast to a signed type and then perform the negation in these circumstances (it will still be a constant literal). Note that the negation is performed as if the literal were of the type it has been cast to, so if the width is different from the target type, the result may not be as expected:

```
short a => 0b11111111_11111111;   // Valid. a = -1
short b => 0xFFFF;                // Valid. b = -1
//short c => -0x1;                // Error
//short c => -(sbyte)0x1;         // Valid, but c = 0b00000000_11111111
short c => -(short)0x1;           // Valid. c = -1
```

# Meta types
These data types are used in compilation to generate the output, and can be used in (almost) all of the same constructs. For obvious reasons, they can only be literal values (or combinations of literal values). Meta types can be freely cast to one another.

## INT
An integer number that can be used for array length, indexing and counting as well as a normal literal.
Consider the following examples:

```
async void Method1()
{
    for(INT i = 0; i < 5; i++) {
        print("Foo"); // Evaluated at compile time
        on (clk);
    }
}

async void Method2()
{
    for(int i = 0; i < 5; i++) {
        print("Foo");
        on (clk);
    }
}
```

Both methods behave identically, but for `Method1`, 'foo' is printed 5 times, but for `Method2` it is printed only once. Now consider these examples:

```
int count;

edge void Method3()
{
    for(INT i = 0; i < 5; i++)
        count++;
}

edge void Method4()
{
    for(int i = 0; i < 5; i++) // This will cause a compiler error
        count++;
}
```

In any block that occurs in a single cycle, including all combinatorial and edge-triggered methods, the number of loop iterations must be determinable, which means the iterators must use only meta-data. `Method4` will therefore fail to compile.

## BOOL
A boolean value, returned by any comparison between two meta-types. Can be compared as per the normal boolean comparisons.

## FLOAT
A number that can have a fractional component. Floats cannot be used for indexing, but they can be compared with and used as literals.

## STRING
A string of characters, used for compiler and debug output. They can have other metadata concatenated to them with the `+` operator. They can also use the `$` symbol to have other variables embedded in them.

```
INT i = 5;
STRING s = "a string"

print("i: $i");                        // i: 5
print("This is a string: $s");         // This is a string: a string
print("This is a dollar sign: $$");    // This is a dollar sign: $
```

Data concatenated with `+` is evaluated once and saved as a string. Strings with embedded values, as shown above, are evaluated each time as necessary.

## TYPE
A type name, generally used in templated classes, but can also be compared for equality with `==` and `!=`, or for inheritance with `is`.

```
module Example0 { }
module Example1 { }
module Example2 : Example1 { }
module Example3 : Example2, Example0 { }

print(Example2 is Example1);   // true
print(Example2 is Example0);   // false
print(Example3 is Example1);   // true
```

# Signal representation

## Nets
OHDL is explicit in how wires are connected in the system and what logic primitives are synthesised.

By default, all fields are compiled as nets, which represent connections between logic blocks. All nets must be connected somewhere at some point in the code, either when they are first defined or elsewhere (in the module's constructor, for instance).

## Registers

To define a register, use the `reg` keyword, which can be applied to module fields only. Registers are assigned using the assignment operator, `=`, which can be used in edge-triggered and async methods.

## Making connections

To join two nets together, use the connection operator `=>`. Module field nets must be connected once and only once, but local variables can be connected multiple times as needed. The right-hand operator can be any expression of the correct type for the variable.

## Tristate buffers
One can use the `tri<T>` type as a wrapper around any value type, which creates a tristate buffer with `Input`, `Output` and `Enable` nets.

## Input and output
All fields, arguments etc. are normally inputted values, unless a register is synthesised (in which case the register is used directly). Prefixing the definitions with `out` will define output nets as necessary, which can be assigned an expression. One can use an explicit `in` for nets as well.

## Bidirectional nets
Direct access to bidirectional ports is generally a bad idea, as it is easily possible to create power-ground shorts. To accommodate external bidirectional ports, OHDL provides the `Sys.IoPort` module which allows safe reading and writing to a single port using tristate drivers. 

However, to give the programmer absolute control, raw bidirectional nets can be defined with the `unsafe` keyword. These nets are can have either a single bit driver, or any number of tri drivers.

Having more than one bit driver will definitely cause a power-ground short, so it is explicitly forbidden.

## Register nets
Register fields, if left unconnected, will have a real register synthesised behind them, however they can be explicitly connected to a separate register or property like any other net. Connected registers can be assigned like any other.

## Writeable nets
Normal nets have their value fixed at compile time. However, if a net is defined with the `&` operator on the type, its value can be set at runtime. This is accomplished by synthesising extra multiplexers and registers.

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

| Name | Description |
|---|---|
|`private`           | Visible only to the containing module. |
|`protected`         | Visible to the containing module and modules derived from it |
|`internal`          | Visible to the current file |
|`public`	     | Visible everywhere |
|`protected internal`| Additive combination of `protected` and `internal` |

## Fields
A field is a named item of data that is normally unique to each instance of a module. A field can be of any data type: registers, nets, primitive types, modules or module references. Fields are defined by their access modifiers, their full data type, followed by an identifier and a semicolon.

```
public module CPUState
{
    public reg uint pc;
    protected reg int eax, ebx;
    internal reg bit[24] flags;
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
    int a, b, c;

    public MyModule()
    {
        a => 5;
        b => 9;
        c => a + b;
    }

    public MyModule(int a, int b, out int o)
    {
        this.a => a;
        this.b => b;
        c => a + b;
        o => c;
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
A property is a set of logic defined like one or two methods, but accessed like a field. Properties have up to two ‘accessors’, get and set, which are defined in the body of the property. The get accessor must return a value of the property’s type using combinatorial logic, and the set accessor takes as input a value of the property’s type (accessed with the `value` keyword) and can set registers, being equivalent to an ‘edge’ method.

```
int a, b, c;

int Output
{
    get { return a & ( b | c ); }
}
```

A property address can be used as a full register net if it has both accessors (since the wires are identical), and to a normal net for the property type if it has at least the get accessor. Since property setters require an edge, they must be declared inline if they are to have multiple calling points.

# Indexers

# Methods
A method represents a clocked action on a particular module. They can take a number of arguments of any type, which are nets by default, though both input and output nets are allowed. They can also return a value. There are four main categories of methods:

## Combinatorial
This is the default type of method declared. All arguments must be nets, and the body can only contain combinational logic, so no registers or properties can be set, nor edge or async methods called. Register values can still be read, and other combinatorial methods can be caled.

Note: Most combinatorial methods should be declared as ‘inline’, as a normal one will only allow one set of arguments. If there are no arguments, an inline declaration will have no effect.

```
int val;

public int SomeBooleanExpression(int a, int b, int c)
{
    int q => a ^ (b | c);
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
reg int myReg;

public edge int SetValues(int a, int b)
{
    myReg = a + b;
    return myReg ^ b;
}
```

Edge methods can have the `inline` keyword applied to them to allow multiple calling points.

## Asynchronous
The ‘asynchronous’ designation and keyword `async` don’t refer to a lack of clocking - they are very much edge-triggered - but instead mean that these methods can take an arbitrary amount of time from when they are first called to when they return - if they ever return. The term is taken from software languages that start separate threads for these methods, and the analogy is reasonably accurate here.

Async methods can include delays (not fixed time delays, but delays until a clock edge) in their body. Remember that arguments are not latched in by default unless `reg` is specified explicitly.

```
public async int Fibonacci(bit clk, reg int n) {
    reg output = 1;
    for(int i = 0; i < n; i++) {
        output += i;
	on(clk);
    }
    return int;
}
```

*TODO: automatically determine number of cycles from platform and frequency*

## Local variables
Methods can define local variables, which, like fields, are nets by default and registers where specified.

```
reg int myVar;

edge int MyMethod(byte a, byte b)
{
    int tmp = a * 3;
    myVar = tmp + tmp >> 4 + b;
    return myVar + 1;
}
```

If a local variable is re-assigned (in the same block, for async methods), it is compiled as a separate wire which uses the old variable as an input. The name will now refer to this new wire.

## Parallel assignment
Consider the following code:

```
reg int v1, v2, v3;

edge void Shift(int input)
{
    v1 = input;
    v2 = v1;
    v3 = v2;
}
```

OHDL synthesises methods as to keep their operation predictable. So in this case, all `v1`, `v2` and `v3` are set to `input`, because that is what is implied by the code. To create a shift register, one could either reverse the order of the statements, or use the shift-into operator, `->`.

```
reg int v1, v2, v3;

edge void Shift(int input)
{
    input -> v1 -> v2 -> v3;
}
```

With this operator, the left hand expression is assigned into the right hand variable, and returns the old value of the right hand variable.

We can also define ring-buffers and swap statements. Consider the following:

```
reg int a, b, c, d, e;

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
    public bit reset;
    reg int count;
    
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
    public bit clk;
    public bit reset;
    reg int count;
    
    public out Count => count;
    event Increment on(clk -> 1) { count = reset ? 0 : count + 1; }
}

module Counter3
{
    public bit clk, reset;
    reg int count;
    
    public out Count => count;
    on(clk -> 1) { count = reset ? 0 : count + 1; }
}
```

All three perform essentially the same function, but in different styles, becoming progressively more terse. Notice that if a member does not need to be accessed by name, it does not require a name in code. This includes module instances, events, and certain methods (when assigned to events).

## Function pointers
*TODO*

# Interfaces
An interface is an abstract group of methods, properties and indexers. In principle, an interface can be defined as a normal abstract module; it’s just a different syntax. All interface members must be public and abstract, being declared as such by default, and they cannot be registers.

```
interface ICollection<T>
{
    int Count { get; }
    T this[int i] { get; set; }
}
```

# Events
An event defines the entry points to the code - the moment when your actions are executed. A single entry point can be defined with the on keyword. What follows is a combination of applicable states defining the clock that causes the action to take place. OHDL uses `->` in this sense to mean ‘becomes’, and allows logical operators to combine multiple expressions (or, in fact, any combinatorial expression).

```
public &clk;
int a;

on (clk -> 1) => a++;
```

We can also define named events which can have methods added to them at other points in the code.

```
public bit clk;
reg int a;

event Increment on (clk -> 1);
Increment => { a++ };
```

The transition state can be any value appropriate for the variable type. The operator can also be omitted completely omitted, which will synthesise differently depending on whether the ddr keyword is present, as shown below.

```
bit[8] clk;

event NormalEvent on (clk);   // Equivalent to (clk -> 1)
ddr event DDREvent on (clk);   // Equivalent to (clk -> 1 or clk -> 0)
```

## Multiple clocks
Clock events can be combined in a single statement using the `or` keyword. Consider the following two snippets:

```
public & clk1, clk2;
inline int a, b;

on (clk1 | clk2 -> 1) a++;
on (clk1 -> 1 or clk2 -> 1) b++;
```

*TODO: Signal diagram here. A goes from 0 to 1, followed by B*

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

# Operator overloading

# Iterators

# Templates

# External connections
The `extern` keyword is used to expose nets to the outside world; to the physical pins of the device or other modules. As with normal nets, the `in`, `out` and `unsafe` keywords can be applied to change the type. The usual pattern is to define a single static module with all the external nets as the ‘entry point’ for the overall program, but external nets can be added to module instances too, and in multiple locations.

External nets have some restrictions when compared with normal nets, in that all methods and registers for that reference are implicitly local (not inline) and singly-clocked (or DDR, if the member is defined as such)
