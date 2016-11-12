
## Names

OHDL names are always valid verilog names. Verilog also allows the dollar sign `$` so we can use this as a separator character and for internal/compiler generated members.

## Class

```
module <class name> ( // <original class name>
<-- Wire IO
  <field name>, // <original field name>
-->
);

<-- Wire IO
{input|output|inout} {reg|wire} [<field size>:0] <field name>;
-->

<-- Meta type fields
parameter <Field name> = <Value> ;
-->

endmodule
```

Test fragment:

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
  bar,
  Bat
);
input [31:0] bar;
output [31:0] Bat;

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
```

## N-edge register

```
reg <size> ntrig$<name> [N-1:0];
wire <name>;
assign <name> = ^ntrig$<name>;

<-- For each signal i
wire <size> wire_i [N-1:0];
assign wire_i = { ntrig$<name>[0], ... <input i>, ... ntrig$<name>[N-1] };
always @(posedge <signal i>) begin ntrig$<name>[i] = ^wire_i;
-->
```

Test fragment:

```
// Input
module Foo {
  public bit clkA, clkB, clkC;
  reg int count;
  
  event (clkA -> 1 or clkB -> 1 or clkC -> 1) => count++;
}

// Output
module Foo (
  clkA,
  clkB,
  clkC
);
input clkA;
input clkB;
input clkC;

reg [31:0] ntrig$count [2:0];
wire [31:0] count;
assign count = ^ntrig$count;

wire [31:0] $event0$count0;
assign $event0$count0 = count + 1;

wire [31:0] nwire_0$count [2:0];
assign nwire_0$count = { $event0$count0, ntrig$count[1], ntrig$count[2] };
always @(posedge clkA) begin ntrig$count[0] = ^nwire_0$count;

wire [31:0] nwire_1$count [2:0];
assign nwire_1$count = { ntrig$count[0], $event0$count0, ntrig$count[2] };
always @(posedge clkB) begin ntrig$count[1] = ^nwire_1$count;

wire [31:0] nwire_2$count [2:0];
assign nwire_2$count = { ntrig$count[0], ntrig$count[1], $event0$count0 };
always @(posedge clkC) begin ntrig$count[2] = ^nwire_2$count;

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
  clk
);
input clk;

always @(posedge clk)
begin
  count1++;
end

always @(negedge clk)
begin
  count2++;
end

endmodule
```

### System clock

```
always @(posedge sysclk)
begin
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
  sysclk,
  Bar$trigger
);
input sysclk;
input Bar$trigger;

reg [31:0] a;

task Bar;
begin
  a *= 3;
end

always @(posedge sysclk)
begin
  if(Bar$trigger)
  begin
    Bar()
  end
end

endmodule
```

## Task methods

```
task <method name>;
<-- Wire IO
{input|output|inout} {reg|wire} [<argument size>:0] <argument name>;
-->
begin
  <method body>
end
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
  b,
  $return,
  $trigger,
  $done
);
input [31:0] b;
output [31:0] $return;
output $done

reg $done;

task Bar;
begin
  reg [31:0] a = 0;
  a += b;
  @(posedge sysclk);
  a *= 3;
  @(posedge sysclk);
  $return = a + b;
end

always @(posedge $trigger)
begin
  $done = 0;
  Bar()
  $done = 1;
end

endmodule

module Foo (
);

wire [31:0] state$0$b;
wire [31:0] state$0$$return;
wire state$0$$trigger;
wire state$0$$done;
task$Foo$Bar state$0(state$0$b, state$0$$return, state$0$$trigger, state$0$$done);



task Baz

endmodule
```
