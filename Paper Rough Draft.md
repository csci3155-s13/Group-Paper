The syntax of this proposed change can be summarized by the following:  

~~~~~~~~~~~~~~  
spawning-expression:  
  cilk_spawn function-or-functor ( expression-list )  
sync-statement:  
  cilk_sync ;  
parallel-loop:  
  cilk_for ( init-clause ; condition-expr ; increment-expr ) statement  
~~~~~~~~~~~~~~  
In the above, expression-list is optional for cilk_spawn. Likewise, init-clause is optional for cilk_for.

Usage of cilk_spawn is relatively straightforward. The keyword cilk_spawn must be followed by a function, functor, or lambda expression. That function, functor, or lamda expression is referred to as the spawned function and the function that contains cilk_spawn is the spawning function. A full expression never contains more than one cilk_spawn.  
  
Assuming a given parallel program is deterministic, its parts are referred to in terms of its serial form, or serialization. To arrive at the serialization of a parallel program implemented with cilk_spawn, cilk_sync, and cilk_for, one can simply remove cilk_spawn and cilk_sync and change cilk_for to a garden-variety for loop. Strands in the program are referred to with respect to their serial order, using the terms earlier, later, earliest, and latest. This concept is called the serialization rule.  
  
In order to define the behavior of the three cilk functions, the concept of a task block must be defined. A task block is the body of a function, the body of a cilk_for loop, or a try block. A task block is considered complete when all of its children have finished executing and have synced using cilk_sync. One should not enter or exit a task block using goto, break, or return.  
  
Cilk_spawn gives permission for execution to proceed asynchronously. Control flow forks at the spawn point, which is the name given to the place where cilk_spawn is called. The spawned function’s arguments are evaluated before the spawn point. It is possible that undefined behavior may arise due to data races as a result of this parallelism.  
  
Cilk_sync demands that all children of the current task block complete their own execution before execution of the task block itself may resume.  
  
