
## Hello world
Let's write a program that will output the ASCII string "Hello world!" in binary.

```
/* A module is a collection of data and behaviour that can be used together as a unit.
 * Here we only need one module because the program is pretty simple. Calling it 'Main'
 * tells the compiler to use it as the top-level entity; the place we connect external
 * inputs and outputs to.
 * All the public members of a module will appear as wires in the design software.
 * The `sysclk` keyword tells us that this module has a single 'system clock' which
 * triggers all events and tasks in the module.
 */
module sysclk Main
{
    /* Here we define a net, a wire that connects somewhere else in the design.
     * In this case it's `public`, meaning it connects outside the module.
     * The `bit` type is a single wire.
     */
    public bit Start;
    
    /* This is our output, declared as an auto-property.
     * An auto-property is a register for which you can specify where each operation is allowed.
     * In this case, we're allowing the value to be read anywhere, but writing is only allowed
     * within this module.
     */
    public bit Output { get; private set; }
    
    /* Here we define a method for serialising a character string into bits and writing out each bit.
     * `task` means that we can perform operations over several clock cycles.
     * The argument type, `STRING`, is a 'meta-type' that will instruct the compiler to produce different logic
     * depending on the value passed in.
     */
    task void WriteString(STRING str) {
        /* We want to access the individual bits of the string. The `raw` keyword lets us do that */
        bit[] data = (raw bit[])str;
        /* This is a standard `for` loop, in which we iterate through the bits */
        for(int i = 0; i < sizeof(data); i++) {
            /* Set the output register to the current bit */
            output = data[i];
            /* Wait for a clock cycle */
            await sysclk;
        }
    }
    
    /* An `event` is a module member that defines a response to a change in state
     * They can be named which allows additional actions to be added to them in a constructor,
     * but it's not required.
     * In this case, when Start == 1, the WriteString task is started with the given parameters.
     */
    event (Start) WriteString("Hello world!");
}
```

...


## Tasks, await and async

A task method can `await` in its method body for an edge, and call other tasks both synchronously and asynchronously.

```
task int Fibbonaci(reg int n) // The 'reg' keyword will latch in the argument
{
    reg int a = 1, b = 1;
    while(n > 0) {
        int c => a + b;
        a = b;
        b = c;
        await sysclk;
    }
}

task int MyTask(int addr)
{
    reg int word = ReadFromDram(addr); // This is a synchronous task call
    var t = async ReadFromDram(addr + 1); // This is an asynchronous task call that returns a Task<int> structure
    // Do some other operations
    reg int word1 = t.Await(); // await t; int word1 => t.Result; would also work here
    return word + word1;
}
```

Synchronous calls are effectively inlined into the task, which is to say their states become part of a single overall state.

Asynchronous calls must however create their own state. By default, each call will create a separate state object, however for efficiency a state name can be specified.

```
module MyModule
{
    reg stateof(MyTask) state1; // This is a module-local state
    
    static reg stateof(MyTask) state2; // Static states are 'global'
    
    // Since states are just registers, they can be re-used for different tasks, provided they're large enough
    
    task int MyTask(int addr)
    {
        reg int word = ReadFromDram(addr) state1; // This is a synchronous task call
        var t = async ReadFromDram(addr + 1) state2; // This is an asynchronous task call that returns a Task<int> structure
        // Do some other operations
        reg int word1 = t.Await(); // await t; int word1 => t.Result; would also work here
        return word + word1;
    }
}
```
