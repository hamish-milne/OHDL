
## Names

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
