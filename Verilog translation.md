
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

## Inheritance

```
<Base class module> base #(<parameters> (<arguments>);
```

## Fields

```
{reg|wire} [<field size>:0] <field name>;
```

## N-edge register

```
reg [N:0] <name>;
assign out = ^<name>;

<-- For each signal i
wire [N:0] wire_i;
assign wire_i = { <name>[0], ... in, ... <name>[N-1] };
always @(posedge <signal i>) begin <name>[i] = ^wire_i;
-->
```

## Events

### Async

```
always @(posedge (<expression> == <value>))
begin
  <event body>
end
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

## Edge-triggered methods

Edge-triggered method bodies are inlined into their respective calling point blocks.

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

### Async task calls
