# Finite State Machine Language (FSML) compiler

## 1. Introduction
FSML is a **formal language** to describe Finite State Machines (FSMs) in a simple and effective manner.

FSML has been thought from programmers **for programmers**.
 
It was born from the need to have a simple but effective way, not using graphical tools, to write an FSM and to have it **translated to C language** (*and maybe others in the future*).

Whit FSML is possible to describe **states and transitions**, as well as arbitrary code that shall be executed within them.
There are also some constructs that allow to describe common features like **retries** and **timeouts**, in a very simple way.

FSML is **portable**, since it is compiled to C and is completely **standalone**, not relying on any other third-party library.
  

## 2. Installation
TODO


## 3. The language
### FSM structure
The first thing to is to define the FSM itself:
```
fsm myFirstFSM {
  // ... and this is a comment
}
```
You jus need to use the keyword `fsm` followed by a *name* and the enclose the FSM code in a pare of `{}`.

### States
Then you have to define states within the FSM:
```
state [start] firstState
on ( /* here it goes a condition */ ) go middleState;

state middleState
on ( /* ...condition... */ ) go endState;

state [end] endState;
```
So, use keyword `state` to define a new state, followed by an optional *type* that can be either `start`, `end` or `err`, followed by the *name* of the state.

Every transition that brings to another state is described by the construct `on-go`. The keyword `on` is followed by a condition between `()`, then `go` and the name of the end state. Before digging into conditions, let's make a digression about variables.

### Input and Variables
FSML has three families of variables:
- internal variables, visible only inside the FSM (defined by the keyword `var`)
- input variables, that the code that uses the FSM can set from the extern (`input`)
- output variables (`output`), more on these later

Except for the variable family, the definition is exactly the same as in C language, with a type, the name and an initializer (*mandatory*!)
```
var int myVar = 0;
input short condition = 0;
```

### Transitions 
Let's go back to trantions and have a deeper look into this construct. We've previously mentioned that a transition is specified by the keyweord `on`, followed by a condition and the *actuator* `go`, that specifies the next state.
In order to be activated, a transition need its condition to be evaluated to true. This condition is expressed in a pair of brackets `()`, that contain a boolean expression using variables and inputs.
```
state [start] firstState
on (myVar > 0) go secondState
on (myVar == 0) go thirdState;
```
In the previous snippet, if the internal variable `myVar` is greater than 0, this will activate a transition toward `secondState`. If the condition `myVar == 0` evaluates to true, instead, the transition will bring the FSM to the state `thirdState`. In case none of the conditions are true, the FSM will stay in the same state, i.e. `firstState`.

### Errors
If an error state exists (i.e. a state with the `err` type), the transition actuator can be `err` instead of `go`, followed by a label that identifies the type of error, e.g.:
```
state [start] firstState
on (myVar > 0) go secondState
on (myVar == 0) go thirdState
on (myVar < 0) err negativeValue;

state [end, err] lastState;
```

### Retries
On thing that is usefull expecially in embedded systems and critical applications, is to retry a certain operation that has failed.
A special construct can be used for this, the `until-retry`.
```
until (MAX_RETRIES) {
   state waitStep1
   on (input1 == OK) go waitStep2;
   
   state waitStep2
   on ((input1 == OK) && (input2 == OK)) go endState
   on (input1 == FAIL) retry;
} err tooManyRetries;

state [end, err] endState;
```
In the code above, there are two states surrounded by an `until` block. This block is composed by the keyword `until`, followed by a pair of brackets `()` that contains a *numeric constant* value, that specifies the maximum number of retries. The `until` block ends with a transition actuator (in this case an `err` actuator), that specifies what the FSM shall do next, if the maximum number of retries have been reached.

The `until` block can contain any number of states, that can use the `retry` transition actuator to go to the first state of the block.

### Declarations
In the previous sections the condition for the `until` block is expressed by the constant `MAX_RETRIES`, whose numeric value shall be defined.
For this reason, the FSML has a special block `decl` that can contain specific declarations. The `decl` block contains user-defined C code that is then included on top of the generated code of the state machine.
```
decl {
#define MAX_RETRIES  3
#define OK 1
#define FAIL 0
}

fsm myFsm {
   until (MAX_RETRIES) {
      // the states go here ...
   }
}
```

