# Things to find out

* Can task wire variables be assigned after declaration?
	* Can't have wire variables
* Can local variables be initialized in-line?
	* Only when inside the begin/end block
* What happens to the state of a task when called twice?
	* Tasks don't have states.
* When do task states synthesise multiple times?
	* Tasks don't have states.
* Can case statements have multiple conditions?
	* Yep - use comma to separate
* Multiple modules in a file? Including files?
	* Seems to work
* Assignment of wire from register - including register writes
	* Works as expected

# Things I found out

* Tasks cannot have wire local variables
* All timing control statements within blocks are ignored.
* Always blocks can't have local variables at all

