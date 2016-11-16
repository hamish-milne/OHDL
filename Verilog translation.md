
## Names

OHDL names are always valid verilog names. Verilog also allows the dollar sign `$` so we can use this as a separator character and for internal/compiler generated members.

We can use `_$` for internal names (which aren't OHDL keywords already) and identifiers which are Verilog keywords, which includes `input`, `output`, `inout`, `parameter`, `wire`. 

## Class

```
// Input
module Foo {
  public int bar;
  reg int baz;
  
  public float Bat => bar + baz;
}
using Main = Foo;

// Output
module Foo ( // Foo
  input [31:0] bar,
  output [31:0] Bat
);

  reg [31:0] baz;

  assign Bat = bar + baz;

endmodule
```
  
## Inheritance

```
<Base class module> base #(<parameters>) (<arguments>);
```

Test fragment:

```
// Input
module Foo { }
module Bar : Foo { }

// Output
module Foo (
);
endmodule

module Bar (
);
  Foo base #() ();
endmodule
```

## Fields

```
{reg|wire} [<field size>:0] <field name>;

// Input
module Foo {
	public int Bar;
	private reg int Baz;
	public int Bat {get; private set;}
}

// Output
module Foo (
	input [31:0] Bar,
	output reg [31:0] Bat
);

reg [31:0] Baz;

endmodule
```

## Public register

```
// Input
sysclk module Foo {
	public reg int Bar;
}

// Output
module Foo (
	input sysclk,
	input [31:0] Bar$input,
	output reg [31:0] Bar,
	input Bar$clk
);

always @(posedge sysclk) begin
	if(Bar$clk) begin
		Bar = Bar$input;
	end
end

endmodule
```

## N-edge register

```
reg <size> ntrig$<name> [N-1:0];
wire <name>;
assign <name> = ntrig$<name>[0] ^ ... ^ ntrig$<name>[N-1];

<-- For each signal i
always @(posedge <signal i>) begin ntrig$<name>[i] = ntrig$<name>[0] ^ ... <input i> ^ ... ntrig$<name>[N-1];
-->


// Input
module Foo {
  public bit clkA, clkB, clkC;
  public int count {get; private set;}
  
  event (clkA -> 1 or clkB -> 1 or clkC -> 1) => count++;
}

// Output
module Foo (
  input clkA,
  input clkB,
  input clkC,
  output [31:0] count
);

  reg [31:0] ntrig$count [2:0];
  assign count = ntrig$count[0] ^ ntrig$count[1] ^ ntrig$count[2];

  wire [31:0] _$event0$count0;
  assign _$event0$count0 = count + 1;

  always @(posedge clkA) begin ntrig$count[0] = _$event0$count0 ^ ntrig$count[1] ^ ntrig$count[2]; end

  always @(posedge clkB) begin ntrig$count[1] = ntrig$count[0] ^ _$event0$count0 ^ ntrig$count[2]; end

  always @(posedge clkC) begin ntrig$count[2] = ntrig$count[0] ^ ntrig$count[1] ^ _$event0$count0; end

endmodule
```

## Events

### Async

```
always @(posedge (<expression> == <value>))
begin
  <event body>
end
```

Test fragment:

```
// Input
module Foo {
  public bit clk;
  reg int count1, count2;

  event Bar(clk -> 1) => count1++;
  event Baz(clk -> 0) => count2++;
}

// Output
module Foo (
  input clk
);

  always @(posedge clk) begin
    count1++;
  end

  always @(negedge clk) begin
    count2++;
  end

endmodule
```

### System clock

```
always @(posedge sysclk) begin
  if(<condition>) begin
    <event body>
  end
end
```

## Combinatorial methods

```
function <method name>;
input <arguments>;
begin
  <method body>
end
```

Test fragment:

```
// Input
module Foo {
  public int Bar(int a, int b, int c) {
    return (a*b) + c;
  }
}

// Output
module Foo (
);

  function Bar;
  input a, b, c;
  begin
    Bar = (a*b) + c;
  end

endmodule
```

## Edge-triggered methods

Edge-triggered method bodies are inlined into their respective calling point blocks.


Test fragment:

```
// Input
sysclk module Foo
{
  reg int a;
  
  public edge void Bar() {
    a *= 3;
  }
}

// Output
module Foo (
  input sysclk,
  input Bar$trigger
);

  reg [31:0] a;

  task Bar;
  begin
    a = a * 3;
  end
  endtask

  always @(posedge sysclk) begin
    if(Bar$trigger) begin
      Bar()
    end
  end

endmodule
```

### Sync/Async task calls


Test fragment:

```
// Input
sysclk module Foo {
  
  task int Bar(reg int b) {
    reg int a = 0;
    a += b;
    await sysclk;
    a *= 3;
    await sysclk;
    return a + 1;
  }
  
  task int Baz() {
    var t = async Bar(3);
    reg int a = Bar(4);
    reg int b = t.Await();
    return a + b;
  }
}

// Output

module task$Foo$Bar (
  input [31:0] b,
  output [31:0] $return,
  input $trigger,
  output $done
);

  reg $done;

  task Bar;
  reg [31:0] a;
  begin // Can be separated into a header file
    a = 0;
    a = a + b;
    @(posedge sysclk);
    a = a * 3;
    @(posedge sysclk);
    $return = a + b;
  end
  endtask

  always @(posedge $trigger) begin
    $done = 0;
    Bar()
    $done = 1;
  end

endmodule

module Foo (
);

  wire [31:0] state$0$b;
  wire [31:0] state$0$$return;
  reg state$0$$trigger;
  wire state$0$$done;
  task$Foo$Bar state$0(state$0$b, state$0$$return, state$0$$trigger, state$0$$done);

  task state$0$Await;
  output [31:0] $return;
  begin
    state$0$$trigger = 0;
    wait (state$0$$done);
    $return = state$0$$return;
  end
  endtask

  task Bar;
  input reg [31:0] b;
  output [31:0] $return;
  reg [31:0] a;
  begin
    a = 0;
    a = a + b;
    @(posedge sysclk);
    a = a * 3;
    @(posedge sysclk);
    $return = a + b;
  end
  endtask

  task Baz;
  output [31:0] $return;
  reg [31:0] a;
  reg [31:0] b;
  begin
    state$0$b = 3;
    state$0$$trigger = 1;
    Bar(4, a);
    state$0$Await(b);
    $return = a + b;
  end
  endtask

endmodule
```

## For loop

Test fragment:

```
// Input
module Foo {
  public bit start;
  reg int a;
  
  event (start -> 1) {
    for(int i = 0; i < 5; i++) {
      a++
    }
  }
}

// Output
module Foo (
  input start
);

  always @(posedge start) begin
    reg [31:0] i = 0;
    while(i < 5) begin
      a = a + 1;
      i = i + 1;
    end
  end

endmodule
```

## Switch statement

Test fragment:

```
// Input
module Foo {

	public int a;
	public int b => Bar(a);

	int Bar(int a) {
		switch(a) {
			case 1:
				return 2;
			case 2:
				return 3;
			case 3:
				return 5;
			default:
				return 0;
		}
	}
}

// Output
module Foo (
	input [31:0] a;
	output [31:0] b;
);

function Bar(
	a
) begin
	case(a)
		1:
			Bar = 2;
		2:
			Bar = 3;
		3:
			Bar = 5;
		default:
			Bar = 0;
	endcase
end

```

## RAM

```
// Input
sysclk module Memory<T, N>
{
	const INT A = ceilog2(N);
	private reg T[N] mem;
	
	public T Read(bit[A] addr)
	{
		return mem[addr];
	}
	
	public edge void Write(bit[A] addr, T value)
	{
		mem[addr] = value;
	}
}

using Main = Memory<int, 10>;

// Output
module Main (
	input [3:0] Read$addr,
	output [31:0] Read$$return,
	input [3:0] Write$addr,
	input [31:0] Write$value,
	input Write$$trigger
);

reg [31:0] mem [9:0];

function Read (
	addr
);
begin
	Read = mem[addr];
end

task Write;
input [3:0] addr;
input [31:0] value;
begin
	mem[addr] = value;
end
endtask

assign Read$$return = Read(Read$addr);

always @(posedge Write$$trigger) begin
	Write(Write$addr, Write$value);
end
	
endmodule
```

## Pipelines

```
// Input
module Foo
{
  static edge T Break<T>(this T input)
  {
    reg T a;
	T b = a;
	a = input;
	return b;
  }

  static int Add(this int input)
  {
    return input + 3;
  }
  
  static int Mul(this int input)
  {
    return input * 5;
  }
  
  pipeline Pipe => input.Add().Break().Mul();
}

// Output

module Foo$Pipe (
	input [31:0] $input,
	output [31:0] $output,
	input clk
);

function Add;
input [31:0] $input;
begin
	Add = $input + 3;
end

function Mul;
input [31:0] $input;
begin
	Mul = $input * 5;
end

wire [31:0] Break$int$input;
reg [31:0] Break$int$a;
wire [31:0] Break$int$return;

task Break$int;
wire [31:0] b;
begin
	b = Break$int$a;
	Break$int$a = $input;
end
endtask

always @(posedge clk) begin
	wire [31:0] Break$input = Add($input);
	wire 
end

endmodule

module Foo (
  input [31:0] Pipe$input,
  output [31:0] Pipe$output,
  input Pipe$clk
);

function Add;
input [31:0] $input;
begin
	Add = $input + 3;
end

function Mul;
input [31:0] $input;
begin
	Mul = $input * 5;
end

task Break$int;
input [31:0] $input;
output [31:0] $return;
reg [31:0] a;
wire [31:0] b;
begin
	b = a;
	a = input;
	$return = b;
end
endtask

task Pipe$pipeline;
begin
	wire [31:0] $input$Break$int$0 = Add(Pipe$input);
	wire [31:0] $return$Break$int$0;
	Break$int($input$Break$int$0, $return$Break$int$0);

always @(posedge Pipe$clk) begin
	
	
endmodule
```