### Timeout
Another useful construct in case of FSMs that are implemented in real-time systems are timeouts. The FSML has a special notation for using timeouts in an easy fashion.
There is a 4th family of variables: timers. The `timer` family specifies a timer variable as in the following examples:
```
timer firstTimer(1000);
timer secondTimer(MAX_TIMEOUT);
```
The `timer` keyword is followed by a *name* and a pair of brackets `()` that contains a *numeric constant* that specifies the initial value of the timer (in milliseconds).

The timer is then used through two statements: `start` and `timeout`.
The `start` keyword can be put at the end of a transition, after the actuator, followed by the name of the timer that shall be started.
The `timeout` keyword can be used as a condition for a transition, followed by the name of the timer between brackets.
```
timer t(100);   /* timer of 100 ms */

state [start] firstState
on (input1 == OK) go waitState start(t);

state waitState
on (input2 == OK) go endState
on timeout(t) err inputIsLate;
```

### Timer implementation
In order to make the implementation of the FSM portable on any system, the user shall specify a way to count time on its specific system.
This can be done by means of two possible sections: `time` or `period`.
Both constructs shall specify the body of a C function that return a POSIX-compliant `struct timespec`.
In case of `time` the structure shall contain the current time, e.g.:
```
time {
   struct timespec t;
   os_get_sys_time(&t);
   return t;
}
```
In case of `period`, the structure shall contain a fixed period of time expressed in milliseconds, e.g.:
```
period {
   struct timespec t;
   t->tv_sec = 0;
   t->tv_nsec = 1000000;		// 1 ms period
   return t;
}
```
Either one construct or the other can be defined, and they shall be put outside of the `fsm` section, like the `decl` section.

### Output
The third family of variables is `output`. This kind of variables can be used to pass values to the outside code, that uses the FSM.
Output variables can only be assigned, but never used in transition conditions.
Output variables can be set using the `out` keyword, followed by the name of the variable and a statement containing the code of a C function that returns the value to be assinged to the variable, as in the following example:
```
output int fsmState = 0;

state [start] s1
on (myVar > 0) go s2
out fsmState { return 0; };

state s2
on (myVar < 0) go s3;

state [end] s3
out fsmState { return 1; };
```
In the example, states `s1` and `s3` set the value of the output variable `fsmState` while state `s2` does not. This means that when going to state `s2`, the value of the output keeps its last value unchanged.

### User-specific code
Both states and transitions can have a specific block of code that is executed as it is.
The code associated to a state is executed every time the FSM runs through that state, while the code associated to a specific transition is executed once the FSM traverses that transition. Let's make an example of a state that resets all internal variables and when receives a new input it increments the corresponding variable:
```
var int myArray[ARRAY_SIZE] = {0};

state resetState {
   /* State user-specific code */
   unsigned int i = 0;
   for (i = 0; i<ARRAY_SIZE; ++i) {
      myArray[i] = 0;
   }
}
on ( (inp >= 0) && (inp < ARRAY_SIZE)) {
   myArray[inp]++;
} go resetState /* this is a self-transitions */
on ( (inp < 0) || (inp >= ARRAY_SIZE)) err INPUT_OUT_OF_RANGE;
```

### Built-in variables
The FSML has a built-in variable `this` that can be used from within the user-specific code, in order to access the FSM interfaces, explained in detail in the following section.
```
state [end, err] lastState
out cur_state { this.err() ? state_ERROR : state_COMPLETE; };
```


### 4. The FSML interface (how to use the FSM in your own code)
TODO
-->
In particular, through the `this` keyword, it is possible to know the current state of the FSM and if there is an error.
`this.state()` is a built-in function that returns an enumerator defining the current state, and `this.err()` returns an enumerator with the current error, if any.
The following example, that refers to the one in the previous section, should clarify a bit how to use them:
```
fsm inputCounter {
   input int inp = 0;
   output inputCounter__State curState = 
state [end, err]
out 
```
