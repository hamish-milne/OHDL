The standard library is written in pure OHDL. The only 'native' types are `bit` and `tri`, whose operators will map directly to the native operations in the target language.

## Included types

* Arithmetic types - integer and floating point
* `Sys.IOPort` for safe bidirectional communication
* `Sys.ShiftReg` - a shift register
* `Sys.RingBuf` - a ring buffer
* `Sys.Queue` - a first-in, first-out buffer
* `Sys.Stack` - a last-in, first-out buffer
* `Sys.ManualTrigger` - allows manual/custom triggering of events
* 
