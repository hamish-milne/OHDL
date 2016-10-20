
## Hello world
Let's write a program that will output the ASCII string "Hello world!" in binary.

```
/* A module is a collection of data and behaviour that can be used together as a unit.
 * Here we only need one module because the program is pretty simple. Calling it 'Main'
 * tells the compiler to use it as the top-level entity; the place we connect external
 * inputs and outputs to.
 * All the public members of a module will appear as wires in the design software.
 */
module Main
{
    /* Here we define a net, a wire that connects somewhere else in the design.
     * In this case it's `public`, meaning it connects outside the module.
     * The `bit` type is a single wire.
     */
    public bit Clk;
    public bit Start;
    
    /* This is our output, declared as an auto-property.
     * An auto-property is a register for which you can specify where each operation is allowed.
     * In this case, we're allowing the value to be read anywhere, but writing is only allowed
     * within this module.
     */
    public bit Output { get; private set; }
    
    /* Here we define a method for serialising a character string into bits and writing out each bit.
     * `task` means that we can perform operations over several clock cycles.
     * The argument type, `STRING`, is a 'meta-type' that will instruct the compiler to produce different logic.
     */
    task void WriteString(STRING str) {
        /* We want to access the individual bits of the string. The `raw` keyword lets us do that */
        bit[] data = (raw bit[])str;
        /* This is a standard `for` loop, in which we iterate through the bits */
        for(int i = 0; i < sizeof(data); i++) {
            /* Set the output register to the current bit */
            output = data[i];
            /* Wait for a clock cycle */
            on(Clk);
        }
    }
    
    /* An `event` is a module member that defines a response to a change in state
     * They can be named which allows additional actions to be added to them in a constructor,
     * but it's not required. The `on` statement tells us the state change that triggers the action.
     * In this case, when Start -> 1, the WriteString task is started with the given parameters.
     */
    event on(Start) WriteString("Hello world!");
}
```

